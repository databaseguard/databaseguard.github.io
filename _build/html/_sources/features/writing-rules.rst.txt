=============================
Writing Rules: a Cookbook
=============================

A task-oriented companion to :doc:`/features/policy-dsl`. Every recipe here is an ordinary
policy rule - drop it into a file under ``policies.dirs`` and it hot-reloads within ~2s.
Validate first with ``dbguard-cli validate-policy`` and confirm behaviour with
``dbguard-cli dry-run`` (shown under each recipe).

.. contents::
   :local:
   :depth: 1

How matching works (read this first)
====================================

Two facts decide whether a rule will do what you expect:

1. **Filters are AND-ed, and scope the rule.** ``tables``, ``users``, ``clients``,
   ``identities``, ``engines``, ``databases`` all narrow *who/what* the rule applies to.
   An empty filter matches everything.
2. **The** ``query`` **matcher reads the SQL text.** ``pattern`` is a Go (RE2) regex;
   ``keywords`` is an any-of substring list. dbguard sees the statement string the client
   sends - so it can match statement *shape* (``UPDATE``, ``DELETE``), table and column
   names, and **literal values that appear in the text**.

.. important::

   **Literal values only.** A value is matchable only when it is written into the SQL
   text. In a **parameterized / prepared** statement the value travels as a separate bind
   parameter, so the text the engine sees is ``salary = $1``, never ``salary = 9``. Rules
   that key on a *value* work for ad-hoc / string-built SQL but **cannot** see the value of
   a parameterized query. Rules that key on the *operation or column* are unaffected - that
   is usually what you want. Each recipe below says which kind it is.

.. _recipe-deny-update:

Deny ``UPDATE`` on a particular table
=====================================

*Operation-level - robust against parameterized queries.*

``tables`` scopes it to the table; the anchored ``pattern`` restricts it to ``UPDATE``.
Everything else on the table keeps flowing.

.. code-block:: yaml

   # configs/policies/30-no-update-employees.yaml
   api_version: dbguard.io/v1
   kind: DbGuardPolicy
   metadata: { name: no-update-employees }
   spec:
     default_action: allow
     rules:
       - id: emp.deny-update
         severity: high
         tables: [employees]
         query: { pattern: '(?i)^\s*update\b' }
         actions: [deny, alert]

- ``(?i)`` folds case; ``^\s*update\b`` anchors to the statement verb so a ``SELECT`` that
  merely contains the word "update" does **not** match.
- A simpler ``query: { keywords: ["update"], case_insensitive: true }`` also works, but
  ``keywords`` match anywhere in the text, so the anchored pattern is stricter.

.. code-block:: bash

   dbguard-cli dry-run --engine postgres --sql "UPDATE employees SET x = 1 WHERE id = 1"
   # -> denied, rule emp.deny-update
   dbguard-cli dry-run --engine postgres --sql "SELECT * FROM employees"
   # -> allowed

Protect one column from being written
=====================================

*Operation/column-level - robust against parameterized queries.*

"Nobody may change ``salary``" should match the **assignment**, not a specific value, so it
holds no matter how the value is passed (``salary = 9``, ``salary = $1``,
``salary = salary + 100``):

.. code-block:: yaml

   - id: emp.deny-salary-write
     severity: high
     tables: [employees]
     query: { pattern: '(?i)\bupdate\b[^;]*\bset\b[^;]*\bsalary\s*=' }
     actions: [deny, alert]

The ``[^;]*`` between tokens allows other assignments before ``salary`` (e.g.
``SET name = 'x', salary = ...``).

.. code-block:: bash

   dbguard-cli dry-run --engine postgres --sql "UPDATE employees SET salary = 9 WHERE id=1"
   # -> denied, rule emp.deny-salary-write
   dbguard-cli dry-run --engine postgres --sql "UPDATE employees SET salary = \$1 WHERE id=\$2"
   # -> denied (matches the assignment, not the value)

Deny a specific **value** being set
===================================

*Value-level - works for literal SQL only (see the caveat above).*

If you really want to block one literal value - say ``salary = 9`` - put the value in the
pattern:

.. code-block:: yaml

   - id: emp.deny-salary-9
     severity: high
     tables: [employees]
     query: { pattern: '(?i)\bset\b[^;]*\bsalary\s*=\s*9\b' }
     actions: [deny, alert]

- ``\bsalary\s*=\s*9\b`` - the trailing ``\b`` word boundary stops it also matching
  ``90`` or ``99``.

