# klug — Roadmap

This file lists open work only. Shipped history lives in [`../../CHANGELOG.md`](../../CHANGELOG.md).

## Adopting cmdit (CLI parsing)

Surveyed against cmdit 1.2.6. Its next tag, `1.3.0`, has the same API.
This is demand-gated, not a forced sweep, and the **verdict is no
migration today**. klug's whole CLI is three exact-match bool flags in a
20-line loop (`src/main.cyr:22-48`). cmdit would delete about ten of those
lines. In exchange it would add a dependency, a mandatory `--version`, and
five edge-case behavior changes. That is a poor trade on a tool whose rule
is "flags are rejected, never ignored" (`src/main.cyr:21`, `:36-44`).
Revisit if klug grows a value flag (a `--level LEVEL` is a `cmdit_enum`).

Site map:

- The `args_init` + `argc()`/`argv(i)` walk (`src/main.cyr:23-28`) →
  `cmdit_parse(h)` on `cmdit_new("klug")`.
- `-h`/`--help` (`:30-31`) → cmdit's auto help. On `CMDIT_HELP`, call the
  existing `klug_usage()` (`src/klug.cyr:198-241`). It must stay: it prints
  to stdout with per-target `#ifdef` blocks, CI asserts that stdout
  (`.github/workflows/ci.yml:89-90`), and `cmdit_help`/`cmdit_help_flags`
  write fd 2.
- `-w`/`--warn` and `-e`/`--err` (`:32-35`) → two `cmdit_bool`s.
- Unknown options and any positional currently get `klug_err_line` on fd 2
  and exit 2 (`:41-44`). Map that to `CMDIT_RESULT_ERR` plus
  `cmdit_require_positionals_max(h, 0)`, printing the same line.
  `CMDIT_EXIT_USAGE` is already 2.
- klug has no `--version`, env fallback, int or str flag, or verb.

What a swap changes (each needs a decision, not a default):

- [ ] `-e -w` is WARN today because the last flag wins. cmdit records no
  order between two bools, so the caller has to choose a precedence.
- [ ] `--` exits 2 today. cmdit consumes it, so `klug --` would become the
  unfiltered dump. On AGNOS that is the console dump that ages the ring
  (`src/klug.cyr:208-225`).
- [ ] `--err=0` exits 2 today. cmdit ignores a bool's value and would
  apply the lens (cmdit `src/cmdit.cyr:569`). The stdlib parser rejects
  this case (`FLAG_ERR_UNEXPECTED_VALUE`), but cmdit does not. That gap
  should close upstream first.
- [ ] `-h --bogus` prints help and exits 0 today, because `-h` returns
  mid-loop. cmdit scans the whole argv first, so it would exit 2.
- [ ] `--version` exits 2 today. cmdit always registers it; only `-V` can
  be dropped, with `cmdit_version_short(h, 0)`. klug would then need a
  `klug X.Y.Z` literal, and nothing checks that against `VERSION`
  (`release.yml:22-26` checks only the tag).

If it happens:

- [ ] `cyrius.cyml`: `"args"` and `"io"` are already in `[deps].stdlib`,
  and there is no `"flags"` entry to drop. Add:

  ```toml
  [deps.cmdit]
  git = "https://github.com/MacCracken/cmdit.git"
  tag = "1.3.0"
  modules = ["dist/cmdit.cyr"]
  ```
- [ ] Tests: none to migrate. `tests/klug.tcyr` includes only
  `src/klug.cyr`, so `main` has no tests, and there is no fuzz harness.
- [ ] Bar: CI's `Smoke` step (`.github/workflows/ci.yml:58-123`) must stay
  green: `--help` on stdout with exit 0, `--bogus` with exit 2 and an
  empty stdout, and the `-w`/`-e` lens rows as root. Add the five cases
  above as new rows.

See cmdit's `README.md` (API) and `docs/development/roadmap.md` (v2
backlog). kii (`src/cli.cyr`) and anuenue (`src/main.cyr`) are the worked
migrations.

### Update — cmdit 1.3.1 (2026-10-14)

cmdit 1.3.1 rejects a bool given `=value` with
`CMDIT_ERR_UNEXPECTED_VALUE`, so `klug --err=0` would stay a parse error
under cmdit instead of applying the lens. That removes the `--err=0` item
from the list above, leaving four behavior changes. 1.3.1 also keeps
positionals past the 128th instead of dropping them silently; klug
rejects positionals anyway. The verdict stands: no migration until klug
grows a value flag. If it does, pin `tag = "1.3.1"`.
