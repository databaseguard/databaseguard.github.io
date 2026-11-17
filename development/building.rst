=======================
Building & Contributing
=======================

Repository layout
=================

.. code-block:: text

   dbguard/
   ├── cmd/                   # dbguardd, dbguard-cli
   ├── bpf/                   # eBPF C sources + headers
   │   ├── src/*.bpf.c
   │   └── include/dbguard/   # shared structs, helpers, min_compat.h
   ├── internal/              # the Go packages (ebpf, proxy, policy, pipeline, ...)
   ├── configs/               # dbguard.yaml + policies/
   ├── deploy/                # Dockerfile, k8s manifests, CiliumNetworkPolicy
   ├── docs/                  # markdown notes (policies.md, ...)
   └── Makefile

Build targets
=============

.. code-block:: bash

   make bpf         # compile bpf/src/*.bpf.c -> build/bpf/*.o  (needs clang/llvm)
   make go-build    # compile cmd/* -> build/bin/{dbguardd,dbguard-cli}
   make test        # go test ./...
   make dbgsym      # install matching DB debug symbols (for stripped binaries)
   make install     # install binaries + configs on the host

Testing
=======

.. code-block:: bash

   go test ./...                 # unit tests across the Go packages
   go vet ./...
   gofmt -l .                    # should print nothing

Notable test coverage includes the policy engine (matching, first-deny-wins, reload
ordering), the proxy wire handling (including the ``SSLRequest`` decline), the Cilium
endpoint parsing, and the MySQL capability/attribute handling.

Writing an eBPF program
=======================

The BPF sources share a small compatibility layer (``bpf/include/dbguard``) rather than a
full ``vmlinux.h``. ``struct sock`` fields are read through a ``DBG_CORE_READ`` macro.
When adding a probe:

1. Add the C in ``bpf/src/<name>.bpf.c`` with the right ``SEC("…")`` and a function whose
   name the loader recognises (see ``internal/ebpf/loader.go``'s dispatch).
2. Add or extend a dispatch case in ``loader.go`` to attach it.
3. ``make bpf`` and confirm the program symbol is present
   (``readelf -s build/bpf/<name>.bpf.o``).

.. note::

   Socket-field reads can differ across kernels. For example ``skc_family`` can read
   incorrectly on some kernels while the adjacent ``skc_num`` (local port) reads fine - the
   client-IP correlation deliberately gates on the **local port** rather than the socket
   family for this reason.

Module path
===========

The Go module path is ``github.com/databaseguard/dbguard`` and all imports use it. It is an
identifier and need not match where you host the repository, though keeping them aligned is
tidy. Everything builds locally regardless of the hosting URL.

Pull requests
=============

- Keep ``go vet`` and ``gofmt`` clean; add tests for engine/proxy/identity behaviour.
- Validate any policy change with ``dbguard-cli validate-policy``.
- Document new config fields in :doc:`/reference/config-reference` and new rule fields in
  :doc:`/reference/policy-schema`.
