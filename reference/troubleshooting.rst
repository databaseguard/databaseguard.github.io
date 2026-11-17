===============
Troubleshooting
===============

A field guide to the symptoms you are most likely to hit, and what each one means.

Query auditing
==============

**"uprobe symbol not found - binary is stripped"**
  The database binary has no ``exec_simple_query`` / symbol table. dbguard still attaches
  to ``pg_parse_query`` (resolved from ``.dynsym``) for core auditing; install the matching
  ``-dbgsym`` package (``sudo make dbgsym``) to widen coverage, then restart.

**"no DB process targets to attach uprobe"**
  No database process was visible when the daemon started. This is a start-up race (e.g.
  the DB came up after dbguard). The process watcher attaches to databases discovered later
  within one tick; if it persists, confirm the database is running and - on Kubernetes -
  that the agent has ``hostPID: true``.

**Events are counted in metrics but absent from ``/api/v1/events``**
  The audit store and the API read the persisted log; a just-run query may not be flushed
  yet. Re-query after a moment, or read the JSON-lines files directly under
  ``audit.dir``.

**Query events have an empty ``client_ip``**
  The client-IP correlation (``tcp_recvmsg`` kprobe) did not record the backend's socket.
  Check that ``ebpf.ports.postgres`` / ``ports.mysql`` match the database's real listen
  port, since the correlation is scoped to DB server-side sockets. (IPv4 is correlated;
  IPv6 database sockets are a known limitation.)

Cilium identity
===============

**``cilium_identity`` is always empty**
  Confirm ``identity.cilium_enabled: true`` and that ``cilium_api_url`` points at the
  **agent socket** (``unix:///var/run/cilium/cilium.sock``), not a TCP health port. On
  Kubernetes ensure the ``/var/run`` hostPath mount is present so the socket is reachable
  in the pod. The log line
  ``cilium enrichment enabled ... api=unix:///var/run/cilium/cilium.sock`` confirms it.

**Identity rule does not fire**
  Identity rules are **fail-closed** - if enrichment is unavailable the rule will not
  match. Verify the event actually carries ``cilium_identity`` first (audit a query from
  the target workload), then confirm the ``identities`` entry matches by id, exact label,
  or substring.

Enforcement
===========

**A denied query still returned rows**
  The detection plane is **reactive**: a fast query can finish before the kernel event is
  processed. Use the inline proxy for guaranteed, synchronous blocking. (Also confirm
  ``ebpf.drop_enabled: true`` if you expected a reactive kill.)

**``relation "x" does not exist`` instead of a policy error**
  That is a **database** error, not a dbguard block - the query was *allowed* through and
  the table simply does not exist. A dbguard block always reads
  ``ERROR: dbguard: query blocked by policy (rule …)``.

Inline proxy
============

**``psql`` hangs when connecting through the proxy**
  Historically this was psql's default ``sslmode=prefer`` tripping the proxy's
  ``SSLRequest`` handling (fixed - the proxy now declines SSL cleanly). If a hang persists,
  it is almost always the proxy failing to reach its **upstream**: confirm ``upstream``
  resolves and is reachable from the dbguard pod/host (see below).

**Every connection through the proxy times out (``EXIT=124``)**
  The proxy accepts but cannot reach ``upstream``. On Kubernetes this is the classic
  ``hostNetwork`` symptom - a host-network pod cannot route to a ``ClusterIP``. The
  DaemonSet ships **without** ``hostNetwork`` for exactly this reason; if you re-enabled
  it, either disable it or point ``upstream`` at a routable address.

**``Connection refused`` on the proxy port**
  The listener is not up. Check ``proxies[].enabled`` and the ``proxy: listening`` log
  line; confirm the ``upstream`` value is a valid ``host:port``.

eBPF load warnings
==================

**"some objects failed to load; continuing with the rest"**
  Some optional programs (certain LSM/XDP/SQLi-drop paths) may fail the verifier on a given
  kernel. dbguard logs the failure and runs with the programs that did load - **auditing
  and the policy engine are unaffected**. The reactive kernel-drop/quarantine path depends
  on those programs, so on a kernel where they do not verify, prefer the inline proxy for
  hard blocking.

General
=======

- ``dbguardd`` must run **privileged** (root or ``CAP_BPF`` + ``CAP_SYS_ADMIN``); without
  it, eBPF load fails and the daemon falls back to a user-space-only mode.
- Use ``dbguard-cli validate-policy`` in CI and ``dbguard-cli dry-run`` to test a rule
  before deploying it - most "my rule didn't work" issues are a matching-semantics
  surprise that a dry-run reveals instantly.
