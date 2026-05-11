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

<p align="center">
  <a href="https://ko-fi.com/coderred">
    <img src="https://ko-fi.com/img/githubbutton_sm.svg" alt="Support proroot on Ko-fi">
  </a>
</p>

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
└── libproroot-linker.so    Clean-room glibc-compatible dynamic linker
```

## Installation

Download all 4 `.so` files from [Releases](../../releases) and place them in `jniLibs/arm64-v8a/`.

Latest validated build: clean-room linker/runtime hardening for Android app-process and Termux workloads. It avoids exporting guest `LD_PRELOAD` into the Android host process, ships a clean-room linker with `PT_PHDR`, handles Android static-pie duplicate `argv[0]`, preserves patched-syscall errno, keeps rootfd-relative `.`/`..` and `/proc/1/root` behavior inside the guest root, removes the fixed `DT_NEEDED` dependency cap for large libraries such as GTK4, virtualizes procfs metadata, handles 16KB-page Android devices, preserves link2symlink anchor files as regular files, remains compatible with legacy proroot link2symlink metadata, masks guest MTE HWCAP exposure, avoids unsafe glibc malloc-state writes, and passes full smoke with Node.js, Python, npm, Playwright Chromium, XFCE/VNC, OpenClaw gateway runtime deps, Codex, esbuild, link/symlink, and static procfs coverage.

### Requirements

- Android 8.0+ (API 26)
- arm64-v8a
- Ubuntu arm64 rootfs with glibc

### Usage

```sh
libproroot.so -r <rootfs> -0 --link2symlink -w /root /bin/sh -c 'node server.js'
```

The launcher auto-discovers `libproroot-runtime.so`, `libproroot-linker.so`, and `libproroot-bridge.so` from its own directory (`/proc/self/exe` dirname). Drop all four `.so` files in the same folder and call the launcher directly — no `LD_*` or `PROROOT_*_PATH` exports required.

#### CLI options

| Option | Description |
|--------|-------------|
| `-r <rootfs>` | Guest root directory (required) |
| `-w <dir>` | Working directory inside the guest |
| `-b <host>` / `-b <host>:<guest>` | Bind-mount a host path into the guest |
| `-0` | Fake `uid=0` / `gid=0` (proot-compatible fakeroot) |
| `--link2symlink` | Emulate hardlinks via anchor + symlink groups (legacy proroot compat) |

Android app-process invocation:

```sh
PROROOT_TMP_DIR=/data/user/0/<package>/files \
libproroot.so \
  -r /data/user/0/<package>/files/rootfs \
  -0 \
  --link2symlink \
  -w /root \
  /bin/sh -c '<command>'
```

### Environment variables

| Variable | Description |
|----------|-------------|
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

Latest validated app-process smoke coverage:

- Samsung Galaxy Flip `SM-F721N`, Android 16 / SDK 36: `android-smoketest/build/smoke-results/20260430-124852-flip-legacy-compat`, all stages `RESULT_EXIT=0`
- Lenovo Tab `TB373FU`, Android 15 / SDK 35: `android-smoketest/build/smoke-results/20260430-124852-lenovo-legacy-compat`, all stages `RESULT_EXIT=0`

- app-private rootfs baseline bootstrap for `curl`, `git`, `python3`, `node`, and `npm`
- `node --version` -> `v22.22.2`
- `npm --version` -> `10.9.7`
- `python3 --version`
- `git --version` -> `git version 2.43.0`
- repeated Node child-process smoke (`NODE_CHILD_OK`)
- GUI package install smoke: `apt-get install -y xauth dbus-x11 tigervnc-standalone-server xfce4` (`GUI_INSTALL_OK`)
- VNC/XFCE desktop startup smoke: write `/root/.vnc/xstartup`, start `vncserver :124`, and verify `xfce4-session` + `xfwm4` are alive (`GUI_VNC_SMOKE_OK`)
- Playwright Chromium screenshot smoke: `npm install playwright`, `npx playwright install chromium`, navigate to `https://www.naver.com`, save a full-page screenshot (`PLAYWRIGHT_CHROMIUM_SCREENSHOT_OK`)
- symlink and hardlink filesystem smoke (`PY_SYMLINK_OK`, `PY_HARDLINK_OK`)
- link2symlink original-anchor preservation smoke (`verify-original-preservation`, all checks passed)
- `npm install openclaw`
- `openclaw --version` -> `OpenClaw 2026.4.26 (be8c246)`
- OpenClaw gateway bundled runtime deps staging and symlinked `plugin-runtime-deps` smoke
- `npm install @openai/codex`
- `codex --version` -> `codex-cli 0.125.0`
- `npm install esbuild`
- `esbuild --version` -> `0.28.0`
- `npm install playwright`
- `npx playwright install chromium`
- Playwright Chromium navigation and screenshot of `https://www.naver.com`
- static procfs smoke coverage (`PROC_SELF`, `PROC_PID`, `PROC_STATX_PRECISION`, `PROC_MOUNTS`, `PROC_THREAD_SELF`, `PROC_TASK_META`)

Latest validated Termux clean-room full-smoke coverage:

- Samsung Galaxy Flip `SM-F721N`, Android 16 / SDK 36: `build/termux-smoke-results/20260430-131607-flip-termux-legacy-compat`, `ALL_PASS`
- Lenovo Tab `TB373FU`, Android 15 / SDK 35: `build/termux-smoke-results/20260430-131607-lenovo-termux-legacy-compat`, `ALL_PASS`
- `RUN_MEDIUM=1 RUN_HEAVY=1 RUN_PLAYWRIGHT=1 RUN_GUI=1 RESET_ROOTFS=1 scripts/termux-full-smoke-cleanroom.sh` -> `ALL_PASS`
- basic smoke: `true`, `cat /etc/hostname`, `ls /`, `date`, `bash`, `python3 --version`, and `id`
- medium smoke: bootstrap, filesystem/tar, Node child process, Python `posix_spawn` file actions, and `tcsetpgrp`
- heavy npm smoke: OpenClaw, Codex, and esbuild installs
- Playwright Chromium screenshot smoke
- GUI smoke: XFCE package install and TigerVNC desktop startup
- link2symlink original-anchor preservation smoke (`verify-original-preservation-termux`, all checks passed)

Known non-blocking runtime noise during Android app-process smoke:

- optional backend misses such as `libnss_db.so.2` can appear when the guest rootfs does not ship that optional NSS module
- GUI package install may emit systemd / D-Bus helper noise while still ending in `RESULT_EXIT=0`

## Source code

The source is not public yet because I'm still stabilizing the implementation and don't want to publish something half-broken.

For now I'm sharing binaries for testing and feedback while I validate compatibility across real workloads.

## License

Proprietary. Free to use in your projects. Redistribution of modified binaries is not permitted.
