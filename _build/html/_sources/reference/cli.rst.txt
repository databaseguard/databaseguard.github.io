=============
CLI Reference
=============

dbguard ships two binaries: the daemon ``dbguardd`` and the admin CLI ``dbguard-cli``.

``dbguardd`` - the daemon
=========================

.. code-block:: text

   dbguardd -config <path>

.. list-table::
   :header-rows: 1
   :widths: 24 76

   * - Flag
     - Meaning
   * - ``-config <path>``
     - Path to the YAML configuration (default ``configs/dbguard.yaml``).

On start it loads the eBPF objects, attaches uprobes to every DB process, loads and
watches the policy bundle, opens the audit store, and starts the API, metrics, and any
configured proxy listeners. It must run privileged (root or ``CAP_BPF`` +
``CAP_SYS_ADMIN``).

``dbguard-cli`` - admin & testing
=================================

``validate-policy``
-------------------

Parse and validate a policy bundle without running the daemon. Use it in CI.

.. code-block:: bash

   dbguard-cli validate-policy configs/policies

``dry-run``
-----------

Evaluate a single statement against the loaded policy and print the verdict - no database,
no proxy. Ideal for authoring and for unit-testing rules.

.. code-block:: bash

   dbguard-cli dry-run --engine postgres --sql "SELECT * FROM customers"

   # with a simulated Cilium identity
   dbguard-cli dry-run --engine postgres \
       --identity k8s:app=api-server \
       --labels "k8s:app=api-server,k8s:io.kubernetes.pod.namespace=prod" \
       --sql "SELECT ssn FROM customers"

.. list-table::
   :header-rows: 1
   :widths: 24 76

   * - Flag
     - Meaning
   * - ``--engine``
     - ``postgres`` | ``mysql`` - the subject engine.
   * - ``--sql``
     - The statement to evaluate.
   * - ``--identity``
     - Simulate a Cilium security identity (id or label).
   * - ``--labels``
     - Comma-separated labels to attach to the simulated identity.
   * - ``--user`` / ``--database`` / ``--client``
     - Simulate the subject user, database, and client IP.

The output names the matched rule and the resulting action, so you can confirm a new rule
fires (and that an allow-list lets the right identity through) before deploying it.

Exit codes
==========

.. list-table::
   :header-rows: 1
   :widths: 16 84

   * - Code
     - Meaning
   * - ``0``
     - Success (policy valid / dry-run allowed).
   * - non-zero
     - Validation error, or a dry-run that was denied - handy for scripting policy tests.
