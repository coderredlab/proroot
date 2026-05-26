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

> **📢 Notice**
>
> Similar LD_PRELOAD-based tools have started appearing recently — some within days of discovering this project. That was a wake-up call.
>
> I'm now building **proroom**, a standalone Android app with proroot at its core. One app, everything integrated — no Termux-X11, no VNC, no extra setup.
>
> Binary releases here continue for existing users. Development focus has shifted to proroom, so expect slower updates in the meantime.

<p align="center">
  <a href="https://ko-fi.com/coderred">
    <img src="https://ko-fi.com/img/githubbutton_sm.svg" alt="Support proroot on Ko-fi">
  </a>
</p>

---

## Installation

Download all 5 `.so` files from [Releases](../../releases) and place them in `jniLibs/arm64-v8a/`.

### Requirements

- Android 8.0+ (API 26)
- arm64-v8a
- Ubuntu arm64 rootfs with glibc

### Usage

```sh
libproroot.so -r <rootfs> -0 --link2symlink -w /root /bin/sh -c 'node server.js'
```

The launcher auto-discovers `libproroot-runtime.so`, `libproroot-linker.so`, `libproroot-bridge.so`, and `libproroot-stub-loader.so` from its own directory (`/proc/self/exe` dirname). Drop all five `.so` files in the same folder and call the launcher directly — no `LD_*` or `PROROOT_*_PATH` exports required.

#### CLI options

| Option | Description |
|--------|-------------|
| `-r <rootfs>` | Guest root directory (required) |
| `-w <dir>` | Working directory inside the guest |
| `-b <host>` / `-b <host>:<guest>` | Bind-mount a host path into the guest |
| `-0` | Fake `uid=0` / `gid=0` (proot-compatible fakeroot) |
| `--link2symlink` | Emulate hardlinks via anchor + symlink groups (legacy proroot compat) |
| `--static-loader` / `--no-static-loader` | Force-enable or disable static-loader exec routing. Default is adaptive: enabled when `libproroot-stub-loader.so` is available. |

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
| `PROROOT_STUB_LOADER` | Optional absolute path override for `libproroot-stub-loader.so` |
| `PROROOT_LOG_APPEND` | Optional diagnostic log file path used by launcher/runtime/stub traces |

## Tested with

- **Node.js 22.22.2** + npm **10.9.7**
- **Python 3.12.3**
- **Git 2.43.0**
- **Chromium / Playwright**
- **XFCE 4 + TigerVNC**
- **curl**, **OpenSSL 3.0**

Validated on Samsung Galaxy Flip (Android 16) and Lenovo Tab (Android 15).

## Source code

The source is not public yet because I'm still stabilizing the implementation and don't want to publish something half-broken.

For now I'm sharing binaries for testing and feedback while I validate compatibility across real workloads.

## License

Proprietary. Free to use in your projects. Redistribution of modified binaries is not permitted.
