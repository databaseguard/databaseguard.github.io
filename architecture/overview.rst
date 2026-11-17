=====================
Architecture Overview
=====================

dbguard is a single Go binary (``dbguardd``) that drives a set of eBPF programs in the
kernel and, optionally, one or more user-space proxy listeners. Everything funnels into a
shared **event pipeline** and a single **policy engine**.

Component map
=============

.. code-block:: text

   dbguard/
   ├── cmd/
   │   ├── dbguardd/           # the daemon: wires everything together
   │   └── dbguard-cli/        # validate-policy, dry-run, admin
   ├── bpf/src/                # eBPF C sources (compiled to build/bpf/*.o)
   │   ├── audit.bpf.c         # uprobes on DB parser + tcp_recvmsg client-IP correlation
   │   ├── conn_anomaly.bpf.c  # connection tracking / anomaly kprobes
   │   ├── sqli_firewall.bpf.c # cgroup/XDP SQLi drop path
   │   ├── rids.bpf.c          # runtime integrity / destructive-DDL tracepoints
   │   ├── tenant_guard_lsm.bpf.c
   │   └── microseg_*.bpf.c    # XDP / sockmap micro-segmentation
   ├── internal/
   │   ├── ebpf/               # Layer 1: load objects, attach probes, read ring buffer
   │   │   ├── manager.go      #   collection lifecycle, target (DB process) tracking
   │   │   ├── loader.go       #   per-program attach (uprobe/kprobe/cgroup/xdp)
   │   │   └── decode.go       #   ring-buffer bytes → events.Event
   │   ├── procwatch/          # discover & watch DB processes (postgres/mysqld)
   │   ├── proxy/              # Layer 2: synchronous inline enforcement proxy
   │   │   ├── proxy.go        #   PostgreSQL wire protocol
   │   │   └── mysql.go        #   MySQL wire protocol
   │   ├── pipeline/           # Layer 3: enrich → policy → detect → finalize → fanout
   │   ├── enrich/             # Cilium identity enrichment cache
   │   ├── identity/           # Cilium agent client (cilium.go, resolver.go)
   │   ├── policy/             # Layer 4: the rule DSL, engine, and rate limiter
   │   │   ├── rules.go        #   the YAML schema
   │   │   └── engine.go       #   evaluation, first-deny-wins, hot reload
   │   ├── audit/              # JSON-lines audit store
   │   ├── api/                # HTTP API + Server-Sent-Events stream
   │   └── config/             # configuration loader
   ├── configs/               # dbguard.yaml + policies/
   └── deploy/                # Dockerfile + Kubernetes manifests + CiliumNetworkPolicy

The five layers
===============

.. list-table::
   :header-rows: 1
   :widths: 14 20 66

   * - Layer
     - Package
     - Responsibility
   * - **1. Kernel data plane**
     - ``internal/ebpf`` + ``bpf/``
     - Load the eBPF objects; attach a uprobe to each DB process's parser; attach the
       connection, integrity, and enforcement programs; read events off the ring buffer
       and decode them.
   * - **2. Inline proxy**
     - ``internal/proxy``
     - Terminate the client's wire protocol, evaluate policy *before* forwarding, and
       reject denied statements synchronously. One listener per protocol.
   * - **3. Pipeline**
     - ``internal/pipeline`` + ``internal/enrich``
     - A staged processor every event passes through: **enrich** (Cilium identity) →
       **policy** (evaluate) → **detect** → **finalize** (metrics) → **fanout** (audit,
       alert sinks, SSE subscribers, the enforcement loop).
   * - **4. Policy engine**
     - ``internal/policy``
     - Compile YAML rules into matchers; evaluate them first-deny-wins; maintain the
       sliding-window rate limiter; hot-reload on file change.
   * - **5. Egress**
     - ``internal/audit`` / ``internal/api``
     - Persist the JSON-lines audit log, serve the HTTP API and live event stream, export
       Prometheus metrics.

Process-wide, per-engine attach
===============================

dbguard discovers every running PostgreSQL and MySQL process (``internal/procwatch``) and
attaches the auditing uprobe to each one. PostgreSQL forks a **backend process per
connection**; MySQL is **threaded** (one ``mysqld`` process). Because the uprobe is keyed
on the binary's **inode**, it fires for every forked backend without re-attaching - and
dbguard's enforcement is careful never to signal the whole threaded ``mysqld`` server
(only the specific PostgreSQL backend can be safely terminated).

When a DB process appears **after** the daemon started (a new pod, or a database that
restarted), the process watcher attaches the uprobe to it within one tick - auditing is
resilient to start-up ordering.

The two planes side by side
===========================

.. code-block:: text

                         ┌──────────────────────────────────────────┐
     application ───────▶│  (optional) inline proxy  :6432           │  SYNCHRONOUS
                         │  parse wire → evaluate → forward/reject    │  (hard block)
                         └───────────────┬──────────────────────────┘
                                         │ allowed only
                                         ▼
                         ┌──────────────────────────────────────────┐
                         │          PostgreSQL / MySQL               │
                         │   parser (pg_parse_query / dispatch...)   │
                         └───────────────┬──────────────────────────┘
                                         │ uprobe fires
                                         ▼
                         ┌──────────────────────────────────────────┐
     kernel (eBPF) ─────▶│  audit.bpf.o → ring buffer → pipeline      │  DETECTION
                         │  enrich(identity) → policy → audit/alert   │  (+reactive)
                         └──────────────────────────────────────────┘

See :doc:`dataflow` for the event-by-event walk-through and :doc:`enforcement` for what
each plane can and cannot do.
