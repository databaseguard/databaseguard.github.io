============
Introduction
============

What is dbguard?
================

**dbguard** is a single Go daemon that sits between your applications and your databases
and answers one question for every SQL statement: *should this query be allowed, and who
is really asking?*

It is built for **PostgreSQL** and **MySQL / MariaDB** on modern Linux (developed and
tested on Ubuntu 24.04, kernel 6.x). It is **identity-aware**: when deployed on
Kubernetes with `Cilium <https://cilium.io/>`_, every audited query is bound to the
**security identity** of the workload that issued it, so policy can be written in terms of
*workloads* ("the ``api-server`` pod") rather than brittle IP addresses.

The problem it solves
=====================

Traditional database security lives *inside* the database (``pg_hba.conf``, ``GRANT``,
row-level security) or *in front of it* as a separate WAF appliance. Both have blind
spots:

- The database sees a **connection**, not a *workload*. In a Kubernetes cluster the source
  IP is ephemeral and shared; "who ran this query" is lost.
- Application-layer firewalls terminate TLS and re-parse the wire protocol, adding a hop
  and a parser of their own.
- Query auditing via the database's own logging is after-the-fact and easy to disable.

dbguard observes the query **at the database's own parser**, in the kernel, using eBPF -
so it sees exactly what the engine sees (even for TLS-encrypted connections), with
negligible overhead and nothing to bolt onto the application.

Two planes, one policy
======================

dbguard is best understood as **two enforcement planes that share one policy engine**:

.. list-table::
   :header-rows: 1
   :widths: 18 42 40

   * - Plane
     - How it sees queries
     - Enforcement
   * - **Detection** (eBPF)
     - A ``uprobe`` on the DB parser fires for every statement the engine parses.
     - **Detect + audit + reactive.** Records the verdict, alerts, and can *reactively*
       terminate the offending backend / quarantine the source. Cannot un-run a query
       that finished before the kernel event was processed.
   * - **Enforcement** (inline proxy)
     - Clients connect to the proxy instead of the DB; it parses the wire protocol.
     - **Synchronous.** A denied statement is rejected with a database ``ERROR`` and is
       **never forwarded** to the engine.

Use the detection plane for **ubiquitous, zero-touch auditing and alerting** across every
database process on a host; add the inline proxy in front of the connections you want to
**hard-block**.

Core capabilities
=================

- **SQL query auditing** - every statement captured with SQL text, user, database, client
  IP/port, PID, cgroup, and (on Cilium) the ``cilium_identity`` + labels.
- **SQL-injection firewall** - UNION extraction, boolean tautologies, stacked queries,
  time-based blind probes.
- **PII / sensitive-column protection** - deny or mask queries touching protected columns
  (``ssn``, ``credit_card``, ``salary`` …).
- **Rate limiting** - sliding-window limits keyed per subject (user + client + rule).
- **Tenant isolation** - deny cross-tenant table access.
- **Identity-based access control** - allow/deny by Cilium security identity.
- **Micro-segmentation** - ships an example ``CiliumNetworkPolicy`` to lock the database
  to approved workloads at L3/L4, complementing dbguard's L7 statement control.
- **Hot-reloadable policy** - drop a YAML file into the policy directory; it is picked up
  within a couple of seconds, no restart.
- **Audit + metrics + API** - JSON-lines audit store, Prometheus metrics, and an HTTP
  API with a live Server-Sent-Events stream.

What it is *not*
================

- It is **not** a replacement for database authentication or ``GRANT``\ s - it is a layer
  on top of them.
- The detection (uprobe) plane is **not** a synchronous blocker for sub-millisecond
  queries; that is the inline proxy's job (see :doc:`/architecture/enforcement`).
- It does **not** decrypt or store your data; it inspects the **statement**, not result
  rows (though it can *mask* detected PII in its own audit/alert output).

Where to go next
================

- :doc:`installation` - build dbguard and its eBPF objects.
- :doc:`quickstart` - run the daemon and block your first query.
- :doc:`/architecture/overview` - the component map and data flow.
- :doc:`/features/policy-dsl` - write your own rules.
