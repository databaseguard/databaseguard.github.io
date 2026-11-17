======================
Single-Host Deployment
======================

On a bare host or VM, dbguard runs as a privileged daemon (ideally under systemd) next to
your database. It audits every local DB process via eBPF and can front specific
connections with the inline proxy.

Run under systemd
=================

A unit file ships in ``deploy/dbguard.service``:

.. code-block:: ini

   [Unit]
   Description=dbguard database firewall
   After=network-online.target

   [Service]
   ExecStart=/usr/local/bin/dbguardd -config /etc/dbguard/dbguard.yaml
   Restart=on-failure
   # eBPF + uprobe attach require privilege
   AmbientCapabilities=CAP_BPF CAP_SYS_ADMIN CAP_SYS_PTRACE CAP_PERFMON
   # or simply run as root

   [Install]
   WantedBy=multi-user.target

Install and start:

.. code-block:: bash

   sudo make install            # or copy build/bin/* and configs/ into place
   sudo systemctl daemon-reload
   sudo systemctl enable --now dbguard
   sudo systemctl status dbguard
   journalctl -u dbguard -f

Layout
======

A conventional single-host layout:

.. code-block:: text

   /usr/local/bin/dbguardd
   /usr/local/bin/dbguard-cli
   /etc/dbguard/dbguard.yaml          # config
   /etc/dbguard/policies/*.yaml       # policy bundle (hot-reloaded)
   /var/lib/dbguard/audit/*.jsonl     # audit store
   /var/lib/dbguard/bpf/*.o           # compiled eBPF objects

Fronting a connection with the proxy
====================================

To hard-block, point applications at the proxy port instead of the database. With
PostgreSQL on ``5432`` and the proxy on ``6432``:

.. code-block:: yaml

   proxies:
     - { enabled: true, engine: postgres, listen: 0.0.0.0:6432, upstream: 127.0.0.1:5432 }

.. code-block:: bash

   # before:  psql -h db-host -p 5432 ...
   # after:   psql -h db-host -p 6432 ...   # now policy-enforced

MySQL is identical with ``engine: mysql`` and the MySQL ports (e.g. listen ``6433`` →
upstream ``3306``). You can run multiple proxy listeners at once.

Enabling reactive enforcement
=============================

To let the detection plane terminate sessions and quarantine sources (not just audit),
set:

.. code-block:: yaml

   ebpf:
     drop_enabled: true

Remember the detection plane is **reactive** - use it for alerting and for long-running
queries; use the proxy for guaranteed blocking (see :doc:`/architecture/enforcement`).

Avoiding stale listeners
========================

If you restart ``dbguardd`` by hand and a previous instance still holds a proxy port,
free it first:

.. code-block:: bash

   for p in 6432 6433; do
     pid=$(sudo ss -ltnp | grep ":$p " | grep -oP 'pid=\K[0-9]+' | head -1)
     [ -n "$pid" ] && sudo kill -9 "$pid"
   done
