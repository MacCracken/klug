# Changelog

All notable changes to klug are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.7] — 2026-09-07 (cyrius 6.6.0; the vendored stdlib stops rotting)

Toolchain pin **6.5.41 → 6.6.0** and a full re-vendor of `lib/`. No `src/` changes: every line of
klug's own code is untouched, and the entire diff is the manifest plus the vendored stdlib snapshot.
Tests **37/37**. Host and `--agnos` targets build clean.

### Fixed

- ⛔ **`[deps].stdlib` was UNDER-DECLARED, which silently froze part of the vendored stdlib.**
  It named **8** modules — `string`, `fmt`, `vec`, `alloc`, `io`, `syscalls`, `args`, `assert` — but
  `lib/io.cyr` includes `lib/result.cyr`, and `result`, `atomic` and `fnptr` were never declared. So
  `cyrius lib sync` skipped all three: they sat in `lib/` **vendored by accident**, with nothing
  keeping them current across toolchain moves. Now declared, so the sync copies **23** files instead
  of 20 and is reproducible from the manifest rather than from whatever happened to be on disk.
- ⭐ **`result.cyr` had genuinely diverged, and it was the one that mattered.** v6.6.0 makes `Result`
  a **value type** — `enum Result<T, E>: stack`, so `Ok(v)` / `Err(e)` return a value rather than an
  allocation. klug carried the **v5.8.28** heap form, i.e. a stale `Result` sitting underneath a
  6.6.0 `io.cyr` that includes it. (`: stack` shipped in v6.5.55; v6.5.67 made it safe by refusing
  lossy binds.) `atomic.cyr` and `fnptr.cyr` happened to still match, which is exactly why an
  accident-vendored file is dangerous: it reads as fine until one of them moves.

### Changed

- **Toolchain pin 6.5.41 → 6.6.0.** 0.1.6 shipped 6.5.41; an intermediate raise to 6.5.45 (alongside
  agnos 1.56.60) never got a klug release, so this entry covers both hops.
- **`lib/` re-vendored with `cyrius lib sync`, from the PIN.** ⚠ Deliberately not as a side effect of
  a build: a build rewrites `lib/` to whatever toolchain is **active**, which is how a sibling ends up
  vendoring a snapshot its own manifest never declared. All **23** `lib/*.cyr` now byte-match
  `~/.cyrius/versions/6.6.0/lib`.
- **The vendored agnos syscall wrapper caught up to the agnos 1.56.59/60 surface** — `SYS_MOUNTLIST`
  (**104**, 80-byte records) with `sys_mountlist`; the four `net_config` counter accessors
  `sys_net_tx_packets` / `sys_net_rx_packets` / `sys_net_tx_bytes` / `sys_net_rx_bytes` (fields
  **8–11**); and a tombstone comment where `SYS_BLKSTATS = 105` used to sit — that number was minted,
  shipped as a peer, then **withdrawn** in v6.5.45 once an extend-vs-mint audit found the counters
  belonged in `sysinfo`#35's tail. ⛔ Do not re-add 105: the agnos kernel has no `num == 105` arm, and
  a wrapper with no kernel arm does not error — the caller gets the dispatch fall-through and may
  render it as a statistic.
- **macOS/aarch64 wrappers gained the threading surface** — `sys_bsdthread_register` / `_create` /
  `_terminate` and `sys_ulock_wait` / `sys_ulock_wake`. Not reachable from klug; carried in because
  the snapshot is vendored whole per declared module.
- **Both binaries shrank on identical source**: `build/klug` **134,832 → 134,816** and
  `build/klug_agnos` **139,008 → 134,928** (−4,080 B), the compiler emitting less for the same input.


## [0.1.6] — 2026-09-02 (cyrius 6.5.41; the severity lens stops lying)

Toolchain pin **6.5.35 → 6.5.41**, two behavioural fixes, and lockstep support for the agnos
1.56.58 kernel-log timestamp. Tests **12 → 37**.

### Fixed

- **`klug -w` / `klug -e` reported "no warnings" on a kernel that has no severity lens at all.**
  The agnos kernel's `klug_info`/`klug_warn`/`klug_err` have exactly **three** call sites and all
  three sit inside `#ifdef EXEC_SELFTEST` (`kernel/core/main.cyr:2737-2739`, guarded `2589-3038`),
  which is off in every production build. So a real boot log carries **zero** tagged lines, every
  line scores level 0, and both filters printed nothing and exited 0 — indistinguishable from a box
  that genuinely logged no warnings. A tool that cannot tell those two apart is worse than one
  without the flag.
