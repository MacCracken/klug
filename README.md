# klug

**klug** — the userland reader for the AGNOS kernel log. A sovereign `dmesg`.

klug is the *Kernel Logs Unified Grep* subsystem. The **kernel half** (the agnos
kernel's `core/klug.cyr`) unifies every line of kernel output — boot messages,
driver logs, leveled `klug_info`/`warn`/`err` — into one 64 KB ring buffer. This
**userland half** is the binary that dumps that ring to stdout.

The "grep" in the name is deliberate: klug *unifies and dumps*; filtering stays
the shell's job.

```sh
klug                  # dump the whole kernel log
klug -w               # only warnings and errors
klug -e               # only errors
klug | grep panic     # filter — grep is the agnsh builtin
sudo klug -e          # Linux dev host: err-and-worse records from /dev/kmsg
```

## How it works

- **AGNOS** (native): the `klug`#36 syscall copies the kernel's klug ring into a
  64 KB buffer (oldest→newest; a smaller buffer would get the newest-N tail), and
  klug writes it to stdout. The syscall only ever copies the log ring, never other
  kernel memory.
- **Linux** (dev-host dogfooding): reads `/dev/kmsg` so the tool runs and is
  testable off-target. Records are dumped as the raw wire format the kernel
  hands out (`PRI,SEQ,TS,FLAG;msg`, plus ` KEY=VALUE` continuation lines on
  device records) — klug does not reformat. The whole log is dumped, however
  big: it is streamed through the 64 KB buffer a batch at a time (through 0.2.0
  klug stopped when the buffer filled, so on a host with more than 64 KB of log
  the newest records were silently missing). If the kernel overwrites records
  faster than klug reads them, klug carries on and says on stderr that the dump
  has a gap. Since 0.2.0 the severity lens works here too, off each record's
  syslog priority (below).

The severity lens (`-w`/`-e`) keys off the `[I]`/`[W]`/`[E]` prefixes the kernel
emits. Unprefixed lines — boot output and raw `kprintln`, which is most of an AGNOS
boot log — are level 0, the same as `[I]`: they appear in the default dump, but
`-w`/`-e` drop them. A panic banner emitted by raw `kprintln` carries no prefix, so
hunt one with `klug | grep -i panic`, not `klug -e`.

⚠ **On a production AGNOS kernel there are currently NO tagged lines at all.** The kernel's
`klug_info`/`klug_warn`/`klug_err` have three call sites and all three sit inside
`#ifdef EXEC_SELFTEST`, which is off in every shipping build — so `-w`/`-e` legitimately match
nothing. Since 0.1.6 klug says so on stderr rather than printing nothing and exiting 0, because
"no warnings" and "no lens" are otherwise the same output.

Since AGNOS 1.56.58 every kernel-origin line also carries a Linux-style uptime field,
`[    4.123456] `, ahead of any severity tag. klug steps over it, so `-w`/`-e` work on prefixed
and unprefixed logs alike. Ring-3 program output is deliberately left undecorated by the kernel,
so it never carries the field.

On the Linux dev-host path the lens reads each `/dev/kmsg` record's `PRI` field
instead: `PRI & 7` is the syslog level (0 emerg … 3 err, 4 warning, 5 notice, 6
info, 7 debug), so `-e` keeps level ≤ 3 and `-w` keeps level ≤ 4 — the same slice
`dmesg --level=emerg,alert,crit,err[,warn]` makes. Facility bits are ignored (a
userspace writer's `12,` is user.warning, still a warning). A ` KEY=VALUE`
continuation line has no priority of its own and inherits its record's, so
`klug -e` keeps an error's `SUBSYSTEM=`/`DEVICE=` context with the error. The
two lenses never mix: the binary is told at build time which log it reads, so a
ring-3 program's `3,foo` on AGNOS is just a line, and a `[E]` inside a `/dev/kmsg`
message is just text. Reading `/dev/kmsg` needs root or `kernel.dmesg_restrict=0`;
without it klug says so on stderr and exits 1.

## Build

```sh
cyrius build src/main.cyr build/klug          # host (Linux, /dev/kmsg)
cyrius build --agnos src/main.cyr build/klug_agnos   # AGNOS (klug#36)
cyrius test tests/klug.tcyr                    # tests
```

The suite runs in-process on the host and covers both lenses, the short-write loop
under both targets' zero-return policies (the AGNOS pipe-backpressure arm included —
driven through a scripted writer, so it no longer needs iron to execute), the
filter over a whole log, byte-exact, and the `/dev/kmsg` drain (a log bigger than
the buffer, end of log, gaps, retries and failed reads, through a scripted reader).
CI additionally executes the host binary as root against the runner's real
`/dev/kmsg`, after writing more than 64 KB into it, and checks that the newest
records come out.

On AGNOS, klug is staged onto the agnos-fs `/bin` via `agnos/scripts/burn/stage-tools.sh`
and run through the shell's exec-from-disk path.

## On AGNOS: redirect, don't dump

**Reading the log on the console consumes the log.** Redirect it:

```sh
run /bin/klug > /f.txt        # do this
run /bin/klug                 # this eats the boot log
```

Every console-bound ring-3 byte is fed back into the klug ring: fd 1 is a
`VFS_DEVICE`, so a `write(1, …)` takes `vfs_write`'s device arm → `dev_write` →
`serial_dev_write` (`core/devs.cyr`) → `kprint` → `klug_append` — the very ring
`klug`#36 just read. Since the dump is up to 64 KB and the ring is exactly 64 KB,
a console dump re-appends the whole log to itself. While the ring is still short
of full (a ~20 KB boot log) that *doubles* it; the next dump wraps and takes the
boot head with it.

A pipe is safe for the same reason the redirect is — a pipe fd takes `vfs_write`'s
`VFS_PIPE` arm and never reaches `kprint`, so only grep's matched lines are
console-bound:

```sh
run /bin/klug | grep panic    # fine — only the matches hit the console
```

This is a kernel-side property, not a klug bug. agnos records it as unfixed by
decision (`docs/development/state.md`, since 1.56.48), and its fault path carries
the same `run /bin/klug > /f.txt` note (`kernel/core/syscall.cyr`,
`fault_kill_current`). `klug --help` repeats the warning on AGNOS builds.

## License

GPL-3.0-only.
