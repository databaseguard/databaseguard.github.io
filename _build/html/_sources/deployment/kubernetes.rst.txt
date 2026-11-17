=====================
Kubernetes Deployment
=====================

On Kubernetes, dbguard runs as a **DaemonSet** so one agent per node can uprobe the
database processes on that node, and - with Cilium - bind every query to its workload
identity. This page follows the manifests under ``deploy/k8s`` and ``deploy/cilium``.

Manifests
=========

.. code-block:: text

   deploy/
   ├── Dockerfile                 # builds the image (make bpf go-build)
   ├── k8s/
   │   ├── namespace.yaml
   │   ├── rbac.yaml
   │   ├── configmap.yaml         # dbguard.yaml (mounted at /etc/dbguard)
   │   ├── daemonset.yaml         # the agent
   │   └── service.yaml           # api (9700), metrics (9701), proxy (6432)
   └── cilium/
       └── ciliumnetworkpolicy-postgres.yaml   # L3/L4 micro-seg example

DaemonSet essentials
====================

The agent pod needs:

.. list-table::
   :header-rows: 1
   :widths: 34 66

   * - Setting
     - Why
   * - ``hostPID: true``
     - To see and uprobe database processes running in **other** pods. The agent resolves
       each target binary through ``/proc/<pid>/root/…`` so the uprobe attaches to the
       right inode across mount namespaces.
   * - ``privileged`` / ``CAP_BPF`` + ``CAP_SYS_ADMIN``
     - Load eBPF programs and attach uprobes/kprobes.
   * - hostPath mount of ``/var/run``
     - Makes the Cilium agent socket ``/var/run/cilium/cilium.sock`` reachable for identity
       enrichment.
   * - **No** ``hostNetwork``
     - The agent uses a normal **pod** network identity so the inline proxy can reach the
       database ``ClusterIP`` and sees the real client pod IP. ``hostPID`` is what the
       uprobes need, not ``hostNetwork``.

.. important::

   The ``ConfigMap`` mounts over ``/etc/dbguard``, which **shadows** the image's
   ``/etc/dbguard/policies`` directory. The in-cluster config therefore points
   ``policies.dirs`` at ``/var/lib/dbguard/policies`` - a writable path baked into the
   image. This both keeps the policies loaded *and* lets you hot-load a rule by writing a
   file into the pod.

Build and load the image
========================

.. code-block:: bash

   docker build -t dbguard:latest -f deploy/Dockerfile .
   # for kind:
   kind load docker-image dbguard:latest --name <cluster>

Deploy
======

.. code-block:: bash

   kubectl apply -f deploy/k8s/namespace.yaml
   kubectl apply -f deploy/k8s/rbac.yaml
   kubectl apply -f deploy/k8s/configmap.yaml
   kubectl apply -f deploy/k8s/daemonset.yaml
   kubectl apply -f deploy/k8s/service.yaml

   kubectl -n dbguard rollout status ds/dbguard
   DBG=$(kubectl -n dbguard get pod -l app.kubernetes.io/name=dbguard -o jsonpath='{.items[0].metadata.name}')
   kubectl -n dbguard logs "$DBG" | grep -iE "uprobe attached|cilium enrichment|proxy: listening"

Enable Cilium identity
======================

In the ConfigMap's ``dbguard.yaml``:

.. code-block:: yaml

   identity:
     cilium_enabled: true
     cilium_api_url: unix:///var/run/cilium/cilium.sock

Apply and restart the DaemonSet; the log should show
``cilium enrichment enabled ... api=unix:///var/run/cilium/cilium.sock``, and query events
will carry ``cilium_identity`` + ``cilium_labels``. See :doc:`/architecture/cilium`.

Route an app through the proxy
==============================

The ``dbguard`` Service exposes the proxy port (``6432``). Point your application (or a
test ``psql``) at it instead of the database Service:

.. code-block:: bash

   kubectl -n dbguard exec api-server -- \
     psql "host=dbguard port=6432 user=postgres dbname=appdb" -c "SELECT 1"

Hot-load a policy into the running agent
========================================

Because ``policies.dirs`` is a writable path, you can add a rule without a rebuild:

.. code-block:: bash

   kubectl -n dbguard exec -i "$DBG" -- \
     sh -c 'cat > /var/lib/dbguard/policies/20-demo.yaml' <<'YAML'
   api_version: dbguard.io/v1
   kind: DbGuardPolicy
   metadata: { name: demo }
   spec:
     default_action: allow
     rules:
       - id: demo.block-bonus
         tables: [employees]
         query: { pattern: '(?i)\bbonus\b' }
         actions: [deny, alert]
   YAML
   # applies within ~2s; delete the file to unload it

Micro-segmentation (optional, recommended)
==========================================

Apply the example ``CiliumNetworkPolicy`` so only approved workloads may open a connection
to the database at all - L3/L4 segmentation underneath dbguard's L7 statement control:

.. code-block:: bash

   kubectl apply -f deploy/cilium/ciliumnetworkpolicy-postgres.yaml

See :doc:`/architecture/cilium` for how the two layers combine.

Operational notes
=================

- **Pod IP changes:** if you pin a proxy ``upstream`` to a database **pod IP** (rather than
  its Service), re-point it when the database pod restarts. Prefer the Service DNS name.
- **Image rebuilds** are only needed for Go/eBPF changes; policy and most config changes
  are a ConfigMap apply + rollout (or a hot-reloaded file).
- The **cilium identity API** is on the agent socket, not a TCP port - ensure the
  ``/var/run`` hostPath mount is present.