- New `klug_has_level_tag(p, len)` answers a question `klug_level_of` **structurally cannot**: that
  function returns `KLUG_LVL_ALL` (0) for a real `[I] ` *and* for an untagged boot line, because
  both must appear in a bare dump. `klug_dump` now counts tagged lines and, when a filter ran
  against zero of them, says so on **stderr**.
- ⚠ **stdout stays byte-exact and the exit code stays 0.** The message is a statement about the
  corpus, not a failed dump — so `klug -w | grep …` and every scripted caller are unaffected. The
  0.1.5 differential-fuzz property (byte-for-byte identical stdout) still holds.

### Changed

- **cyrius toolchain pin 6.5.35 → 6.5.41**, in lockstep with agnos 1.56.58. The vendored `lib/`
  snapshot resyncs with it (`syscalls_x86_64_agnos.cyr` gains `SYS_PROCLIST`#99 and
  `SYS_READDIR_AT`#101, plus the 6.5.37 AGNOS peers).
- ⛔ **On a dev box this resync happens whether you ask for it or not.** Measured 2026-09-02:
  `cyrius build --no-deps` in this repo rewrote all five vendored `lib/*.cyr` files to the
  **active** toolchain's stdlib, ignoring both `--no-deps` and the manifest pin — so a sibling repo
  cannot be built at its own pin without `cyriusly use <pin>` first. Related and separately
  confirmed: the versioned wrapper does not pin `cycc` either (cyrius
  `issues/2026-08-22-versioned-wrapper-does-not-pin-cycc.md`, unlanded). Before this bump, that
  combination left this repo one build away from a 6.5.41 stdlib under a 6.5.35 manifest.

### Added — the agnos 1.56.58 uptime prefix, seen through rather than tripped over

- agnos 1.56.58 prefixes every KERNEL-ORIGIN log line with a fixed-width 15-byte uptime field,
  Linux printk shape: `[    4.123456] ` (`kernel/core/kprint.cyr`, `klog_build_prefix`).
- ⛔ **That would have disarmed the severity lens silently.** `klug_level_of` tests byte 0 for `[`
  and byte 2 for `]`. Under a prefix byte 0 is still `[`, so the first guard passes — but byte 2 is a
  space, so EVERY line scored level 0 and `klug -w` printed nothing and exited 0. Same silent
  false-negative as the untagged-corpus bug above, from the opposite direction.
- New `klug_time_prefix_len` steps over the field. It requires **four** anchors — `[` at 0, `.` at 6,
  `]` at 13, space at 14 — because byte 0 alone false-positives on a real `[E] disk error` line
  (whose byte 6 is `s`, which is what declines it).
- ⚠ **The prefix is OPTIONAL and the unprefixed path is unchanged.** Lines emitted before the kernel
  timebase calibrates, all ring-3 output (agnos deliberately leaves userland bytes undecorated), and
  every log captured before 1.56.58 carry no prefix; `test_unprefixed_lines_still_work` pins that.

### Added — tests 12 → 37

- `klug_has_level_tag` coverage: tag-presence is asserted to be **distinct from level**
  (`[I] ` scores 0 but IS tagged; `ext2: mounted` scores 0 and is NOT), all three levels, and the
  malformed cases.
- `test_lens_sees_through_the_uptime_prefix` is the acceptance record for the agnos-side change;
  `test_time_prefix_needs_all_four_anchors` breaks each anchor in turn, so a future "simplification"
  down to a single `[` test reddens rather than silently mis-classifying real `[E]` lines.


## [0.1.5] — 2026-08-25 (P-1 hardening sweep)

A priority-1 audit/refactor/hardening/optimization/security sweep. No new features. Every behavioural
change below was checked against 0.1.4 with a 2000-case differential fuzz (random prefixes, adversarial
lengths, unterminated tails, newline-free buffers, every valid flag): **byte-for-byte identical output
and exit codes in every case that was already correct.**

### Fixed

- **`write(2)`'s return value was discarded at all three write sites, so a dump could be silently
  truncated while klug reported success.** This is the release. A single `write(1, buf, 65536)` is not
  a promise of 65536 bytes, and on the primary target it is very far from one: agnos `pipe_write`
  (`kernel/core/vfs.cyr`) refuses past `PIPE_RING = 4080` and returns a **short count on purpose** —
  its own comment reads *"a short return is the contract — the caller sees fewer bytes accepted than
  offered and can act on it. Silence cannot be acted on."* klug never looped, so
  `run /bin/klug | grep panic` — the invocation klug's own `--help` and README advertise as the safe
  one — delivered at most **4080 bytes of a 20–64 KB log** and exited 0. Since `klug`#36 returns the
  ring oldest→newest, a panic line is necessarily among the newest, so it was exactly the thing
  guaranteed to be dropped. Measured on a booted AGNOS kernel in QEMU with the shipped 0.1.4 binary.
  The same hole hit `run /bin/klug > /f.txt` on a full disk (`ext2_write_at` returns a partial count
  on ENOSPC) and, on Linux, any non-blocking stdout — reproduced there at **4095 of 52500 bytes,
  exit 0, empty stderr**.
  All writes now go through `klug_write_all()`, which loops to completion and reports failure.
  A zero return means different things on the two targets and is handled separately: on agnos it is
  backpressure from a pipe ring whose consumer has not been scheduled yet, so klug yields
  (`sys_sched_yield`) and retries under a bounded stall counter — an unbounded retry would trade
  truncation for a hang, because agnos gives the writer no reader-gone signal; on Linux it is
  pathological and fails. A short/failed dump now exits 1 with
  `klug: stdout write failed — the dump is INCOMPLETE` on **stderr**.
  ⚠ The agnos *console* arm was never affected: `serial_dev_write` chunks internally and returns the
  full count — it was rewritten precisely because of this bug. The kernel worked around klug on that
  one arm; the pipe and ext2 arms were left, and klug never grew the loop.
- **`--help` told operators to redirect onto `/klug.txt`, which is the kernel's reserved crash-spill
  file.** agnos `kernel/core/klug.cyr:172` hardcodes `/klug.txt` in `klug_spill_prepare()`, called at
  boot from `main.cyr:875`, which pre-allocates all 16 blocks at mount time so a post-mortem spill is
  a pure overwrite-in-place and never has to run the allocator with the console dead. Redirecting a
  short dump over it truncates the file and frees that preallocation. Worse on a latch-blocked
  recovery boot: `klug_spill_prepare()` deliberately targets `/klug-2.txt` there and never re-prepares
  `/klug.txt`, so the damage is not repaired by the next boot. klug was the **only** thing in the
  ecosystem naming that file — the agnos kernel, its docs and its CHANGELOG all say `/f.txt`, and
  klug's own README already cited `/f.txt` two lines away from printing `/klug.txt`. Now `/f.txt`
  everywhere.
- **Diagnostics were printed to stdout**, so they landed inside `klug > f.txt` and were piped into
  `klug | grep` — on a tool whose entire idiom is `klug | grep`. `println` (`lib/string.cyr`) writes
  to fd 1. All error prose now goes to fd 2 via `klug_err_line()`; `--help` stays on stdout, being
  output rather than an error.
- **An unknown flag was silently ignored and fell through to the unfiltered dump.** `klug --nonsence`
  was not a typo that got caught, it was the full console dump — which on agnos is the one that eats
  the ring. Unknown options now write to stderr and exit **2** (distinct from the exit 1 that means a
  read or write failed).
- **The docs claimed unprefixed lines "always show"; `-w`/`-e` drop them** — on agnos too, not just on
  the dev host. The gate is `level >= min_level` and unprefixed is level 0, so a raw `kprintln` panic
  banner is invisible under `klug -e`. `klug_level_of`'s contract is unchanged (it is coherent, and
  changing it would break `-e` = "only errors"); the comment, the README and the test name now
  describe what it actually does, and point at `klug | grep -i panic` for unprefixed banners. The
  README also now says that every `/dev/kmsg` record is unprefixed wire format (`PRI,SEQ,TS,FLAG;msg`),
  so `-w`/`-e` print nothing on the Linux host — the lens is an AGNOS feature.
