=============
Policy Schema
=============

The exhaustive reference for the policy YAML. For the practical guide and examples see
:doc:`/features/policy-dsl`.

Document
========

.. code-block:: yaml

   api_version: dbguard.io/v1     # required, literal - the loader refuses unknown versions
   kind: DbGuardPolicy            # required, literal
   metadata:
     name: <string>              # required
     description: <string>       # optional
     labels: { <k>: <v> }        # optional, free-form
   spec:
     default_action: allow|deny  # verdict when no rule matches
     rules: [ <Rule>, ... ]

Rule
====

.. list-table::
   :header-rows: 1
   :widths: 18 14 68

   * - Field
     - Type
     - Meaning
   * - ``id``
     - string
     - Unique rule id across all documents (metrics/event label, dedupe key).
   * - ``name``
     - string
     - Human title (optional).
   * - ``description``
     - string
     - Free text (optional).
   * - ``enabled``
     - bool
     - Optional; default ``true``. ``false`` silences the rule.
   * - ``severity``
     - enum
     - ``info`` | ``low`` | ``medium`` | ``high`` | ``critical``.
   * - ``engines``
     - []string
     - Subject filter: ``postgres`` | ``mysql`` | ``sqlite`` (empty = all).
   * - ``databases``
     - []string
     - Subject filter on database name (empty = all).
   * - ``users``
     - []string
     - Subject filter on DB user (empty = all).
   * - ``clients``
     - []string
     - Subject filter on client IP / CIDR, e.g. ``[10.0.0.0/8]``.
   * - ``identities``
     - []string
     - Subject filter on Cilium security identity - id (``"44982"``), exact label
       (``"k8s:app=api-server"``), or label substring (``"api-server"``). Requires
       ``identity.cilium_enabled``. Fail-closed.
   * - ``tables``
     - []string
     - Object filter: table names parsed from the SQL (empty = all).
   * - ``columns``
     - []string
     - Object filter: referenced columns parsed from the SQL (empty = all). Matches **bare**
       references; use ``query.pattern`` to catch columns inside functions.
   * - ``query``
     - QueryMatcher
     - Content matcher (below).
   * - ``rate``
     - RateRule
     - Rate limiting (below).
   * - ``actions``
     - []string
     - ``allow`` ``alert`` ``deny`` ``drop`` ``reset`` ``rate_limit`` ``mask`` ``kill``
       ``freeze`` ``quarantine``.

QueryMatcher
============

.. list-table::
   :header-rows: 1
   :widths: 22 12 66

   * - Field
     - Type
     - Meaning
   * - ``pattern``
     - string
     - Go (RE2) regex matched against the SQL text. Use ``(?i)`` for case-insensitive.
   * - ``keywords``
     - []string
     - ANY-of substring alternative to ``pattern``.
   * - ``case_insensitive``
     - bool
     - Fold case for ``keywords``.
   * - ``min_length``
     - int
     - Skip SQL shorter than this.

An empty ``query`` matches every statement (subject/object filters still apply).

RateRule
========

.. list-table::
   :header-rows: 1
   :widths: 18 12 70

   * - Field
     - Type
     - Meaning
   * - ``events``
     - int
     - Max matches per window.
   * - ``window``
     - string
     - Window length: ``s`` / ``m`` / ``h`` (e.g. ``10s``, ``1m``). Subject key = user +
       client + rule.

Actions
=======

.. list-table::
   :header-rows: 1
   :widths: 16 44 40

   * - Action
     - Effect
     - Kernel / mechanism
   * - ``allow``
     - Explicit allow, stops deny evaluation
     - verdict only
   * - ``alert``
     - Annotate, route to sinks
     - user space
   * - ``deny``
     - Reject the statement/session
     - proxy ``ERROR`` (sync) / ``cgroup_skb`` DROP (needs ``drop_enabled``)
   * - ``drop``
     - Explicit packet drop
     - ``cgroup_skb`` DROP
   * - ``reset``
     - Terminate the client connection
     - TCP ``RST`` via quarantine map
   * - ``rate_limit``
     - Throttle matching subjects
     - per-subject window (user space)
   * - ``mask``
     - Redact detected PII in audit/alert
     - user space
   * - ``kill``
     - ``SIGKILL``/``SIGTERM`` the offending backend
     - reactive kill path (PostgreSQL only)
   * - ``freeze``
     - Freeze the offending cgroup
     - cgroup v2 freezer
   * - ``quarantine``
     - Put the source on the quarantine map
     - kernel map + expiry

Evaluation order
================

1. Documents load in **filename** order; rules within a document in **list** order.
2. All filters on a rule must pass for it to match.
3. **First-deny-wins**: the first denying action stops evaluation. Alert/mask rules
   continue.
4. If nothing denies, ``spec.default_action`` applies.
5. Every match increments ``rule_matches_total{rule}``.

Versioning
==========

Keep ``api_version: dbguard.io/v1``. Future schema changes bump the version and the loader
refuses unknown versions, so an incompatible policy fails fast rather than loading
partially.
