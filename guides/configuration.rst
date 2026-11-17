=============
Configuration
=============

dbguard reads a single YAML file (``-config``, default ``configs/dbguard.yaml``). This
page is a tour of the main sections; see :doc:`/reference/config-reference` for the
exhaustive field list.

.. code-block:: yaml

   general:
     name: dbguard
     data_dir: /var/lib/dbguard

   ebpf:
     drop_enabled: false          # true = allow kernel enforcement actions (DROP/kill/quarantine)
     ringbuf_pages: 256
     lsm_enabled: true
     xdp_interfaces: []           # interfaces for XDP micro-seg (empty = off)
     ports:
       postgres: 5432             # the DB listen ports (used to scope socket correlation)
       mysql: 3306

   policies:
     dirs: [/etc/dbguard/policies]
     watch: true                  # hot-reload on file changes

   audit:
     enabled: true
     store: jsonl
     dir: /var/lib/dbguard/audit
     redact_pii: true             # mask detected PII in the audit output

   identity:
     cilium_enabled: false
     cilium_api_url: unix:///var/run/cilium/cilium.sock

   api:
     enabled: true
     listen: 0.0.0.0:9700
   metrics:
     enabled: true
     listen: 0.0.0.0:9701
     path: /metrics

   proxies:
     - enabled: false
       engine: postgres
       listen: 0.0.0.0:6432
       upstream: 127.0.0.1:5432

Key sections
============

``ebpf``
--------

Controls the kernel data plane.

- ``drop_enabled`` - the master switch for **enforcement actions** on the detection plane.
  When ``false`` (the default), denied queries are **detected and audited** but not
  terminated. When ``true``, the daemon may SIGTERM the offending backend and quarantine
  the source (see :doc:`/architecture/enforcement`).
- ``ports.postgres`` / ``ports.mysql`` - the listen ports of your databases. dbguard uses
  these to recognise a DB server-side socket when correlating the client IP onto a query
  event.
- ``xdp_interfaces`` - network interfaces to attach the XDP micro-segmentation program to;
  leave empty unless you are using the XDP path.

``policies``
------------

- ``dirs`` - one or more directories of policy YAML files. Files load in **filename
  order** (``00-`` before ``90-``); sub-directories are ignored.
- ``watch: true`` - the engine watches these directories and **hot-reloads** on any add,
  change, or delete, rebuilding the rule set from scratch.

.. tip::

   In Kubernetes the config is delivered by a ``ConfigMap`` mounted at ``/etc/dbguard``.
   Because that mount shadows the image's ``/etc/dbguard/policies`` directory, the
   container points ``policies.dirs`` at ``/var/lib/dbguard/policies`` instead - a
   writable path baked into the image, which also lets you hot-load a rule by writing a
   file into the pod. See :doc:`/deployment/kubernetes`.

``identity``
------------

- ``cilium_enabled: true`` turns on **Cilium identity enrichment**: dbguard resolves each
  client IP to its Cilium security identity and labels.
- ``cilium_api_url`` - where to reach the Cilium agent. The real agent serves its API over
  a unix socket: ``unix:///var/run/cilium/cilium.sock``. An ``http://host:port`` value is
  also accepted (for a test stub or proxy).

See :doc:`/architecture/cilium`.

``proxies``
-----------

A list of synchronous inline proxy listeners. Each entry:

.. code-block:: yaml

   proxies:
     - enabled: true
       engine: postgres          # postgres | mysql | mariadb
       listen: 0.0.0.0:6432      # clients connect here
       upstream: 127.0.0.1:5432  # the real database

Only ``enabled`` entries with both ``listen`` and ``upstream`` are started. A legacy
singular ``proxy:`` block is also accepted for back-compat.

``audit`` / ``api`` / ``metrics``
---------------------------------

- ``audit`` - the JSON-lines audit store (rotating files under ``dir``). ``redact_pii``
  masks detected sensitive values in the stored events.
- ``api`` - the HTTP API (events query + live SSE stream).
- ``metrics`` - Prometheus endpoint (``events_total`` by type/severity, rule match
  counters, ring-buffer stats).

Minimal configurations
=======================

**Audit-only on a single host** (no blocking):

.. code-block:: yaml

   ebpf: { drop_enabled: false }
   policies: { dirs: [configs/policies], watch: true }
   audit: { enabled: true, store: jsonl, dir: /var/lib/dbguard/audit }

**Synchronous blocking in front of PostgreSQL:**

.. code-block:: yaml

   policies: { dirs: [configs/policies], watch: true }
   proxies:
     - { enabled: true, engine: postgres, listen: 0.0.0.0:6432, upstream: 127.0.0.1:5432 }

**Kubernetes with Cilium identity:**

.. code-block:: yaml

   ebpf: { drop_enabled: true, ports: { postgres: 5432 } }
   policies: { dirs: [/var/lib/dbguard/policies], watch: true }
   identity: { cilium_enabled: true, cilium_api_url: unix:///var/run/cilium/cilium.sock }
