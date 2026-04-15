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

Latest public binary release: **v1.0.12** — child-startup optimization follow-up, app baseline bootstrap validation, and no recurrence of `pthread_create: Invalid argument` during rerun smoke validation.

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
| `PROROOT_TMP_DIR` | Writable directory for runtime config files (use app `filesDir` in Android app processes) |

## Tested with

- **Node.js 22.22.2** + npm **10.9.7**
- **Python 3.12.3**
- **Git 2.43.0**
- **Chromium / Playwright app-process smoke**
- **XFCE 4 + TigerVNC** app-process GUI smoke
- **curl**, **OpenSSL 3.0**

Latest validated app-process smoke coverage (`v1.0.12`):

- app-private rootfs baseline bootstrap for `curl`, `git`, `python3`, `node`, and `npm`
- `node --version` -> `v22.22.2`
- `npm --version` -> `10.9.7`
- `python3 --version` -> `Python 3.12.3`
- `git --version` -> `git version 2.43.0`
- repeated Node child-process smoke (`NODE_CHILD_OK`)
- GUI package install smoke: `apt-get install -y xauth dbus-x11 tigervnc-standalone-server xfce4` (`GUI_INSTALL_OK`)
- VNC/XFCE desktop startup smoke: write `/root/.vnc/xstartup`, start `vncserver :124`, and verify `xfce4-session` + `xfwm4` are alive (`GUI_VNC_SMOKE_OK`)
- Playwright Chromium screenshot smoke: `npm install playwright`, `npx playwright install chromium`, navigate to `https://www.naver.com`, save a full-page screenshot (`PLAYWRIGHT_CHROMIUM_SCREENSHOT_OK`)
- `npm install openclaw`
- `openclaw --version` -> `OpenClaw 2026.4.14 (323493f)`
- `npm install @openai/codex`
- `codex --version` -> `codex-cli 0.120.0`
- `npm install esbuild`
- `esbuild --version` -> `0.28.0`
- `npm install playwright`
- `npx playwright install chromium`
- Playwright Chromium navigation and screenshot of `https://www.naver.com`
- static procfs smoke coverage (`PROC_SELF`, `PROC_PID`, `PROC_STATX_PRECISION`, `PROC_MOUNTS`, `PROC_THREAD_SELF`, `PROC_TASK_META`)

Known non-blocking runtime noise during Android app-process smoke:

- `exec linker failed ... Permission denied` can appear before fallback succeeds
- GUI package install may emit systemd / D-Bus helper noise while still ending in `RESULT_EXIT=0`

## Source code

The source is not public yet because I'm still stabilizing the implementation and don't want to publish something half-broken.

For now I'm sharing binaries for testing and feedback while I validate compatibility across real workloads.

## License

Proprietary. Free to use in your projects. Redistribution of modified binaries is not permitted.

`libldlinux.so` is derived from GNU C Library and is licensed under [LGPL-2.1](https://www.gnu.org/licenses/old-licenses/lgpl-2.1.html).
