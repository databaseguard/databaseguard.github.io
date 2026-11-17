==========
Policy DSL
==========

Policies are YAML documents under a directory in ``policies.dirs``. Each document declares
a set of **rules**; every query is matched against all enabled rules and the engine
returns a verdict. This page is the practical guide; :doc:`/reference/policy-schema` is the
exhaustive field reference.

Validate a bundle at any time:

.. code-block:: bash

   dbguard-cli validate-policy configs/policies

Document skeleton
=================

.. code-block:: yaml

   api_version: dbguard.io/v1          # required, literal
   kind: DbGuardPolicy                 # required, literal
   metadata:
     name: my-policy                   # required
     description: ...                  # optional
     labels: { tier: firewall }        # optional, free-form
   spec:
     default_action: allow             # verdict when no rule matches: allow | deny
     rules:
       - id: my.first_rule             # unique across all documents
         ...

Rule fields
===========

.. list-table::
   :header-rows: 1
   :widths: 20 20 60

   * - Field
     - Type
     - Meaning
   * - ``id``
     - string
     - Unique rule id (appears in events, metrics, dedupe).
   * - ``severity``
     - enum
     - ``info`` | ``low`` | ``medium`` | ``high`` | ``critical``.
   * - ``enabled``
     - bool
     - Optional; default ``true``. ``false`` silences the rule.
   * - ``engines``
     - list
     - Subject filter: ``postgres`` | ``mysql`` | ``sqlite`` (empty = all).
   * - ``databases`` / ``users``
     - list
     - Subject filters on database name / DB user (empty = all).
   * - ``clients``
     - list
     - Subject filter on client IP / CIDR, e.g. ``[10.0.0.0/8]``.
   * - ``identities``
     - list
     - Subject filter on **Cilium security identity** - id or label/substring
       (requires ``identity.cilium_enabled``).
   * - ``tables`` / ``columns``
     - list
     - Object filters parsed from the SQL.
   * - ``query``
     - object
     - Content matcher on the SQL text (see below).
   * - ``rate``
     - object
     - Rate limiting (see below).
   * - ``actions``
     - list
     - Decisions on match: ``allow`` ``alert`` ``deny`` ``drop`` ``reset`` ``rate_limit``
       ``mask`` ``kill`` ``freeze`` ``quarantine``.

All filters are **AND**-ed: every specified filter must pass for the rule to match. An
empty filter matches everything.

The ``query`` matcher
=====================

.. list-table::
   :header-rows: 1
   :widths: 24 12 64

   * - Field
     - Type
     - Meaning
   * - ``pattern``
     - string
     - Go (RE2) regular expression matched against the SQL text.
   * - ``keywords``
     - list
     - ANY-of substring alternative to ``pattern``.
   * - ``case_insensitive``
     - bool
     - Fold case for ``keywords`` (for ``pattern`` use ``(?i)``).
   * - ``min_length``
     - int
     - Skip SQL shorter than this (noise reduction).

.. tip::

   A column-list filter (``columns: [salary]``) matches **bare** column references. To
   catch a column wrapped in a function (``sum(salary)``) or anywhere in the statement,
   use a ``query.pattern`` such as ``'(?i)\bsalary\b'``. A text pattern is the stricter way
   to protect a sensitive field.

The ``rate`` matcher
====================

.. code-block:: yaml

   rate:
     events: 3        # at most N matches...
     window: 10s      # ...per window, per subject key (user + client + rule)

Windows accept ``s`` / ``m`` / ``h``. When the limiter trips, the rule's ``actions`` apply
(e.g. ``deny``); further matches inside the same window are suppressed to one alert.

Matching semantics
==================

1. Documents load in **filename order** (``00-`` before ``50-``); within a document, rules
   evaluate in list order. Sub-directories are ignored.
2. All filters on a rule must pass for it to match.
3. **First-deny-wins**: evaluation stops at the first rule whose action is denying
   (``deny`` / ``drop`` / ``reset`` / ``kill`` / ``freeze`` / ``quarantine``). Alert- and
   mask-only rules annotate and let evaluation continue.
4. If nothing denies, ``spec.default_action`` applies.
5. A match is always recorded in metrics, even for alert-only rules.

.. note::

   Because of first-deny-wins **and** filename ordering, a broad allow/rate rule placed in
   a late-numbered file will not mask a specific deny in an early file. Name files to
   encode the order you intend: e.g. identity allow-lists in ``05-``, the SQLi firewall in
   ``20-``, rate limits in ``90-``.

Hot reload
==========

With ``policies.watch: true``, the engine watches the policy directories and **rebuilds
the entire rule set from scratch** on any add, change, or delete - so a removed file's
rules are truly unloaded, and ordering is always recomputed. No restart, no signal.

.. code-block:: bash

   # add a rule at runtime (it applies within ~2s)
   cp 70-extra.yaml /etc/dbguard/policies/
   # remove it just as easily
   rm /etc/dbguard/policies/70-extra.yaml

Four worked examples
====================

**1. Block UNION extraction everywhere**

.. code-block:: yaml

   - id: sqli.union
     severity: critical
     query: { pattern: '(?i)union\s+(all\s+)?select' }
     actions: [deny, alert]

**2. Mask PII columns but never block**

.. code-block:: yaml

   - id: pii.cc.tables
     severity: high
     tables: [customers, employees]
     columns: [credit_card, ssn, salary]
     actions: [alert, mask]

**3. Rate-limit a bulk-export pattern**

.. code-block:: yaml

   - id: exfil.bulk
     severity: medium
     query: { pattern: '(?i)select\s+\*.*from' }
     rate: { events: 3, window: 10s }
     actions: [alert, rate_limit]

**4. Allow a table only for one workload identity**

.. code-block:: yaml

   - id: db.payments.only-billing
     severity: critical
     tables: [payments]
     identities: [k8s:app=billing]   # NOT billing → denied by a following default-deny
     actions: [allow]
