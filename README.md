# klug

**klug** — the userland reader for the AGNOS kernel log. A sovereign `dmesg`.

klug is the *Kernel Logs Unified Grep* subsystem. The **kernel half** (the agnos
kernel's `core/klug.cyr`) unifies every line of kernel output — boot messages,
driver logs, leveled `klog_info`/`warn`/`err` — into one 16 KB ring buffer. This
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

- **AGNOS** (native): the `klog`#36 syscall copies the kernel's klug ring into a
  16 KB buffer (oldest→newest; a smaller buffer would get the newest-N tail), and
  klug writes it to stdout. The syscall only ever copies the log ring, never other
  kernel memory.
- **Linux** (dev-host dogfooding): reads `/dev/kmsg` so the tool runs and is
  testable off-target.

The severity lens (`-w`/`-e`) keys off the `[I]`/`[W]`/`[E]` prefixes the kernel
emits; unprefixed lines (boot output, raw `kprintln`) always show.

## Build

```sh
cyrius build src/main.cyr build/klug          # host (Linux, /dev/kmsg)
cyrius build --agnos src/main.cyr build/klug_agnos   # AGNOS (klog#36)
cyrius test tests/klug.tcyr                    # tests
```

On AGNOS, klug is staged onto the agnos-fs `/bin` via `agnos/scripts/stage-tools.sh`
and run through the shell's exec-from-disk path.

## License

GPL-3.0-only.
