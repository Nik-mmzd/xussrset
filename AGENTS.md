# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this is

**xUSSR Set** — a NewGRF train set for OpenTTD: locomotives, multiple units
and rolling stock of the former Russian Empire, USSR and post-Soviet
countries. Written in NML with a C-preprocessor macro layer on top (`.pnml`
files), compiled to `.grf` with `nmlc`.

Detailed documentation for agents lives in `docs/agent/`:

- [architecture.md](docs/agent/architecture.md) — build pipeline, modules, GRF IDs, params, versioning
- [adding-a-vehicle.md](docs/agent/adding-a-vehicle.md) — vehicle file anatomy and the full checklist
- [templates.md](docs/agent/templates.md) — the macro/template system reference
- [cargo-system.md](docs/agent/cargo-system.md) — wagon capacity/refit/load-graphics model
- [strings-and-ids.md](docs/agent/strings-and-ids.md) — language files, string naming, usage registries
- [tooling.md](docs/agent/tooling.md) — scripts, Windows/CI build flows, known gotchas

## Build

```sh
./compile.sh emu          # one module        → build/xussr-emu.grf
./compile.sh combined     # the full set      → build/xussr.grf
./compile.sh all          # all enabled modules
./compile.sh -b emu       # bump versions/xussr-emu.ver, then compile
```

Pipeline per GRF: `<name>.pnml → gcc -E (cpp) → <name>.nml → nmlc → build/<name>.grf`.

Toolchain: `gcc`, `nmlc` (`pip install nml`, **NML >= 0.7.6** — the combined
build overflows the D0xx string range on 0.7.5), `git`, plus GNU `getopt` and
GNU `sed` — **stock macOS BSD tools fail**; install `gnu-getopt`/`gnu-sed` and
put them on PATH. CI
(`.github/workflows/release.yaml`) builds every push into nightly
prereleases; tags become releases.

## Validation

There are no tests. The consistency check is:

```sh
perl scripts/clean-lng.pl   # from the repo root
```

It regenerates the committed registries `src/IDs_usage`, `src/PNGs_usage`,
`src/PNMLs_usage`, normalises `lang/*.lng`, and **dies** on duplicate vehicle
IDs or on strings used in code but missing from every language file
(`lang/_Missing.txt`). Neither `compile.sh` nor CI runs it — run it yourself
after touching vehicles, sprites or strings, and commit the regenerated
files. Also always verify the build: `./compile.sh combined` plus the
affected module.

## Architecture in brief

- **Modules**: steam, diesel, electric, dmu, emu, subway, cars (coaches),
  wagons (freight) — each builds standalone *and* into the combined
  `xussr.grf`; addon (foreign high-speed stock), rails (track set) and
  stationratings are standalone-only. Entry points are two-line stubs in the
  repo root (`xussr-<module>.pnml`) including `src/<module>/<module>-xussr.pnml`;
  the combined assembly is `src/xussr.pnml`.
- **Everything is cpp macros.** Shared `#define` template families live in
  `src/definition.pnml`, `src/code-templates-*.pnml` (logic switches) and
  `src/graph-templates-*.pnml` (spritesets); sprite layouts in
  `src/template.pnml`. Macro bodies are `\`-continued lines with the
  backslashes column-aligned; identifiers are built by token pasting
  (`name##_sprites_left`).
- **A vehicle** is one `.pnml` + one sprite-sheet `.png` side by side under
  `src/<module>/<manufacturer>/`, following a fixed skeleton: `PROP_<id>_*`
  constants → spritesets (`#define IMAGEFILE` / `#undef` pairs, repo-root
  relative paths) → livery/year switches → consist logic → `item` block with
  a numeric ID from `src/IDs_usage` → `allow_<class>(id)` tail. See
  [adding-a-vehicle.md](docs/agent/adding-a-vehicle.md).
- **Buy-menu order** comes from per-module `<mod>-sort-order.pnml` fragments
  pasted into `sort()` blocks — new vehicles must be added there too.
- **Strings**: `lang/english.lng` is the master; always update it and
  `lang/russian.lng`; other languages are optional. String IDs follow
  `STR_NAME_<VEHID>` / `STR_LONGNAME_<VEHID>`.

## Conventions

- File and sprite names are lowercase (`clean-lng.pl` enforces by renaming).
- `graphics {}` blocks: keys alphabetically sorted, colon-aligned. Macro
  continuation backslashes right-aligned (house style from
  `scripts/MonaLisa.pl`).
- Sprite sheets must match the blank templates in `src/align/templates/`;
  every sprite switch defaults to `align_<n>_sprites` (visible magenta grid),
  never to a silent fallback.
- Vehicle performance numbers come from `docs/xUSSR Set.xlsx`, not from
  guesswork.
- Preserve encodings: `.pnml`/`.lng` are UTF-8 (some with BOM), the
  `*_usage` registries are CRLF, `.bat` files are CP866.

## Do not touch

`openttd-src/` and `jgrpp-src/` are local reference checkouts of OpenTTD and
JGRPP, excluded via `.git/info/exclude`. Never commit them and exclude them
from code searches over the set's own code.