.. code-block:: bash

   dbguard-cli dry-run --engine postgres --sql "UPDATE employees SET salary = 9 WHERE id=1"
   # -> denied, rule emp.deny-salary-9
   dbguard-cli dry-run --engine postgres --sql "UPDATE employees SET salary = 10 WHERE id=1"
   # -> allowed

.. warning::

   ``UPDATE employees SET salary = $1`` (parameterized) will **not** be caught by a
   value rule - the literal ``9`` is not in the text. Use the column-write recipe above if
   you need the guarantee regardless of how the app sends the value.

Block ``DELETE`` without a ``WHERE`` clause
===========================================

*Operation-level.* A classic foot-gun - a full-table delete:

.. code-block:: yaml

   - id: safety.delete-no-where
     severity: critical
     query: { pattern: '(?i)^\s*delete\s+from\s+\w+\s*;?\s*$' }
     actions: [deny, alert]

Matches ``DELETE FROM t`` / ``DELETE FROM t;`` but not ``DELETE FROM t WHERE ...``. Pair it
with a ``tables`` filter to scope it to high-value tables, or leave it global.

Restrict writes to a table to one user or workload
==================================================

*Subject + operation.* Only ``etl_writer`` may write to ``ledger``; everyone else is
read-only. Because of **first-deny-wins** and **filename order**, put the allow *before*
the deny:

.. code-block:: yaml

   # configs/policies/20-ledger-writes.yaml
   api_version: dbguard.io/v1
   kind: DbGuardPolicy
   metadata: { name: ledger-writes }
   spec:
     default_action: allow
     rules:
       - id: ledger.allow-etl
         tables: [ledger]
         users: [etl_writer]
         query: { pattern: '(?i)^\s*(update|insert|delete)\b' }
         actions: [allow]          # the writer hits this and stops
       - id: ledger.deny-writes
         tables: [ledger]
         query: { pattern: '(?i)^\s*(update|insert|delete)\b' }
         actions: [deny, alert]    # anyone else falls through to here

On Kubernetes with Cilium, swap ``users:`` for ``identities:`` to key on workload identity
instead of DB user - see :doc:`/features/identity-access`.

Patterns you will reuse
=======================

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Goal
     - ``query.pattern``
   * - Any ``UPDATE``
     - ``(?i)^\s*update\b``
   * - Any write (U/I/D)
     - ``(?i)^\s*(update|insert|delete)\b``
   * - Assign a column ``col``
     - ``(?i)\bset\b[^;]*\bcol\s*=``
   * - Set ``col`` to literal ``N``
     - ``(?i)\bset\b[^;]*\bcol\s*=\s*N\b``
   * - ``DROP`` / ``TRUNCATE`` (DDL)
     - ``(?i)^\s*(drop|truncate)\b``
   * - ``GRANT`` / ``ALTER ROLE`` (privilege change)
     - ``(?i)\b(grant|alter\s+role|create\s+role)\b``

.. tip::

   RE2 has **no backreferences or look-around**. Keep patterns anchored (``^\s*``) and use
   ``[^;]*`` rather than ``.*`` so a rule can't accidentally reach across a second
   statement. Prefer a ``tables`` filter + a focused ``pattern`` over one giant regex - it
   is easier to read, and the table filter keeps the rule from firing on unrelated SQL.

Authoring workflow
==================

1. Write the rule in a new, late-enough-numbered file (see ordering in
   :doc:`/features/policy-dsl`).
2. ``dbguard-cli validate-policy configs/policies`` - catches schema and regex errors.
   Good in CI.
3. ``dbguard-cli dry-run --engine postgres --sql "..."`` for both a statement that
   **should** trip and one that **should not** - most "my rule didn't work" surprises are a
   matching-semantics issue a dry-run reveals instantly.
4. Roll out by copying the file into ``policies.dirs`` (hot-reloads). Start in observe mode
   (``actions: [alert]``), watch ``rule_matches_total{rule}`` in metrics, then flip to
   ``[deny, alert]`` once you trust it.

Enforcement reminder
====================

A ``deny`` is **synchronous and guaranteed only through the inline proxy** - the client
gets ``ERROR: dbguard: query blocked by policy (rule …)`` and the statement never reaches
the database. On the detection plane (eBPF, no proxy) the rule is still evaluated and
audited, but blocking is reactive and depends on ``ebpf.drop_enabled: true``. For a hard
guarantee on writes, route the application through the proxy - see
:doc:`/architecture/enforcement`.
