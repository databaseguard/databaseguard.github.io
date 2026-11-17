======================
SQL-Injection Firewall
======================

The default policy bundle ships a SQL-injection firewall (``20-sql-firewall.yaml``) that
denies the classic injection shapes. Every rule is an ordinary policy rule, so you can
tune, disable, or extend them.

Covered patterns
================

.. list-table::
   :header-rows: 1
   :widths: 24 40 36

   * - Rule id
     - Catches
     - Example that is denied
   * - ``sqli.union``
     - UNION-based column extraction
     - ``SELECT 1 UNION SELECT 2``
   * - ``sqli.tautology``
     - Boolean tautology bypass
     - ``... WHERE 1=1 OR 2=2``
   * - ``sqli.stacked``
     - Stacked / piggy-backed statements
     - ``SELECT 1; DROP TABLE users``
   * - ``sqli.timebased``
     - Time-based blind probes
     - ``SELECT pg_sleep(5)``

Each is a ``query.pattern`` rule with ``actions: [deny, alert]``, for example:

.. code-block:: yaml

   - id: sqli.union
     severity: critical
     query: { pattern: '(?i)union\s+(all\s+)?select' }
     actions: [deny, alert]

   - id: sqli.stacked
     severity: critical
     query: { pattern: ';\s*(drop|delete|truncate|update)\s' }
     actions: [deny, alert]

   - id: sqli.timebased
     severity: high
     query: { pattern: '(?i)(pg_sleep|sleep|benchmark|waitfor)\s*\(' }
     actions: [deny, alert]

Testing it
==========

Through the inline proxy (synchronous - the client sees the error):

.. code-block:: bash

   psql "host=127.0.0.1 port=6432 user=postgres dbname=appdb" \
        -c "SELECT 1 UNION SELECT 2"
   # ERROR:  dbguard: query blocked by policy (rule sqli.union)

On the detection plane the same query is audited with ``allowed: false`` and
``rule_id: sqli.union``; whether the session is also terminated depends on
``ebpf.drop_enabled`` (see :doc:`/architecture/enforcement`).

Tuning and false positives
==========================

- Start in **observe** mode: set every firewall rule to ``actions: [alert]``, run for a
  while, review ``rule_matches_total`` in the metrics, then flip high-confidence rules to
  ``[deny, alert]``.
- Scope a rule to specific ``engines``, ``databases``, ``users``, or ``clients`` if a
  pattern is only dangerous in one context.
- Disable a single rule without deleting it: ``enabled: false``.

Extending the firewall
======================

Add your own patterns in a new file (it hot-reloads):

.. code-block:: yaml

   # configs/policies/21-sqli-extra.yaml
   api_version: dbguard.io/v1
   kind: DbGuardPolicy
   metadata: { name: sqli-extra }
   spec:
     default_action: allow
     rules:
       - id: sqli.comment-evasion
         severity: high
         query: { pattern: '(?i)/\*.*\*/.*(union|select)' }
         actions: [deny, alert]
       - id: sqli.information-schema
         severity: medium
         tables: [information_schema.tables, information_schema.columns]
         actions: [alert]
