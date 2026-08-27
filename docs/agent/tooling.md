# Tooling, build flows, and gotchas

## Three build flows

### `compile.sh` (Linux/macOS — the reference)

```
./compile.sh [-b|--bump-revision] [-M|--min-revision N] [-B|--basename X]
             [-V|--grf-version S] [-D|--delete-nml] <modules…|all|combined>
```

Per module: read `versions/<name>.ver` → write `custom_tags.txt` →
`gcc -E -C -P -x c` with `-D REPO_REVISION` / `-D MIN_COMPATIBLE_REVISION` →
one `sed` pass to re-break lines → `nmlc` → `build/<name>.grf`. Module titles,
enable flags and per-module `min_compatible_revision` overrides live in the
`XUSSR_MODULE_*` arrays inside the script.

Requires GNU `getopt` (long options) and GNU `sed -i` — **stock macOS BSD
tools fail**; install `gnu-getopt`/`gnu-sed` (Homebrew) and put them on PATH.
Also needs `git` (for `git describe`), `gcc`, `nmlc`.

### `compile*.bat` (Windows — the historical maintainer flow)

Twelve near-identical copies (`compile.bat` = combined, `compile-<mod>.bat`
per module, `compile-all.bat` chains them). Encoded in **CP866** — read via
`iconv -f CP866`. Config comes from `..\config.bat` (one level **above** the
repo root; sample: `config.bat.sample`; expects MinGW gcc and NML 0.7.5).

The `.bat` flow does more than `compile.sh`:

- runs `scripts/clean-lng.pl` first and aborts on its errors;
- runs `scripts/MonaLisa.pl` (the formatter) over `lang/` and `src/`;
- runs `scripts/CleanCargoesLists.pl` on the generated `.nml` (comments out
  duplicate cargo-label cases produced by overlapping `cargo_check_*list`
  groups) — `compile.sh` does **not** do this;
- keeps revision counters on a personal cloud drive and hard-codes marketing
  version strings (`0.8.2.r…`) that drift from `git describe`.

### CI (`.github/workflows/release.yaml`)

Every push: `Nik-mmzd/nml-compile@v5` (pins the nmlc toolchain) builds each
GRF using `versions/*.ver`, uploads `build/*.grf` as an artifact, publishes a
`nightly/<branch>` prerelease; tags produce a real release. CI runs neither
`clean-lng.pl` nor `CleanCargoesLists.pl`.

## Perl scripts inventory

| Script | Purpose | Status |
|---|---|---|
| `clean-lng.pl` | lng normaliser + usage-registry generator + duplicate-ID / missing-string guard | **the** validation tool; run manually (see [strings-and-ids.md](strings-and-ids.md)) |
| `MonaLisa.pl` | formatter: 2-space bracket indent, trailing-`\` column alignment, `:` alignment | defines the house style; wired into `.bat` flow |
| `CleanCargoesLists.pl` | dedups repeated cargo-label cases in the generated `.nml` | relevant; Windows flow only |
| `Change.pl` | generic regex replace; re-breaks the one-line `.nml` | superseded by `sed` in `compile.sh` |
| `Adjust2Master.pl` | syncs sections between master/slave files: `lng` mode (english.lng → other languages), `ct2cl` mode (cargotable.pnml → capacity/loadspeed lists) | relevant in principle; driven by `prepare.bat`, whose paths are stale |
| `Add_props_wagon.pl` | one-shot generator of a wagon `-group.pnml` | Windows-only paths; useful as documentation of the group-item shape |
| `copy-branch.pl` | copies GRFs to a personal cloud folder | obsolete (replaced by CI) |

## Known gotchas

1. **The build flows are not equivalent.** Only the Windows `.bat` path runs
   `clean-lng.pl` (ID/string guard) and `CleanCargoesLists.pl`. On the
   sh/CI path, run `perl scripts/clean-lng.pl` yourself.
2. **`src/template_v2.pnml`** (+1px sprite widths) is used only by subway;
   don't unify it with `template.pnml` without re-checking sprite sheets.
3. **`prepare.bat` is broken** — its paths still say `src\freight\`, renamed
   to `src/wagons/` long ago.
4. **Encodings:** `.bat` files are CP866; `.pnml`/`.lng` are UTF-8, several
   with a BOM; the generated `*_usage` registries are CRLF. Preserve what you
   find.
5. **`custom_tags.txt`** carries four keys but only `{TITLE}` is consumed;
   version numbers actually travel via gcc `-D` defines.
6. **`xUSSR_musa.ini`** (BaNaNaS metadata) claims OpenTTD 1.13.1 minimum;
   the code (`src/check-main.pnml`) enforces 1.14.0.
7. **`src/code-templates-strings.pnml` is included twice** in `src/xussr.pnml`
   — harmless, historical.
8. `openttd-src/` and `jgrpp-src/` are local reference checkouts of OpenTTD /
   JGRPP, excluded via `.git/info/exclude` (not `.gitignore`). Never commit
   them; don't search them when working on the set itself.
9. **The combined build needs nml >= 0.7.6.** On 0.7.5 `./compile.sh combined`
   dies with `Unable to allocate ID for string, no more free IDs available
   (maximum is 1024)`; OpenTTD/nml#326 moved most strings to the DCxx range.
   Per-module builds still work on 0.7.5.
10. **The `sed` pass splits lines on `; `.** A trailing comment after a
   statement is fine — it just moves to its own line — but a `; ` *inside*
   comment text strands the rest of the sentence on a line with no `//`, and
   `nmlc` then reads it as code. Keep `; ` out of comment prose.
11. **`clean-lng.pl` rewrites `lang/*.lng` with CRLF endings**, though the
   committed files use LF. Convert them back before committing, otherwise
   every language file shows up as fully rewritten. Its `*_usage` output is
   also order-unstable — revert registries it churned without real changes.

## Design data in `docs/`

Not consumed by the build, but the source of truth for numbers:

- `xUSSR Set.xlsx` — master vehicle database (the `PROP_*` values);
- `ral.xlsx` — palette/recolour data behind `src/recolour.pnml`;
- `cargoes.vsd` / `tankers.vsd` (+ `cargoes.png`) — cargo taxonomy diagrams;
- `Electric.xlsx`, `types.xlsx` — working data;
- `changelog.txt` / `changelog_ru.txt`, `readme.txt` / `readme_ru.txt`,
  `license.txt` (CC BY-NC-SA 3.0 for code/graphics/sound; GPL 2.0+ for build
  and translation-status files).
