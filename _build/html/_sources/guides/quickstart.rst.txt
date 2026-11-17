==========
Quickstart
==========

This walkthrough starts the daemon, audits a live query, then blocks one - first on the
**detection plane**, then synchronously through the **inline proxy**.

It assumes you have built dbguard (:doc:`installation`) and have a local PostgreSQL
listening on ``5432``.

1. Start the daemon
===================

.. code-block:: bash

   sudo ./build/bin/dbguardd -config configs/dbguard.yaml

On startup the daemon:

- loads the eBPF objects and attaches uprobes to every running PostgreSQL/MySQL process,
- loads every policy file under ``policies.dirs`` and begins watching them for changes,
- opens the audit store, the HTTP API (``:9700``) and Prometheus metrics (``:9701``),
- starts any inline proxy listeners you configured.

You should see lines like::

   msg="ebpf: uprobe attached (SQL query auditing active)" engine=postgres symbol=pg_parse_query
   msg="ebpf: started" collections=7 targets=3
   msg="dbguard started" api=0.0.0.0:9700 metrics=0.0.0.0:9701 ebpf=true

2. Watch queries being audited
==============================

Run any query against your database and read it back from the audit stream:

.. code-block:: bash

   psql -h 127.0.0.1 -U postgres -d appdb -c "SELECT 1"

   # live event stream (Server-Sent Events)
   curl -sN http://127.0.0.1:9700/api/v1/events/stream

   # or the persisted audit log
   tail -n1 /var/lib/dbguard/audit/*.jsonl

A ``query.audit`` event looks like:

.. code-block:: json

   {
     "type": "query.audit",
     "engine": "postgres",
     "sql": "SELECT 1",
     "client_ip": "10.0.0.5",
     "allowed": true,
     "source": "kernel"
   }

3. Block a query (detection plane)
==================================

The default policy bundle already denies classic SQL-injection shapes. Try a UNION
extraction:

.. code-block:: bash

   psql -h 127.0.0.1 -U postgres -d appdb -c "SELECT 1 UNION SELECT 2"

The audit event now shows ``"allowed": false`` with ``"rule_id": "sqli.union"``. With
``ebpf.drop_enabled: true`` the daemon will also *reactively* terminate the backend and
quarantine the source - but a fast query may complete before the kernel event is
processed. For **guaranteed** blocking, use the proxy.

4. Block a query *synchronously* (inline proxy)
===============================================

Enable the proxy in ``configs/dbguard.yaml``:

.. code-block:: yaml

   proxies:
     - enabled: true
       engine: postgres
       listen: 0.0.0.0:6432
       upstream: 127.0.0.1:5432

Restart the daemon, then connect through the proxy's port (``6432``) instead of the
database's:

.. code-block:: bash

   # allowed → returns rows
   psql "host=127.0.0.1 port=6432 user=postgres dbname=appdb" -c "SELECT 1"

   # denied → rejected at the wire, never reaches PostgreSQL
   psql "host=127.0.0.1 port=6432 user=postgres dbname=appdb" -c "SELECT 1 UNION SELECT 2"
   # ERROR:  dbguard: query blocked by policy (rule sqli.union)

5. Write your own rule
======================

Drop a file into the policy directory - it hot-reloads within ~2 seconds, no restart:

.. code-block:: yaml

   # configs/policies/50-my-rule.yaml
   api_version: dbguard.io/v1
   kind: DbGuardPolicy
   metadata: { name: my-rule }
   spec:
     default_action: allow
     rules:
       - id: block.payments-table
         severity: critical
         tables: [payments]
         actions: [deny, alert]

Now any query that touches ``payments`` is denied. Delete the file and it is allowed
again.

Where to go next
================

- :doc:`/features/policy-dsl` - the full rule language.
- :doc:`/architecture/enforcement` - detection vs. proxy, and the kernel actions.
- :doc:`/deployment/kubernetes` - run it as a DaemonSet with Cilium identity.
