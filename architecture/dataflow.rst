=========
Data Flow
=========

Every query travels one of two paths. Both converge on the same **pipeline** and **policy
engine**.

Detection path (eBPF uprobe)
============================

.. code-block:: text

   client ──SQL──▶ PostgreSQL/MySQL backend
                        │
                        │  backend reads the query bytes      ┌─ kprobe: tcp_recvmsg ─┐
                        │  off its client socket  ───────────▶│ record client tuple   │
                        │                                      │ keyed by backend tgid │
                        ▼                                      └───────────┬───────────┘
                 parser symbol fires                                       │
                 (pg_parse_query / dispatch_command)                       │
                        │  UPROBE                                          │ lookup
                        ▼                                                   │
                 audit.bpf.o builds a query event  ◀───────────────────────┘
                        │   (sql, comm, pid, saddr/daddr/ports, flags)
                        ▼
                 ring buffer ──▶ internal/ebpf/decode.go ──▶ events.Event
                        │
                        ▼
                 ┌──────────────── pipeline (internal/pipeline) ─────────────────┐
                 │ enrich  → resolve client IP to Cilium identity + labels       │
                 │ policy  → engine.Evaluate(ctx) → Decision                     │
                 │ detect  → annotate severity / findings                        │
                 │ finalize→ metrics (events_total, rule matches)                │
                 │ fanout  → audit store | alert sinks | SSE | enforcement loop  │
                 └───────────────────────────────────────────────────────────────┘

The **client IP correlation** is the subtle part. A uprobe on the parser has no socket
context, so a companion ``kprobe`` on ``tcp_recvmsg`` - which runs in the backend's own
context right before it parses - records the backend's client tuple (remote address,
ports) keyed by the backend's thread-group id. ``submit_query`` then stamps that tuple
onto the query event. This is what lets a kernel-captured statement carry a **client IP**,
which the pipeline then resolves to a **Cilium identity**.

Enforcement path (inline proxy)
===============================

.. code-block:: text

   client ──wire protocol──▶ proxy :6432 (internal/proxy)
                                  │
                                  │ 1. handle SSLRequest / startup (decline TLS, inspect plaintext)
                                  │ 2. frame each client message; extract the SQL of a Query
                                  │ 3. resolve client IP → Cilium identity (if enabled)
                                  ▼
                           engine.Evaluate(ctx)
                             │                 │
                     allowed │                 │ denied
                             ▼                 ▼
                 forward bytes to          write a protocol ERROR
                 upstream DB, relay        back to the client;
                 results back              NOTHING is forwarded
                             │                 │
                             ▼                 ▼
                    DB executes query     query.decision event → pipeline (audit/alert)

On the proxy path the verdict is **synchronous**: a denied statement never reaches the
database, and the client receives a native ``ERROR`` (e.g.
``dbguard: query blocked by policy (rule sqli.union)``). Allowed statements are forwarded
untouched; the database then executes them and - because the query still reaches the
engine - the **uprobe also fires**, so the same statement is independently audited on the
detection plane.

The pipeline stages
===================

Each event, from either path, passes through the stages in ``internal/pipeline``:

.. list-table::
   :header-rows: 1
   :widths: 16 84

   * - Stage
     - What it does
   * - ``enrich``
     - If ``identity.cilium_enabled``, look up the event's ``client_ip`` in the Cilium
       agent and stamp ``cilium_identity`` + ``cilium_labels`` onto the event (cached,
       with a bounded TTL).
   * - ``policy``
     - Build a ``policy.Context`` (engine, db, user, client IP, identity, parsed tables/
       columns, SQL text) and call ``engine.Evaluate`` → ``Decision`` (allowed, matched
       rule id, actions, severity).
   * - ``detect``
     - Attach severity and any findings; mark the event allowed/denied.
   * - ``finalize``
     - Increment Prometheus counters (``events_total`` by type+severity, per-rule match
       counts).
   * - ``fanout``
     - Publish the event to: the **audit** store, configured **alert** sinks, any live
       **SSE** subscribers, and the **enforcement loop** (which may act on a denial).

Policy evaluation, briefly
==========================

The engine evaluates rules in **filename then list order**, and the **first rule that
yields a denying action wins** (``deny`` / ``drop`` / ``reset`` / ``kill`` / ``freeze`` /
``quarantine``). Alert- and mask-only rules annotate the event and let evaluation
continue. If no rule denies, ``spec.default_action`` applies. The rate limiter advances
per matching rule, keyed by *subject* (user + client + rule). See
:doc:`/features/policy-dsl` and :doc:`/reference/policy-schema`.
