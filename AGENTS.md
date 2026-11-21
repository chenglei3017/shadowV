# AGENTS.md

Guidance for agents working in this repository.

## Project Overview

ShadowVPN is a small, fast VPN daemon written in C, built on libsodium (salsa20/poly1305) and a TUN device. It targets low-end devices (OpenWRT routers) and also runs on Linux, FreeBSD, macOS, Windows (MinGW), and Android. Traffic is sent as encrypted UDP datagrams between one server and one client (with optional multi-user NAT on the server via `user_token`).

Version 0.2.0, GPLv3 (see `COPYING`). This is beta software.

## Build & Test

Linux/Unix:

```sh
git submodule init && git submodule update   # fetches libsodium into ./libsodium
./autogen.sh                                 # autoreconf / libtoolize
./configure --enable-static --sysconfdir=/etc
make
sudo make install
```

- `libsodium/` is a git submodule; nothing builds without it. Do not edit it.
- Build system is autotools: `configure.ac`, top-level `Makefile.am`, `src/Makefile.am`, `samples/Makefile.am`.
- The library `src/libshadowvpn.la` holds all logic; `src/main.c` is a thin CLI wrapper. `main.c` is the only file linked into the `shadowvpn` binary.
- Platform selected at configure time via `TARGET_LINUX`, `TARGET_DARWIN`, `TARGET_FREEBSD`, `TARGET_WIN32` macros (generated in `config.h`). Code is guarded by `#ifdef TARGET_*` throughout — always keep all platform branches compiling.
- CI: `.travis.yml` runs `./configure && make` and clang static analysis (`scan-build`) with both gcc and clang.

There is no unit-test suite. `tests/concurrency.sh` is a manual smoke test requiring two real machines (client + SSH-accessible server); it pings and downloads a file. Platform build helpers: `dist-build/android-*.sh`, `packaging/release.sh`, `openwrt/Makefile`.

## Repository Layout

- `src/` — daemon source (see table below)
- `samples/` — example configs (`client.conf`, `server.conf`) and up/down scripts for Linux, `darwin/`, `freebsd/`, `windows/`
- `debian/`, `openwrt/`, `dist-build/`, `packaging/` — packaging and cross-build scripts
- `tools/` — chnroutes generator scripts and route tables (not built)
- `tests/concurrency.sh` — manual end-to-end test

Key source files:

| File | Responsibility |
| --- | --- |
| `main.c` | Parse args, daemonize start/stop/restart, init crypto, signal handlers, run VPN |
| `args.c/.h` | CLI (`-c -s -v -h`) and INI-style config file parsing into `shadowvpn_args_t`; defaults |
| `crypto.c/.h` | Password→key derivation (`crypto_generichash`), encrypt/decrypt, packet buffer layout constants |
| `crypto_secretbox_salsa208poly1305.c` | Custom libsodium-based secretbox variant (8-byte nonce, 16-byte MAC) |
| `vpn.c/.h` | TUN allocation per OS, UDP socket allocation, main `select()` loop, packet encrypt/decrypt path |
| `nat.c/.h` | Server-side multi-user NAT keyed by `user_token` (uthash tables token→client, IP→client) |
| `daemon.c/.h` | fork/setsid daemonization, PID file, stop/restart via PID |
| `shell.c/.h` | Runs `up`/`down` scripts with `sh` (or `cmd /c` on Windows) |
| `log.c/.h` | `logf`/`errf`/`err`/`debugf` macros; Android log backend under `HAVE_ANDROID_LOG` |
| `nat.c`, `uthash.h` | Hash tables (`uthash.h` is vendored, do not modify) |
| `win32.c/.h` | Windows TUN via a delegated helper process (`TUN_DELEGATE_*`) |
| `portable_endian.h` | Byte-order helpers (`htobe64`, etc.) |

## Runtime Data Flow

Client: TUN read → optionally prepend 8-byte `user_token` → `crypto_encrypt` → `sendto` UDP.
Server: `recvfrom` → `crypto_decrypt` → `nat_fix_upstream` (rewrite src IP, remember client addr) → TUN write.
Server reverse: TUN read → `nat_fix_downstream` (restore dst) → encrypt → send to mapped client.
NAT is only active when `user_token` is set; the server assigns `netip + 1, +2, ...` to each token at startup.

Packet buffer layout (see `crypto.h`):

```
[SALSA20_RESERVED 8][NONCE 8][MAC 16][OPTIONAL USERTOKEN 8][PAYLOAD MTU]
```

- `SHADOWVPN_ZERO_BYTES` = 32 (reserved + nonce + MAC), `SHADOWVPN_OVERHEAD_LEN` = 24, `SHADOWVPN_PACKET_OFFSET` = 8, `SHADOWVPN_USERTOKEN_LEN` = 8.
- TUN reads/writes at offset `SHADOWVPN_ZERO_BYTES + usertoken_len`; UDP at `SHADOWVPN_PACKET_OFFSET`.

## Configuration

- INI-like: `key=value`, no spaces around `=`, `#` comments. Parsed in `args.c:process_key_value`.
- Required: `mode`, `server`, `port`, `password`; Windows also `tunip`.
- All keys except `password` are exported as environment variables so `up`/`down` scripts can use them. Unknown keys are passed through with a warning.
- Defaults: `intf=tun0` (`utun0` on Darwin), `mtu=1440`, `pidfile=/var/run/shadowvpn.pid`, `logfile=/var/log/shadowvpn.log`, `concurrency=1`.
- `concurrency` is currently disabled in the data path (only socket 0 is used); warn users to set `concurrency=1` on both sides.
- MTU must be between `68 + SHADOWVPN_OVERHEAD_LEN` and `MAX_MTU` (9000).
- `user_token` is comma-separated hex (8 bytes each); parsed in `parse_user_tokens`.

## Coding Conventions & Gotchas

- C (C99-style declarations allowed mid-block); 2-space indentation; K&R braces; GPL header comment at the top of each file. Follow the surrounding style.
- Be careful with the packet buffer offsets above — encryption/decryption and NAT rely on them exactly.
- `vpn_ctx_init`/`vpn_run` return `-1` on error and intentionally do not clean up partial allocations; normal exit from `vpn_run` also returns `-1` (after `shell_down` and socket close), so `main`'s return is not a success indicator.
- Stop is signaled through a control pipe (`vpn_stop` writes one byte); on Windows it is a UDP control socket to the TUN delegate. Do not replace this with a shared flag only.
- Keep non-Windows and Windows code paths in sync; Windows has known limitations (no daemonization, requires `tunip`, separate helper).
- Cross-compilation matters (OpenWRT/router MIPS/ARM, Android). Avoid adding dependencies or non-portable APIs.
- Never modify vendored `src/uthash.h` or `libsodium/`.
- Don't commit generated autotools output (`configure`, `Makefile.in`, etc. are gitignored).
