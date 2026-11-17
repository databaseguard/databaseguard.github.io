==========================================
dbguard Documentation
==========================================

**An eBPF-based, identity-aware database firewall and query auditor for PostgreSQL and MySQL.**

.. image:: https://img.shields.io/badge/version-1.0.0-blue
   :alt: Version
.. image:: https://img.shields.io/badge/license-Apache--2.0-green
   :alt: License
.. image:: https://img.shields.io/badge/go-1.22+-00ADD8
   :alt: Go
.. image:: https://img.shields.io/badge/kernel-eBPF%20(CO--RE)-orange
   :alt: eBPF
.. image:: https://img.shields.io/badge/cilium-identity--aware-9b59b6
   :alt: Cilium

**dbguard** watches, audits, and enforces policy on every SQL statement that reaches
your database - in the kernel, with eBPF, and (optionally) synchronously in front of the
wire. It combines two complementary enforcement models:

* A **detection plane** built on eBPF ``uprobes`` attached to the database's own parser
  (``pg_parse_query`` for PostgreSQL, ``dispatch_command`` for MySQL). Every query is
  captured, evaluated against policy, audited, and - on Kubernetes with Cilium - stamped
  with the **workload identity** of the client that issued it.
* An **enforcement plane** built as a protocol-aware, synchronous **inline proxy**. A
  denied query is rejected at the wire with a database error *before the engine ever runs
  it*.

On top of both planes sits a single, hot-reloadable **policy engine**: a declarative
YAML DSL for a SQL-injection firewall, PII/sensitive-column protection, rate limiting,
tenant isolation, and **Cilium identity-based access control** ("only the ``api-server``
workload may read this table").

.. code-block:: bash

   # build
   make bpf go-build

   # run the daemon (detection + optional inline proxy)
   sudo ./build/bin/dbguardd -config configs/dbguard.yaml

   # validate / dry-run a policy
   ./build/bin/dbguard-cli validate-policy configs/policies
   ./build/bin/dbguard-cli dry-run --engine postgres --sql "SELECT * FROM customers"

At a glance
===========

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Capability
     - What it does
   * - SQL query auditing
     - eBPF uprobe on the DB parser captures every statement (even over TLS), with the
       client IP, user, database, and - on Cilium - the workload identity.
   * - SQL-injection firewall
     - Pattern rules for UNION extraction, tautologies, stacked queries, and time-based
       probes.
   * - PII / sensitive-column protection
     - Deny or mask queries that touch ``ssn``, ``credit_card``, ``salary`` and friends.
   * - Rate limiting
     - Sliding-window limits per subject (user + client + rule).
   * - Identity-based access control
     - Rules keyed on the client's **Cilium security identity**
       (``identities: [k8s:app=api-server]``).
   * - Synchronous blocking
     - The inline proxy rejects a denied statement at the wire - the engine never sees it.
   * - Hot reload
     - Add, change, or remove policy files at runtime; no restart.

.. toctree::
   :maxdepth: 2
   :caption: Getting Started

   guides/introduction
   guides/installation
   guides/quickstart
   guides/configuration

.. toctree::
   :maxdepth: 2
   :caption: Architecture

   architecture/overview
   architecture/dataflow
   architecture/enforcement
   architecture/cilium

.. toctree::
   :maxdepth: 2
   :caption: Features

   features/policy-dsl
   features/writing-rules
   features/sql-firewall
   features/pii-protection
   features/rate-limiting
   features/identity-access
   features/audit-events

.. toctree::
   :maxdepth: 2
   :caption: Deployment

   deployment/single-host
   deployment/kubernetes

.. toctree::
   :maxdepth: 2
   :caption: Reference

   reference/cli
   reference/policy-schema
   reference/config-reference
   reference/troubleshooting

.. toctree::
   :maxdepth: 1
   :caption: Development

   development/building
   development/changelog


Indices and tables
===================

* :ref:`genindex`
* :ref:`search`
