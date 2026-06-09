# Changelog

All notable changes to klug are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
