=======================
Audit, Events & Metrics
=======================

Every query dbguard sees becomes an **event**. Events are persisted to a JSON-lines audit
store, exposed over an HTTP API (query + live stream), and summarised as Prometheus
metrics.

The audit store
===============

With ``audit.enabled: true`` events are appended to rotating JSON-lines files under
``audit.dir`` (default ``/var/lib/dbguard/audit``):

.. code-block:: bash

   tail -f /var/lib/dbguard/audit/*.jsonl

A ``query.audit`` event (detection plane):

.. code-block:: json

   {
     "id": "a162f05509e9f5b5",
     "type": "query.audit",
     "time": "2026-11-17T12:39:02Z",
     "severity": "info",
     "allowed": true,
     "source": "kernel",
     "engine": "postgres",
     "sql": "SELECT id, name FROM employees",
     "client_ip": "10.244.0.160",
     "client_port": 59986,
     "server_ip": "10.244.0.170",
     "server_port": 5432,
     "pid": 98521,
     "cilium_identity": "44982",
     "cilium_labels": ["k8s:app=api-server", "..."]
   }

A denied query carries ``"allowed": false`` and the matched ``rule_id`` (and, for a
proxy-side block, ``"source": "proxy"`` with event type ``policy.decision``). With
``audit.redact_pii: true`` any detected sensitive values are masked in the stored event.

The HTTP API
============

With ``api.enabled: true`` the daemon serves (default ``:9700``):

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Endpoint
     - Returns
   * - ``GET /api/v1/events``
     - Recent events, filterable by ``type``, ``client_ip``, ``min_severity``, ``user``,
       ``engine``, ``database``, ``limit``.
   * - ``GET /api/v1/events/stream``
     - A live **Server-Sent-Events** stream of events as they happen.

.. code-block:: bash

   # query the stored audit log
   curl -s 'http://127.0.0.1:9700/api/v1/events?type=query.audit&limit=20'

   # live stream (Ctrl-C to stop)
   curl -sN http://127.0.0.1:9700/api/v1/events/stream

A compact live monitor, colour-coding allow vs. deny:

.. code-block:: bash

   curl -sN http://127.0.0.1:9700/api/v1/events/stream | python3 -u -c '
   import sys, json
   for line in sys.stdin:
       line = line.strip().removeprefix("data:").strip()
       if not line.startswith("{"): continue
       e = json.loads(line)
       if e.get("type") not in ("query.audit", "policy.decision"): continue
       v = "ALLOW" if e.get("allowed") else "DENY "
       print(v, e.get("rule_id") or "-", "|", (e.get("sql") or "")[:60])'

Metrics
=======

With ``metrics.enabled: true`` a Prometheus endpoint is served (default ``:9701/metrics``):

.. code-block:: bash

   curl -s http://127.0.0.1:9701/metrics | grep -E '^(events_total|rule_matches_total)'

.. list-table::
   :header-rows: 1
   :widths: 34 66

   * - Metric
     - Meaning
   * - ``events_total{type,severity}``
     - Count of events by type (``query.audit``, ``conn.open`` …) and severity.
   * - ``rule_matches_total{rule}``
     - How often each rule matched (even alert-only rules) - your tuning signal.

.. tip::

   ``rule_matches_total`` is the number to watch during a staged roll-out: run the firewall
   in ``alert`` mode, let ``rule_matches_total`` accumulate for a week, and only then flip
   high-confidence, low-false-positive rules to ``deny``.
