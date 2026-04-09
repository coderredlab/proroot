```
                                     __
    ____  _________  _________  ____/ /_
   / __ \/ ___/ __ \/ ___/ __ \/ __ / __/
  / /_/ / /  / /_/ / /  / /_/ / /_/ / /_
 / .___/_/   \____/_/   \____/\__,_/\__/
/_/
```

**Rootless Linux runtime for Android.**

Run full glibc Linux userspace — Node.js, Python, Git, Chromium — inside Android apps. No root required.

A drop-in replacement for [proot](https://proot-me.github.io/) with **zero ptrace overhead**.

---

## Why proroot?

proot intercepts every syscall via `ptrace` — 2 context switches each time. On a phone running a Node.js server with Chromium, that's millions of wasted cycles.

proroot takes a different approach:

```
proot:    App ──ptrace──▶ Kernel ──ptrace──▶ Handler ──ptrace──▶ Kernel ──▶ Done
                 ↑ 2 context switches per syscall

proroot:  App ──LD_PRELOAD──▶ translate() ──SVC──▶ Kernel ──▶ Done
                 ↑ 0 context switches, in-process path translation
```

## How it works

```
┌──────────────────────────────────────────────────────┐
│  Guest Process (Node.js / Chromium / Python / Git)   │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  libproroot-runtime.so  (LD_PRELOAD)           │  │
│  │                                                │  │
│  │  ┌─────────────┐  ┌──────────────────────────┐ │  │
│  │  │ PLT Wrapper  │  │ Binary Patching          │ │  │
│  │  │              │  │                          │ │  │
│  │  │ openat()     │  │ svc #0 → bl trampoline   │ │  │
│  │  │ stat()       │  │ (catches glibc-internal   │ │  │
│  │  │ execve()     │  │  raw syscalls too)        │ │  │
│  │  │ dlopen()     │  │                          │ │  │
│  │  │ connect()    │  └──────────────────────────┘ │  │
│  │  └──────┬──────┘                                │  │
│  │         ▼                                       │  │
│  │  translate_path(guest → host)                   │  │
│  │         ▼                                       │  │
│  │  raw_syscall6(SVC #0)  ─────────────────────▶ Kernel
│  │                                                 │  │
│  │  ┌──────────────────────────────────────────┐   │  │
│  │  │ Signal Handlers                          │   │  │
│  │  │  SIGSYS  → seccomp accept→accept4        │   │  │
│  │  │  SIGTRAP → Chrome BRK → NOP patch        │   │  │
│  │  │  SIGILL  → NOP fallback → LR return      │   │  │
│  │  └──────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

## Components

```
jniLibs/arm64-v8a/
├── libproroot.so           Launcher (NDK/bionic)
├── libproroot-runtime.so   LD_PRELOAD runtime (glibc)
├── libproroot-bridge.so    Child exec trampoline (NDK static)
└── libldlinux.so           glibc dynamic linker (LGPL-2.1)
```

## Installation

Download all 4 `.so` files from [Releases](../../releases) and place them in `jniLibs/arm64-v8a/`.

### Requirements

- Android 8.0+ (API 26)
- arm64-v8a
- Ubuntu arm64 rootfs with glibc

### Usage

```sh
libproroot.so -r <rootfs> -w /root --link2symlink /usr/local/bin/node server.js
```

### Environment variables

| Variable | Description |
|----------|-------------|
| `PROROOT_LIB_PATH` | Path to runtime .so (auto-detected) |
| `PROROOT_LINKER_PATH` | Path to glibc ld.so (auto-detected) |
| `PROROOT_TRAMPOLINE_PATH` | Path to bridge binary (auto-detected) |
| `PROROOT_VERBOSE=1` | Debug logging |
| `PROROOT_GUEST_EXE` | Guest path for /proc/self/exe emulation |

## Tested with

- **Node.js 24** + npm
- **Python 3.12**
- **Git 2.43**
- **Chromium headless_shell 131** (Playwright)
- **curl**, **wget**, **OpenSSL 3.0**

## Source code

The source is not public yet because I'm still stabilizing the implementation and don't want to publish something half-broken.

For now I'm sharing binaries for testing and feedback while I validate compatibility across real workloads.

## License

Proprietary. Free to use in your projects. Redistribution of modified binaries is not permitted.

`libldlinux.so` is derived from GNU C Library and is licensed under [LGPL-2.1](https://www.gnu.org/licenses/old-licenses/lgpl-2.1.html).
