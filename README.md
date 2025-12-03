# BusyPy — README

**BusyPy** is a minimal, Python-first root filesystem and init framework for *cross-architecture* deployments.
It provides a shared rootfs layout (NFS-exportable) that lets heterogeneous targets (e.g. x86_64, aarch64) run the *same* userland scripts while resolving native interpreters and C libraries per-target at boot. BusyPy’s core programs are Python scripts / small Python applications, so userland programs normally do **not** require per-architecture compilation.


## Key ideas 

* One canonical, server-side rootfs tree holds per-architecture runtimes under architecture-scoped subtrees (e.g. `/etc/python/aarch64`, `/etc/libc/aarch64`).
* On boot, the target’s Python-based `init` composes a local runtime namespace using `tmpfs` + `MS_BIND` mounts so `/usr` and `/lib` resolve to that target’s native interpreter and shared libs.
* User scripts on `/bin` are identical bytes on the server (single source), and their shebang `#!/usr/bin/python3` resolves to the native interpreter on each client.
* Interpreters are ELF-patched during staging (PT_INTERP & RUNPATH) so they work after the client assembles `/lib`.

## Features

* Single shared rootfs for multiple ISAs (cross-architecture execution of same scripts).
* Minimal storage overhead on the server — per-arch artifacts are staged under `/etc/*` and exposed at runtime.
* Developers write Python programs; no mandatory cross-compile cycle for pure-Python apps.
* Deterministic, syscall-level early userspace (`init`) that does not depend on higher level tools during early boot.
* Reproducible staging (patchelf + readelf outputs and checksums preserved as artifacts).


## Project layout (example)

Only examples shown — content and paths in your repository may vary.

```
.
├── bin/                  # userland scripts (identical across targets)
│   ├── ls
│   ├── ps
│   └── shell
├── sbin/                 # architecture-specific init scripts (shebang points to per-arch python)
│   ├── aarch64-virt-init
│   └── x86_64-pc-init
├── etc/
│   ├── python/           # staged per-arch Python runtimes: etc/python/aarch64/...
│   └── clib/             # staged per-arch C libs: etc/clib/aarch64/...
├── lib/                  # usually empty on server; filled at client runtime via tmpfs+bind
├── lib64 -> lib
├── site-package/         # shared pure-Python packages (optional)
├── README.md
└── (dev, proc, sys, usr) # other standard FHS entries
```



## Getting started — quick outline

1. **Stage & patch interpreter on build host**

   * Build cross-compiled Python for each target (e.g. using Buildroot).
   * Place runtime into `/etc/python/<arch>/usr/...` inside your export tree.
   * Patch ELF fields so `PT_INTERP=/lib/ld-linux-<arch>.so.*` and `RUNPATH=/lib` (example: `patchelf --set-interpreter /lib/ld-linux-aarch64.so.1 --set-rpath /lib /etc/python/aarch64/usr/bin/python3`).
   * Save `readelf` outputs and checksums for reproducibility.

2. **Prepare export tree on NFS server**

   * Extract finalized `rootfs.tar.gz` (or image) in the export directory.
   * Create `etc/python/<arch>` and `etc/clib/<arch>` subtrees; ensure permissions and symlinks (`lib64 -> lib`) as needed.

3. **Configure NFS export**

   * Add export (example): `/srv/rootfs 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)` — tune options per your security/performance tradeoffs.

4. **Boot client(s)**

   * Kernel command line points to NFS root (e.g. `root=/dev/nfs nfsroot=192.168.1.8:/srv/rootfs,vers=3,proto=tcp init=/sbin/<arch>-init`).
   * Client `init` (Python) runs early and performs `tmpfs` mounts + `MS_BIND` of `/etc/python/<arch>/usr` → `/usr` and `/etc/clib/<arch>` → `/lib`, mounts `/proc` and `/sys`, validates interpreter, then forks/execs `/bin/shell`. Parent (PID 1) remains to reap children.



## How to add or update programs

* **Pure Python**: add your `.py` modules or scripts into `bin/` or `site-package/` on the export. No recompilation required; after atomic deployment, clients see changes immediately (subject to cache and overlay semantics).
* **Native extensions**: compile per-architecture and stage compiled `.so` files under `/etc/python/<arch>/usr/lib/...` and corresponding C libraries under `/etc/clib/<arch>/`.
* **Atomic deployment**: publish to a versioned directory then switch a symlink like `current -> vN` or use `rsync --delay-updates` to avoid partially visible updates.


## Validation & smoke tests (recommended)

Record stdout/stderr of these checks and include them as artifacts:

```bash
# On build host (staging step)
<cross>-readelf -l /etc/python/aarch64/usr/bin/python3  > readelf_l.before.txt
<cross>-readelf -d /etc/python/aarch64/usr/bin/python3  > readelf_d.before.txt

# After patchelf
<cross>-readelf -l /etc/python/aarch64/usr/bin/python3  > readelf_l.after.txt
<cross>-readelf -d /etc/python/aarch64/usr/bin/python3  > readelf_d.after.txt

sha256sum /etc/python/aarch64/usr/bin/python3 > python3.sha256
```

On the client (after init assembled `/usr` and `/lib`):

```bash
readelf -l /usr/bin/python3
readelf -d /usr/bin/python3
ldd /usr/bin/python3
/usr/bin/python3 -c 'import sys,platform; print(sys.executable, platform.machine(), sys.version)'
cat /proc/self/mounts
```



## Runtime notes & rationale (brief)

* **tmpfs + bind order**: mount `tmpfs` first (gives writable, per-client upper), then `MS_BIND` the server subtree to provide authoritative content while preserving writable local overlay. This isolates clients and prevents server pollution.
* **Why per-arch init?** Init is PID 1 and must be executable immediately after `execve` by the kernel—its interpreter must be available. Per-arch init scripts carry shebangs pointing at their native `/etc/python/<arch>/usr/bin/python3` so the kernel launches a compatible interpreter for PID 1.
* **ELF re-pathing**: required because PT_INTERP/RUNPATH metadata must match the client assembled view (`/lib`) after init has mounted per-arch libraries. Patching is done at staging so clients work deterministically.



## Troubleshooting (common)

* `Kernel panic - Attempted to kill init!`: ensure PID 1 (init) does not exit; parent must remain.
* `No such file or directory` when execing interpreter: check that `/lib/ld-linux-*.so.*` exists in the client `/lib` after init composition.
* Missing modules like `_posixsubprocess` or `_ctypes`: ensure staged Python stdlib and `lib-dynload` were copied for the correct Python version and arch.
* `mount tmpfs` errors: confirm kernel supports tmpfs and that init uses raw libc/syscalls (no external mount binary required).


## Contributing

* Add scripts or pure-Python utilities under `bin/` or `site-package/`.
* For native modules, provide build artifacts in a per-arch folder under `etc/` and document the toolchain used.
* Include tests, `readelf` outputs, and checksums whenever you add or modify an interpreter or C libraries.
* Submit PRs with clear description and a short test plan.



## License & Support

* **License:** MIT — see `LICENSE`.
* **Support / Contact:** `xmersadkarimi@gmail.com`

---