- **`cyrius build --aarch64` failed outright**: `undefined variable 'SYS_OPEN'`. aarch64 Linux has no
  bare `open(2)`, so its syscall peer defines only `SYS_OPENAT` plus a portable `sys_open` wrapper.
  klug reached around the stdlib's arch selector with a raw `syscall(SYS_OPEN, …)`; it now calls
  `sys_open`, which is byte-identical on x86_64 (the x86_64 wrapper is literally that call). All four
  targets — host, `--agnos`, `--aarch64`, `--win` — now build, and the aarch64 binary was verified
  running correctly under `qemu-aarch64`. ⚠ The call is portable only where it sits: the *agnos* peer
  defines an unrelated three-argument `sys_open(name, namelen, flags)`, so this must stay inside its
  `#ifndef CYRIUS_TARGET_AGNOS` block, where it would otherwise compile silently and wrongly.

### Changed

- **The filtered path (`-w`/`-e`) coalesces adjacent passing lines into one write.** It previously
  issued one `write(2)` per emitted line. Adjacent passing lines are contiguous in the buffer, so a run
  is now held open and flushed when a dropped line or the end breaks it. Measured on a full ring of
  passing lines: **1500 writes → 1**. On agnos each of those writes took `fs_spin_lock` and drove a
  full `ext2_get_inode` → read-modify-write → `ext2_put_inode` round trip to the device, so the
  documented `klug -w > /f.txt` was thousands of block-device operations instead of a handful. Output
  is byte-for-byte unchanged (2000-case fuzz, plus explicit adjacent-run and unterminated-tail cases).
