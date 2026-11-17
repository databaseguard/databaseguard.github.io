=========
Changelog
=========

Version 1.0.0
=============

First complete release - a single eBPF-based, identity-aware database firewall for
PostgreSQL and MySQL.

**Detection plane**

- eBPF uprobe auditing on the DB parser (``pg_parse_query`` for PostgreSQL,
  ``dispatch_command`` for MySQL 8.0), process-wide and inode-keyed.
- Cross-mount-namespace attach (resolves the target binary via ``/proc/<pid>/root/…``), so
  a DaemonSet agent can uprobe database processes in other pods.
- Resilient attach: databases discovered **after** start-up get their uprobe attached by
  the process watcher within one tick.
- Client-IP correlation: a ``tcp_recvmsg`` kprobe records the backend's client socket tuple
  and stamps it onto the query event (gated on the DB local port for kernel-portability).

**Enforcement plane**

- Synchronous inline proxy for PostgreSQL and MySQL - protocol-aware, auth pass-through,
  denied statements rejected at the wire with a native ``ERROR``.
- Correct ``SSLRequest`` / ``GSSENCRequest`` handling so stock clients (psql default
  ``sslmode=prefer``) connect without ``sslmode=disable``.

**Policy engine**

- Declarative YAML DSL: subject filters (engines/databases/users/clients/identities),
  object filters (tables/columns), content matcher (pattern/keywords), rate limiting, and
  a full action catalogue.
- First-deny-wins evaluation; filename-ordered loading; hot reload that rebuilds the rule
  set from scratch (so deletions truly unload).
- Sliding-window rate limiter keyed per subject.

**Cilium integration**

- Identity enrichment via the real Cilium agent socket (``GET /v1/endpoint`` over
  ``/var/run/cilium/cilium.sock``), binding each query to its workload security identity.
- Identity-based access-control rules (``identities:``) enforced on both planes.
- Example ``CiliumNetworkPolicy`` for L3/L4 micro-segmentation.

**Operations**

- JSON-lines audit store, HTTP API with a live SSE stream, Prometheus metrics.
- Kubernetes DaemonSet (hostPID, no hostNetwork), ConfigMap-driven policy with a writable,
  hot-loadable policy directory.
- ``dbguard-cli`` with ``validate-policy`` and ``dry-run`` (including simulated identity).

.. note::

   Known limitations in 1.0.0: the detection-plane reactive kill cannot block a
   sub-millisecond query (use the proxy); IPv6 database sockets are not yet correlated for
   client IP; some optional LSM/XDP kernel-drop programs may not verify on every kernel
   (auditing and the proxy are unaffected).
