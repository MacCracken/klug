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
```

## How it works

- **AGNOS** (native): the `klug`#36 syscall copies the kernel's klug ring into a
  64 KB buffer (oldest→newest; a smaller buffer would get the newest-N tail), and
  klug writes it to stdout. The syscall only ever copies the log ring, never other
  kernel memory.
- **Linux** (dev-host dogfooding): reads `/dev/kmsg` so the tool runs and is
  testable off-target.

The severity lens (`-w`/`-e`) keys off the `[I]`/`[W]`/`[E]` prefixes the kernel
emits; unprefixed lines (boot output, raw `kprintln`) always show.

## Build

```sh
cyrius build src/main.cyr build/klug          # host (Linux, /dev/kmsg)
cyrius build --agnos src/main.cyr build/klug_agnos   # AGNOS (klug#36)
cyrius test tests/klug.tcyr                    # tests
```

On AGNOS, klug is staged onto the agnos-fs `/bin` via `agnos/scripts/burn/stage-tools.sh`
and run through the shell's exec-from-disk path.

## On AGNOS: redirect, don't dump

**Reading the log on the console consumes the log.** Redirect it:

```sh
run /bin/klug > /klug.txt     # do this
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

This is a kernel-side property, not a klug bug, and it is tracked on the agnos
side (`docs/development/state.md`, and the `run /bin/klug > /f.txt` note at
`kernel/core/syscall.cyr:442`). `klug --help` repeats the warning on AGNOS builds.

## License

GPL-3.0-only.
