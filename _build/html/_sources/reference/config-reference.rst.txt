=======================
Configuration Reference
=======================

Every field of ``dbguard.yaml``. For the narrative tour see :doc:`/guides/configuration`.

``general``
===========

.. list-table::
   :header-rows: 1
   :widths: 24 16 60

   * - Field
     - Type
     - Meaning
   * - ``name``
     - string
     - Instance name (appears in events/metrics).
   * - ``data_dir``
     - path
     - Base data directory (default ``/var/lib/dbguard``).

``ebpf``
========

.. list-table::
   :header-rows: 1
   :widths: 24 16 60

   * - Field
     - Type
     - Meaning
   * - ``drop_enabled``
     - bool
     - Master switch for detection-plane enforcement actions. ``false`` = detect/audit
       only.
   * - ``ringbuf_pages``
     - int
     - Ring-buffer size in pages.
   * - ``lsm_enabled``
     - bool
     - Load the LSM-based programs (tenant guard, file-open integrity) where supported.
   * - ``xdp_interfaces``
     - []string
     - Interfaces to attach the XDP micro-seg program to (empty = off).
   * - ``ports.postgres``
     - int
     - PostgreSQL listen port, used to recognise a DB server socket for client-IP
       correlation.
   * - ``ports.mysql``
     - int
     - MySQL listen port (same purpose).

``policies``
============

.. list-table::
   :header-rows: 1
   :widths: 24 16 60

   * - Field
     - Type
     - Meaning
   * - ``dirs``
     - []path
     - Directories of policy YAML (loaded in filename order; sub-dirs ignored).
   * - ``watch``
     - bool
     - Hot-reload on add/change/delete.

``audit``
=========

.. list-table::
   :header-rows: 1
   :widths: 24 16 60

   * - Field
     - Type
     - Meaning
   * - ``enabled``
     - bool
     - Persist events.
   * - ``store``
     - string
     - Store backend (``jsonl``).
   * - ``dir``
     - path
     - Directory for rotating JSON-lines files.
   * - ``redact_pii``
     - bool
     - Mask detected PII values in stored events.

``identity``
============

.. list-table::
   :header-rows: 1
   :widths: 24 16 60

   * - Field
     - Type
     - Meaning
   * - ``cilium_enabled``
     - bool
     - Resolve client IPs to Cilium security identities.
   * - ``cilium_api_url``
     - string
     - ``unix:///var/run/cilium/cilium.sock`` (the real agent) or ``http://host:port`` (a
       stub/proxy).

``api`` / ``metrics``
=====================

.. list-table::
   :header-rows: 1
   :widths: 24 16 60

   * - Field
     - Type
     - Meaning
   * - ``api.enabled`` / ``api.listen``
     - bool / addr
     - HTTP API (events + SSE). Default ``0.0.0.0:9700``.
   * - ``metrics.enabled`` / ``metrics.listen`` / ``metrics.path``
     - bool / addr / path
     - Prometheus endpoint. Default ``0.0.0.0:9701`` ``/metrics``.

``proxies``
===========

A list; each entry is an inline proxy listener. A legacy singular ``proxy:`` block is also
accepted.

.. list-table::
   :header-rows: 1
   :widths: 24 16 60

   * - Field
     - Type
     - Meaning
   * - ``enabled``
     - bool
     - Start this listener.
   * - ``engine``
     - string
     - ``postgres`` | ``mysql`` | ``mariadb`` (defaults to ``postgres``).
   * - ``listen``
     - addr
     - Address clients connect to, e.g. ``0.0.0.0:6432``.
   * - ``upstream``
     - addr
     - The real database, e.g. ``127.0.0.1:5432`` or ``postgres:5432``.

Only entries that are ``enabled`` **and** have both ``listen`` and ``upstream`` are
started.
