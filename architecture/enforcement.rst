=================
Enforcement Model
=================

dbguard can **observe** every query and **block** the ones you connect through the proxy.
Understanding which plane does which is the key to deploying it correctly.

Detection plane: detect, audit, react
======================================

The eBPF detection plane fires *after* the database has begun parsing the statement.
It always **audits** the verdict. What it does *beyond* auditing depends on
``ebpf.drop_enabled``:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - ``drop_enabled``
     - Behaviour on a denied query
   * - ``false`` (default)
     - **Detect + audit only.** The event records ``allowed: false`` and the matched rule;
       the query still runs. Safe for roll-out and observation.
   * - ``true``
     - **Reactive enforcement.** In addition to auditing, the daemon may terminate the
       offending backend and quarantine the source (below).

.. important::

   The detection plane is **reactive**, not synchronous. The uprobe fires at parse time,
   the event travels the ring buffer and pipeline, and only then does the enforcement loop
   act - microseconds later. A sub-millisecond query (``SELECT 1 UNION …``) can **finish
   and return** before the action lands. A long-running query (``SELECT pg_sleep(5)``) is
   interrupted cleanly. For **guaranteed** blocking of any statement, use the inline proxy.

Reactive actions (when ``drop_enabled: true``)
----------------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 22 78

   * - Action
     - Mechanism
   * - Terminate session
     - ``SIGTERM`` to the **PostgreSQL backend** that ran the query - ends just that one
       session. dbguard **refuses** to signal a threaded ``mysqld`` server (that would
       drop every connection); MySQL denials are audited/alerted on this plane.
   * - Quarantine source
     - Add the client IP to a kernel quarantine map so subsequent packets are dropped for
       a TTL (requires the cgroup/XDP drop program to load on your kernel).
   * - Drop / RST
     - ``cgroup_skb`` packet drop or TCP ``RST`` injection, where the corresponding kernel
       program is available.

Enforcement plane: synchronous inline proxy
===========================================

The inline proxy is a protocol-aware man-in-the-middle for the connections you route
through it. It evaluates policy **before** forwarding, so a denied statement is **never
sent to the database** - the client gets a native protocol ``ERROR``:

.. code-block:: text

   ERROR:  dbguard: query blocked by policy (rule sqli.union)

Because the decision happens on the wire, it is **deterministic for every statement**,
fast or slow. This is the plane to put in front of application connections you want to
hard-block. It is protocol-aware for both PostgreSQL and MySQL (handshake, auth
pass-through, ``COM_QUERY`` / simple-query framing, TLS decline so plaintext can be
inspected).

.. note::

   The proxy declines TLS (replies to the client's ``SSLRequest`` so it proceeds in
   plaintext) in order to inspect statements - the correct, safe default for a local
   enforcement proxy. For production TLS you terminate TLS at the proxy.

Choosing a model
================

.. list-table::
   :header-rows: 1
   :widths: 34 33 33

   * - Goal
     - Detection plane (eBPF)
     - Inline proxy
   * - Audit **every** DB process on a host
     - ✅ zero-touch, process-wide
     - ✗ only routed connections
   * - Guaranteed block of a fast query
     - ✗ reactive (may finish first)
     - ✅ synchronous
   * - No change to how apps connect
     - ✅
     - ✗ apps connect to the proxy port
   * - Works for threaded MySQL enforcement
     - detect/alert only
     - ✅ synchronous block
   * - Cilium workload identity on the event
     - ✅
     - ✅

**Recommended deployment:** run the detection plane **everywhere** for ubiquitous
auditing and alerting, and place the **inline proxy** in front of the specific
application→database connections that require hard, synchronous enforcement.

Action catalogue
================

Rules declare one or more ``actions``. How each maps to a mechanism:

.. list-table::
   :header-rows: 1
   :widths: 16 44 40

   * - Action
     - Effect
     - Mechanism
   * - ``allow``
     - Explicit allow; stops deny evaluation.
     - verdict only
   * - ``alert``
     - Annotate the event, route to sinks.
     - user space
   * - ``deny`` / ``drop``
     - Reject the statement.
     - proxy ``ERROR`` (sync) / ``cgroup_skb`` drop (needs ``drop_enabled``)
   * - ``reset``
     - Terminate the client connection.
     - TCP ``RST`` via quarantine map
   * - ``rate_limit``
     - Throttle matching subjects.
     - per-subject sliding window (user space)
   * - ``mask``
     - Redact detected PII in audit/alert output.
     - user space
   * - ``kill``
     - ``SIGTERM`` the offending backend (PostgreSQL).
     - reactive kill path
   * - ``freeze``
     - Freeze the offending cgroup.
     - cgroup v2 freezer
   * - ``quarantine``
     - Put the source on the kernel quarantine map.
     - kernel map + expiry

Enforcement-level actions require the corresponding kernel capability; where a mechanism
is unavailable (no LSM, drop disabled, program failed to verify on your kernel) the daemon
logs the fallback and still **audits and alerts**.
