# Changelog

All notable changes to klug are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **`klug --help` now warns, on AGNOS builds only, that dumping the log to the console consumes
  the log — redirect instead (`run /bin/klug > /klug.txt`).** fd 1 is a `VFS_DEVICE`, so klug's own
  `write(1, …)` takes `vfs_write`'s device arm → `dev_write` → `serial_dev_write` (agnos
  `core/devs.cyr`) → `kprint` → `klug_append`: every console-bound ring-3 byte is appended to the
  very ring `klug`#36 just read. The dump is up to `KLUG_RING_BYTES` and the kernel ring is exactly
  `KLUG_RING_BYTES`, so a console dump re-appends the whole log to itself — and while the ring is
  still short of full (a ~20 KB iron boot log) that *doubles* it, so the second dump wraps and takes
  the boot head with it. That is the one thing you were reading the log to find. A pipe is safe for
  the same reason the redirect is: a pipe fd takes `vfs_write`'s `VFS_PIPE` arm and never reaches
  `kprint`, so only grep's matched lines are console-bound — `klug | grep <pattern>`, which the help
  text has always recommended, was never the hazard; the bare dump is. Documented in `README.md`
  under a new "On AGNOS: redirect, don't dump" section.

  ⚠ The note is gated `#ifdef CYRIUS_TARGET_AGNOS`, so the Linux dev-host build is **byte-identical**
  to 0.1.4's — the hazard is a property of the agnos console path and has no meaning against
  `/dev/kmsg`. Only `build/klug_agnos` changes.

  ⛔ This is a *warning*, not a fix, and it is deliberately not a substitute for one. The kernel
  could suppress the tap while `klug`#36's own output is being written; that is tracked agnos-side
  (`docs/development/state.md` OPEN, `roadmap.md`, and the `run /bin/klug > /f.txt` note already
  standing at `kernel/core/syscall.cyr:442-444`). Such a fix would settle *this tool*, not the
  property — any large console dump still ages the ring — and klug ships as a committed binary that
  `agnos/scripts/burn/stage-tools.sh` stages onto whatever kernel is on the box, including older
  ones. The comment above the block in `src/klug.cyr` says so, so the note is not dropped on the
  assumption that the kernel fix covers it.

## [0.1.4] — 2026-08-25 (cyrius 6.5.35, lib resync, CI unbreak)

### Fixed

- **CI and Release would have hard-failed the moment the pin moved — the bump and its fix ship
  together.** Both workflows hand-rolled the toolchain install, flattening the release tarball into
  `$HOME/.cyrius/{bin,lib}` with no `versions/` directory. Since cyrius **6.5.24** every verb resolves
  the `cyrius.cyml` pin against `$CYRIUS_HOME/versions/<pin>/lib` and hard-errors when that path is
  absent — `error: cyrius.cyml pins version 6.5.35 but it is not installed at …/versions/6.5.35/lib`,
  raised *before* anything compiles, so `build`, `build --agnos`, `test` and `lib sync` all die.
  Reproduced against the flat layout at the new pin; bisected as fine at 6.5.23, fatal from 6.5.24 on.
  It stayed green only because the old pin (6.2.24) predates the check. Both workflows now use the
  upstream `scripts/install.sh`, which lays out `versions/<ver>/{bin,lib}` + `current` and verifies the
  release SHA-256 — the recipe patra and abaco already use.
- **`warning: undefined function 'vec_get'` on every build**, host and agnos. `lib/fmt.cyr`'s variadic
  path (`fmt_printf`/`fmt_sprintf`/`fmt_float`) has called `vec_get` since before the old pin, but its
  header declares only `string.cyr` and `[deps].stdlib` never named `vec`. Harmless in practice — all
  three callers are dead code in klug — but a release that prints "undefined function" is a trap for
  whoever next touches `fmt`. `vec` is now declared; the warning is gone.
- **Nomenclature: the syscall is `klug`, not `klog`.** The kernel defines `klug_info`/`klug_warn`/
  `klug_err` (agnos `core/klug.cyr:60-62`) and the syscall table spells #36 `klug`; `klog_*` appears
  nowhere in the agnos kernel. Corrected across `src/klug.cyr`, `src/main.cyr`, `tests/klug.tcyr`,
  `README.md` and `release.yml`, including the user-visible error string.
- **The prose still said "16 KB" in five places** — 0.1.3 moved the value to 65536 but left the
  narration behind, which is the exact confusion 0.1.3 existed to end. Fixed in `src/klug.cyr` (module
  header + the `KLUG_RING_BYTES` comment three lines above `= 65536`), `src/main.cyr:12` (which
  contradicted its own next line), and `README.md` (×2). The dated 0.1.0–0.1.3 entries below are
  history and are left alone.
- `README.md` pointed at `agnos/scripts/stage-tools.sh`; the script is at
  `agnos/scripts/burn/stage-tools.sh`.
- `src/main.cyr`'s bare-call rationale cited a cyrius bug as live. 6.1.32 moved the agnos init-rsp
  capture to an entry-landing r15 park that precedes all gvar-init code, so the 0.1.1 workaround has
  been unnecessary since before the *old* pin. The code is unchanged — it is still correct and it is
  what the agnos exec path expects — but the comment no longer claims a fix that already landed.
- The CI test-step comment was stale in all three of its claims: bare `cyrius test` works, `cyrius
  tests` exists (and walks `tests/` recursively since 6.4.72), and neither is 6.0.x-gated.
- `src/klug.cyr` called `/dev/kmsg` "world-readable on modern Linux". The node is mode 0644, but
  `kernel.dmesg_restrict=1` — the systemd-distro default — makes `open()` return EPERM for non-root
  regardless. The comment now says so and the failure message names the two ways out (root, or
  `dmesg_restrict=0`) instead of implying the host is broken.

