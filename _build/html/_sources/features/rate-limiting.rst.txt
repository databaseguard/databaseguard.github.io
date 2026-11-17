=============
Rate Limiting
=============

A rule with a ``rate`` block throttles matching queries. The limiter is a **sliding
window** keyed per *subject* - the combination of **user + client + rule** - so one
noisy client cannot exhaust another's budget, and two different rules count independently.

.. code-block:: yaml

   - id: exfil.bulk
     severity: medium
     query: { pattern: '(?i)select\s+\*.*from' }
     rate: { events: 3, window: 10s }
     actions: [alert, rate_limit]

- ``events`` - the maximum number of matches allowed per window.
- ``window`` - the window length (``s`` / ``m`` / ``h``).

When the window is **not yet full** the query passes; once the count exceeds ``events``
within the window, the rule's ``actions`` apply (e.g. ``deny`` or ``rate_limit``). Further
matches inside the same window are suppressed to a single alert rather than N.

Example: cap a tagged query
===========================

.. code-block:: yaml

   # configs/policies/90-rate.yaml
   - id: emp.rate-select
     severity: high
     tables: [employees]
     query: { keywords: ["RATECHECK"] }
     rate: { events: 3, window: 10s }
     actions: [deny, alert]

Test it through the proxy - the first three succeed, then it throttles:

.. code-block:: bash

   for i in 1 2 3 4 5; do
     psql "host=127.0.0.1 port=6432 user=postgres dbname=appdb" \
          -c "SELECT id,name FROM employees WHERE id>0 /*RATECHECK*/"
   done

::

   attempt 1  → rows
   attempt 2  → rows
   attempt 3  → rows
   attempt 4  → ERROR: dbguard: query blocked by policy (rule emp.rate-select)
   attempt 5  → ERROR: dbguard: query blocked by policy (rule emp.rate-select)

.. note::

   Because the window is sliding, bursts fired back-to-back can keep the window saturated
   - wait out the ``window`` for the count to decay before expecting the allow/deny pattern
   to repeat.

Choosing a key and a limit
==========================

- The subject key is **user + client + rule**. Add ``clients`` / ``users`` filters to
  scope *who* a limit applies to (e.g. rate-limit only a reporting service account).
- Pair ``rate`` with ``deny`` for a hard cap, or with ``rate_limit`` / ``alert`` for a
  softer throttle that only annotates.
- Put rate rules in a **late-numbered file** (``90-``) so they evaluate after specific
  deny rules; first-deny-wins means an early specific rule still takes precedence.

Built-in rate rules
===================

The default bundle includes broad protective limits, for example a per-client ceiling and
a bulk-``SELECT *`` throttle. Review them in ``configs/policies/`` and tune the thresholds
to your workload before enabling ``deny`` actions in production.