- **`klug_buf`'s size and `KLUG_RING_BYTES` can no longer drift apart.** The two were independent
  literals in different files, and the "ring-size contract" test pinned only one of them — shrinking
  the buffer 8× still passed 11/11, because `cyrius test` never compiles `src/main.cyr` and the harness
  cannot see `klug_buf` at all. A shared `enum KlugRing { KLUG_RING_WORDS = 8192; }` in `src/klug.cyr`
  now *is* the buffer's extent (`var klug_buf[KLUG_RING_WORDS]`), and a 12th assertion cross-checks
  `KLUG_RING_WORDS * 8 == KLUG_RING_BYTES`. The same sabotage now fails the suite. `KLUG_RING_BYTES`
  stays an independent literal on purpose — the check only has teeth while the two are stated
  separately. (`sizeof` cannot express this: cyrius's `sizeof` takes a type, not a variable.)

### Deliberately not done

Each of these was investigated, reproduced, and rejected with a reason — recorded so they are not
re-proposed:

- **Parsing the `/dev/kmsg` PRI field** to make `-w`/`-e` work on the Linux host. Implemented and
  measured: it *kills the agnos lens*, because `'['` (91) > `'9'` (57), so every prefixed line falls to
  level 0 and both `-w` and `-e` emit nothing on the target klug exists for. It also splits kmsg
  continuation lines from their parent record. That is feature work for a path the source itself calls
  dogfooding, not P-1 hardening.
- **Retaining the newest 64 KB on the host path** instead of the oldest. Implemented against a FUSE
  emulator of real `devkmsg_read` semantics: a **verified no-op**. Once the contiguous remainder is
  shorter than the next record, `read()` returns `-EINVAL` having *already consumed* it, so any scheme
  handing `read()` a shrinking count silently eats a record per buffer boundary. The premise was also
  wrong — agnos returns the whole ring (klug passes the full ring size, so `klug`#36's newest-N branch
  is dead code from this call site), so host and target do not in fact disagree.
- **Retrying the `/dev/kmsg` drain on `-EPIPE`.** The trigger needs >2000 printk records inside a ~1–3 ms
  drain window; and the proposed exit-code change regressed every ordinary large-log dump to exit 1.
- **Heap-allocating `klug_buf`** to halve the binary. It works (−65536 bytes) but replaces an infallible
  static buffer with a fallible 2 MB agnos mmap and adds a `sys_mmap` runtime dependency to a binary
  deliberately staged onto *older* kernels. Trading an infallible read path for a new OOM failure mode
  in the tool you reach for when the machine is sick is backwards. The underlying 64 KB of file-backed
  `.bss` is a cyrius ELF-writer artifact (`.rodata` is placed above `.bss` in the same RW `PT_LOAD`, so
  `p_filesz` spans the NOBITS window) affecting every cyrius binary, not a klug defect — and it costs
  ~320 bytes in the packed git object, not 64 KB.
- **Dropping `"assert"` from `[deps].stdlib`.** Saves 4224 bytes on agnos, but ~4096 of that is a
  one-time page-alignment artifact that the next KB of code takes back, and it drops `lib/assert.cyr`
  out of the `cyrius lib sync` closure so it silently freezes at the current pin — the exact stale-`lib/`
  failure mode 0.1.4 existed to fix.

### Added

- **`klug --help` now warns, on AGNOS builds only, that dumping the log to the console consumes
  the log — redirect instead (`run /bin/klug > /f.txt`).** fd 1 is a `VFS_DEVICE`, so klug's own
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

  ⚠ The note itself is gated `#ifdef CYRIUS_TARGET_AGNOS` — the hazard is a property of the agnos
  console path and has no meaning against `/dev/kmsg`, so this entry alone changes only
  `build/klug_agnos`. (When it landed it left the host build byte-identical to 0.1.4's; the rest of
  0.1.5 changes both binaries, so that no longer holds for the release as a whole.)

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
