=================================
PII & Sensitive-Column Protection
=================================

dbguard can **deny** or **mask** queries that touch sensitive columns. Two built-in
documents cover the common cases; both are ordinary policy rules you can tune.

.. list-table::
   :header-rows: 1
   :widths: 26 20 54

   * - Rule id (built-in)
     - Action
     - Behaviour
   * - ``pii.cc.tables``
     - ``alert, mask``
     - Flags and masks reads of ``credit_card`` / ``ssn`` / ``salary`` on the
       ``customers`` / ``employees`` tables - **annotates, does not block**.
   * - ``fw.sensitive-columns``
     - ``deny, alert``
     - **Hard-denies** any query referencing ``ssn`` / ``credit_card`` / ``password`` /
       ``card_number``.
   * - ``pii.bulk_select``
     - ``alert``
     - Flags ``SELECT *`` on sensitive tables (alert only).

Deny vs. mask
=============

- **Deny** rejects the whole statement (``actions: [deny, alert]``). On the proxy the
  client gets ``ERROR: dbguard: query blocked by policy (rule …)``.
- **Mask** (``actions: [alert, mask]``) lets the query run but **redacts the detected PII
  in dbguard's own audit / alert output** - it does not rewrite result rows. Combine with
  ``audit.redact_pii: true``.

Protecting a column the right way
=================================

A ``columns`` filter matches **bare** references (``SELECT salary``) but **not** a column
inside a function (``sum(salary)``) - so a column-only rule can be bypassed with an
aggregate. For airtight protection use a ``query.pattern`` that matches the column name
anywhere in the statement:

.. code-block:: yaml

   # catches SELECT salary, sum(salary), WHERE salary > 100000, ...
   - id: emp.block-salary
     severity: critical
     tables: [employees]
     query: { pattern: '(?i)\bsalary\b' }
     actions: [deny, alert]

.. tip::

   This is a genuinely useful rule of thumb: **column-list rules are convenient, a
   ``query`` pattern is stricter.** Use the column list for broad PII tagging/masking, and
   a pattern for a field you must never leak.

Example: lock down an ``employees`` table
=========================================

.. code-block:: yaml

   # configs/policies/20-employees.yaml
   api_version: dbguard.io/v1
   kind: DbGuardPolicy
   metadata: { name: employees }
   spec:
     default_action: allow
     rules:
       - id: emp.block-salary
         severity: critical
         tables: [employees]
         query: { pattern: '(?i)\bsalary\b' }
         actions: [deny, alert]

Test it (through the proxy):

.. code-block:: bash

   P() { psql "host=127.0.0.1 port=6432 user=postgres dbname=appdb" -c "$1"; }

   P "SELECT id, name, department FROM employees"   # ✅ returns rows
   P "SELECT id, name, salary FROM employees"       # ❌ blocked (emp.block-salary)
   P "SELECT department, sum(salary) FROM employees GROUP BY department"  # ❌ blocked too

The first query resolves and prints rows; the other two return
``ERROR: dbguard: query blocked by policy (rule emp.block-salary)`` - same table, same
connection, the only difference being whether the statement touches the protected column.