### Changed

- **cyrius toolchain pin 6.2.24 → 6.5.35.**
- **Vendored `lib/` resynced and re-shaped: a stale 97-file mirror → the 23-file `[deps].stdlib`
  closure**, produced by `cyrius lib sync` rather than by hand. The removed 74 files were never
  reachable from `src/` or `tests/`. Ten of them were stale by construction: `json`/`toml`/`cyml`/
  `csv`/`base64`/`bigint`/`u128` were carved into `lib/bayan.cyr` at cyrius **6.1.25** and
  `matrix`/`linalg` into `lib/ganita.cyr` at **6.1.26**, so klug was vendoring modules the toolchain
  had not shipped for four minor versions. The eleventh, `agnosys.cyr`, was a `cyrius distlib` bundle
  of agnosys **1.2.6** — never stdlib, never included, 330 KB of fossil.
- `[deps].stdlib` gains `vec` (see above). It is a pre-existing gap, not a 6.5.35 requirement.
- **The agnos read is now `sys_klug(buf, len)` instead of a raw `syscall(36, …)`.** cyrius 6.3.14 added
  `SYS_KLUG = 36` and the `sys_klug` wrapper to the agnos syscall peer expressly to retire raw numeric
  syscalls and the mis-dispatch class they invite; the constant is a literal alias for 36, so this is a
  rename, not a behaviour change (same binary size, and the wrapper is absent from the Linux peer,
  which keeps the `#ifdef CYRIUS_TARGET_AGNOS` guard honest).
- **CI now builds the `--agnos` target too.** For an agnos-first tool, gating only the host build meant
  an `--agnos` break could not surface until tag time, inside Release.

### Added

- `LICENSE` — GPL-3.0-only. The license was asserted in both `README.md` and `cyrius.cyml` but no
  license file was ever committed.

### Note for the agnos side

`build/klug_agnos` is regenerated and committed with this release, and that matters:
`agnos/scripts/burn/stage-tools.sh` auto-builds only its own in-tree rows, and deliberately **not**
sibling repos (each sibling pins its own cyrius). Without `--build` it stages whatever binary is
committed here — so the pin bump and lib resync only reach the agnos rootfs because the artifact was
rebuilt. For the same reason `build/` stays tracked; a `.gitignore` over it would silently break
staging.

## [0.1.3] — 2026-07-12 (64 KB ring)

### Changed

- **Ring read buffer 16 KB → 64 KB**, in lockstep with the agnos kernel ring (`core/klug.cyr`):
  `KLUG_RING_BYTES` 16384 → 65536 and `main.cyr` `klug_buf[2048]` → `[8192]`. The tool's buffer and the
  kernel ring MUST match — `klog#36` returns at most the tool's requested size, so a buffer smaller than
  the ring pulls only the tail.

### Fixed

- **The tool could only return the last 16 KB of the kernel log, losing the earliest boot lines.** The
  iron boot log outgrew 16 KB (the agnos GPU arc + full device enumeration + SMP push a clean boot to
  ~16–20 KB), so the kernel ring wrapped dmesg-style and the reader faithfully returned the wrapped
  tail — which read as "the log starts partway through / stops at the kybernet handoff." 64 KB holds the
  whole boot log ~3× over. The `tests/klug.tcyr` ring-size contract now pins 64 KB (it caught the
  kernel/tool mismatch, as designed). Requires the paired agnos kernel change (klug ring → 64 KB).

## [0.1.2] — 2026-06-19 (cyrius toolchain bump)

### Changed

- cyrius toolchain pin 6.1.14 → 6.2.24.

## [0.1.1] — 2026-06-08 (agnos argv fix)

### Changed

- cyrius toolchain pin 6.0.56 → 6.1.14.

### Fixed

- **agnos: command-line args (e.g. severity-lens flags) weren't seen.** Call `main` from a bare top-level statement (`_agnos_entry();`) instead of `var r = main();`. The latter runs `main` as a module-global initializer, *before* cyrius's init-stack capture, so `argc()`/`argv()` read 0/null. cyrius issue: agnos argv init-rsp capture.

## [0.1.0] — 2026-06-06 (the userland half of klug — a sovereign dmesg)

### Added

- **klug** — the userland reader for the AGNOS kernel log, completing the *Kernel Logs Unified Grep* subsystem: the kernel half (agnos `core/klug.cyr`, shipped 1.42.11–12) unifies all kernel output into one 16 KB ring buffer; this binary dumps that ring to stdout. Filtering past the severity lens stays the shell's job (`klug | grep panic`) — the kernel UNIFIES, GREP stays userland.
- **AGNOS-native** via the `klog`#36 read syscall (`cyrius build --agnos`): one read drains the full 16 KB ring (oldest→newest), written straight to stdout.
- **Dev-host dogfooding** on Linux: reads `/dev/kmsg` (O_RDONLY|O_NONBLOCK) so the tool runs + is testable off-target.
- **Severity lens**: `-w`/`--warn` (warnings + errors) and `-e`/`--err` (errors only), keyed off the kernel's `[I]`/`[W]`/`[E]` line prefixes (`klog_info`/`warn`/`err`). Unprefixed lines (boot output, raw `kprintln`) always show. `-h`/`--help` for usage.
- Built `--agnos` as a static ELF64 and **banked onto the agnos-fs `/bin`** via `agnos/scripts/stage-tools.sh`, runnable through the in-kernel recovery shell's `run /bin/klug` exec-from-disk path (validated on iron as item H5).
- Test harness (`tests/klug.tcyr`) pins the `klug_level_of` lens against the kernel's prefix contract + the 16 KB ring-size invariant; the I/O dump path is integration-tested via the agnos exec path.
