===================
Cilium Integration
===================

On Kubernetes with `Cilium <https://cilium.io/>`_, dbguard binds every query to the
**security identity** of the workload that issued it. This turns an IP-based audit trail
into an **identity-aware** one, and unlocks policy written in terms of workloads rather
than addresses.

.. code-block:: text

   L3 identity  ──────────────▶  L7 statement
   "k8s:app=api-server"           "SELECT ssn FROM customers"
   (Cilium security identity)     (captured by the eBPF uprobe)
                   \             /
                    dbguard binds them together on every event

Why identity, not IP
====================

In a cluster the source IP of a database connection is ephemeral and shared across pods;
it tells you almost nothing about *who* is querying. Cilium assigns each workload a stable
**security identity** derived from its Kubernetes labels (e.g.
``k8s:app=api-server``). dbguard resolves the client IP of each query to that identity, so:

- audit events carry ``cilium_identity`` and the full label set, and
- policy rules can **allow or deny by identity** (see :doc:`/features/identity-access`).

How enrichment works
====================

When ``identity.cilium_enabled: true``:

.. code-block:: text

   query event (client_ip = 10.244.0.160)
        │
        ▼
   enrich stage ──▶ identity.resolver.LookupForIP(ip)
        │                 │
        │                 ▼
        │        Cilium agent API over the unix socket
        │        GET /v1/endpoint  (via /var/run/cilium/cilium.sock)
        │                 │
        │                 ▼
        │        match ip to an endpoint's status.networking.addressing[]
        │        → status.identity.id + labels
        ▼
   event.cilium_identity = "44982"
   event.cilium_labels   = ["k8s:app=api-server", ...]

The resolver talks to the **real Cilium agent** over its unix socket
(``unix:///var/run/cilium/cilium.sock``) and reads the **endpoint list**
(``GET /v1/endpoint``), matching the query's client IP to an endpoint's addressing and
reading that endpoint's identity id and labels. Results are cached with a bounded TTL so
an agent hiccup can never turn the hot path into a hot loop. An ``http://host:port`` base
URL is also supported for a test stub.

.. note::

   The agent's identity API is served on the unix socket, **not** on the cilium-health
   port. On Kubernetes the dbguard DaemonSet mounts ``/var/run`` from the host, so
   ``/var/run/cilium/cilium.sock`` is reachable inside the pod.

Identity-based access control
=============================

Once the identity is on the event, a rule can gate on it:

.. code-block:: yaml

   spec:
     default_action: allow
     rules:
       - id: db.only-api-server
         identities: [k8s:app=api-server]   # match by id or label (substring ok)
         actions: [deny, alert]             # e.g. deny everyone... see identity-access

See :doc:`/features/identity-access` for allow-list patterns and the matching semantics
(by numeric id, exact label, or label substring; fail-closed).

Micro-segmentation (defence in depth)
=====================================

dbguard's L7 statement control pairs naturally with Cilium's L3/L4
**micro-segmentation**. The repo ships an example ``CiliumNetworkPolicy`` that locks the
database so that **only approved workloads can open a connection at all**:

.. code-block:: yaml

   # deploy/cilium/ciliumnetworkpolicy-postgres.yaml (excerpt)
   endpointSelector:
     matchLabels: { k8s:app: postgres }
   ingress:
     - fromEndpoints:
         - matchLabels: { k8s:app: api-server }
       toPorts:
         - ports: [ { port: "5432", protocol: TCP } ]

With both in place you get two independent layers: Cilium decides *who may connect*
(L3/L4), and dbguard decides *what they may run* (L7) - each query stamped with the
identity that ran it.

End-to-end, on a live cluster
=============================

The combination has been validated on a ``kind`` + Cilium cluster: a query from the
``api-server`` pod is audited as:

.. code-block:: json

   {
     "sql": "SELECT 3",
     "client_ip": "10.244.0.160",
     "cilium_identity": "44982",
     "cilium_labels": ["k8s:app=api-server", "k8s:io.kubernetes.pod.namespace=prod"]
   }

- the complete L3-identity → L7-statement binding. See :doc:`/deployment/kubernetes` to
reproduce it.
