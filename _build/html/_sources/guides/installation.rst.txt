============
Installation
============

dbguard ships as a Go daemon (``dbguardd``), a CLI (``dbguard-cli``), and a set of
compiled eBPF objects. This page covers building from source on a Linux host; for
containers and Kubernetes see :doc:`/deployment/kubernetes`.

Requirements
============

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Requirement
     - Notes
   * - Linux kernel 5.8+ (6.x recommended)
     - eBPF ``CO-RE``, ``BTF``, ring buffer, cgroup/LSM hooks. Developed on Ubuntu 24.04.
   * - Go 1.22+
     - Builds ``dbguardd`` and ``dbguard-cli``.
   * - clang / llvm, libelf, linux headers
     - Compile the BPF objects (``make bpf``).
   * - Root / ``CAP_BPF`` + ``CAP_SYS_ADMIN``
     - Loading eBPF programs and attaching uprobes requires privilege.
   * - PostgreSQL 14–16 and/or MySQL 8.0
     - The databases you want to protect.

.. note::

   For **SQL query auditing** the daemon attaches a uprobe to a symbol inside the database
   binary (``pg_parse_query`` / ``dispatch_command``). Distribution packages are often
   stripped; dbguard resolves the symbol from ``.dynsym`` when possible and falls back to
   debug symbols. Installing the matching ``-dbgsym`` package widens coverage - see
   :doc:`/reference/troubleshooting`.

Install the build toolchain (Ubuntu)
====================================

.. code-block:: bash

   sudo apt-get update
   sudo apt-get install -y \
       clang llvm libelf-dev linux-headers-$(uname -r) \
       build-essential pkg-config golang-go

Build from source
=================

.. code-block:: bash

   git clone https://github.com/databaseguard/dbguard.git
   cd dbguard

   # 1. compile the eBPF objects into build/bpf/
   make bpf

   # 2. compile the daemon + CLI into build/bin/
   make go-build

   # or both at once
   make bpf go-build

Artifacts land in ``build/``:

.. code-block:: text

   build/
   ├── bin/
   │   ├── dbguardd          # the daemon
   │   └── dbguard-cli       # policy validation / dry-run / admin
   └── bpf/
       ├── audit.bpf.o       # SQL uprobes + client-IP correlation
       ├── conn_anomaly.bpf.o
       ├── rids.bpf.o
       ├── sqli_firewall.bpf.o
       └── ...               # one object per capability

Optional: debug symbols for stripped DB binaries
=================================================

If query auditing reports *"uprobe symbol not found - binary is stripped"*, install the
matching debug-symbol package so dbguard can resolve the parser symbol:

.. code-block:: bash

   # PostgreSQL example (Debian/Ubuntu ddebs)
   sudo make dbgsym      # installs postgresql-NN-dbgsym, then restart dbguardd

Verify
======

.. code-block:: bash

   # the binaries exist and run
   ./build/bin/dbguardd --help
   ./build/bin/dbguard-cli --help

   # the policy bundle is valid
   ./build/bin/dbguard-cli validate-policy configs/policies

Next: :doc:`quickstart`.
