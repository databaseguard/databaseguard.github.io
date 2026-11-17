=============================
Identity-Based Access Control
=============================

On Kubernetes with Cilium, dbguard can allow or deny queries by the **workload identity**
of the client - "only the ``api-server`` pod may read this table" - instead of by IP.
This is the feature that turns dbguard from an auditor into an identity-aware firewall.

Prerequisites
=============

- ``identity.cilium_enabled: true`` and a reachable Cilium agent socket
  (``cilium_api_url: unix:///var/run/cilium/cilium.sock``). See :doc:`/architecture/cilium`.
- The daemon enriches each event with ``cilium_identity`` + ``cilium_labels``; policy then
  matches on them.

The ``identities`` filter
=========================

A rule's ``identities`` list matches the client's Cilium identity. Each entry can be:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Entry
     - Matches
   * - ``"44982"``
     - The numeric Cilium security identity id exactly.
   * - ``"k8s:app=api-server"``
     - An exact label on the identity.
   * - ``"api-server"``
     - A label **substring** (case-insensitive).

Matching is **fail-closed**: if identity enrichment is unavailable for an event, an
``identities`` rule does not spuriously match.

Allow-list pattern
==================

The idiomatic "only workload X may touch table Y" is a **default-deny scoped to the
table**, with an allow for the permitted identity:

.. code-block:: yaml

   api_version: dbguard.io/v1
   kind: DbGuardPolicy
   metadata: { name: payments-access }
   spec:
     default_action: allow            # the rest of the DB is unaffected
     rules:
       # allow the billing workload to use payments
       - id: payments.allow-billing
         tables: [payments]
         identities: [k8s:app=billing]
         actions: [allow]
       # deny everyone else on payments
       - id: payments.deny-others
         tables: [payments]
         actions: [deny, alert]

Because of **first-match / first-deny-wins** and filename ordering, put the allow rule
*before* the deny (same file, earlier in the list, or a lower-numbered file). The billing
identity hits ``allow`` and stops; any other identity falls through to ``deny``.

Deny a specific identity
========================

The inverse - block one workload - is a single rule:

.. code-block:: yaml

   - id: id.deny-reporting
     identities: [k8s:app=reporting]
     query: { keywords: ["bonus"] }   # optionally scope it
     actions: [deny, alert]

Testing on a live cluster
=========================

Load the rule into the DaemonSet's policy directory (it hot-reloads), then run a query
from the target workload:

.. code-block:: bash

   # from the api-server pod, through the dbguard proxy
   kubectl -n dbguard exec api-server -- \
     psql "host=dbguard port=6432 user=postgres dbname=appdb" -c "SELECT 'IDTEST'"
   # ERROR: dbguard: query blocked by policy (rule id.deny-api-server)

The audit event shows the verdict **and** the identity that produced it:

.. code-block:: json

   {
     "allowed": false,
     "rule_id": "id.deny-api-server",
     "cilium_identity": "44982",
     "cilium_labels": ["k8s:app=api-server", "k8s:io.kubernetes.pod.namespace=prod"]
   }

- the query was blocked **purely because of its Cilium identity**, not its SQL or its IP.

Where it enforces
=================

Identity rules evaluate on **both** planes:

- On the **inline proxy** they block synchronously (the client gets the error above).
- On the **detection plane** they are audited (and reactively enforced if
  ``drop_enabled``).

The proxy resolves the client's identity from its real accepted TCP connection; the
detection plane resolves it from the client IP correlated onto the uprobe event.
