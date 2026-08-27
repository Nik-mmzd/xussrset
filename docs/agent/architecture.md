# Architecture

## Build pipeline

Every GRF is built from a `.pnml` root through two stages:

```
<name>.pnml --(gcc -E -C -P -x c  -D REPO_REVISION=N -D MIN_COMPATIBLE_REVISION=M)--> <name>.nml --(nmlc)--> build/<name>.grf
```

`.pnml` means "preprocessed NML": `#include`, `#define`, `##` token pasting are
GNU cpp features, not NML. `nmlc` never sees them. This shapes the entire code
style — see [templates.md](templates.md).

`compile.sh` also writes `custom_tags.txt` (git-ignored) before each nmlc run.
Of its four keys only `{TITLE}` is consumed — it is the value of every
`STR_GRF_*_NAME` string in `lang/*.lng`. `REPO_REVISION` /
`MIN_COMPATIBLE_REVISION` reach the `grf {}` block via the gcc `-D` defines, not
via custom tags.

## Modules

Each module is a standalone GRF; most of them are also assembled into the
combined `xussr.grf`. Entry points are two-line stubs in the repo root
(`xussr-<module>.pnml`) that include the real assembly file in
`src/<module>/`.

| Module | GRF ID | Assembly file | In combined? |
|---|---|---|---|
| combined `xussr` | `AKA\08` | `src/xussr.pnml` | — |
| steam | `Meo\B2` | `src/steam/steam-xussr.pnml` | yes |
| diesel | `Meo\B3` | `src/diesel/diesel-xussr.pnml` | yes |
| electric | `Meo\B4` | `src/electric/electric-xussr.pnml` | yes |
| dmu | `Meo\B5` | `src/dmu/dmu-xussr.pnml` | yes |
| emu | `Meo\B6` | `src/emu/emu-xussr.pnml` | yes |
| wagons (freight) | `Meo\B7` | `src/wagons/wagons-xussr.pnml` | yes |
| cars (coaches) | `Meo\B8` | `src/cars/cars-xussr.pnml` | yes |
| subway | `Meo\B9` | `src/subway/subway-xussr.pnml` | yes |
| addon (foreign high-speed stock) | `Meo\B0` | `src/addon/xussr-addon.pnml` | no |
| rails (track set) | `Meo\B1` | `src/rails/xussr-rails.pnml` | no |
| stationratings | `Meo\BA` | `src/override/stationratings.pnml` | no |

The combined set carries a different GRF ID (`AKA\08`) so it can coexist with
the split `Meo\Bx` GRFs. `stationratings` is disabled in `compile.sh`'s `all`
list and not built by CI.

### Standalone vs combined assembly

`src/xussr.pnml` and each `src/<mod>/<mod>-xussr.pnml` follow the same recipe:
shared infra (`check.pnml`, `basecost.pnml`, `cargotable.pnml`,
`railtypetable.pnml`, `definition.pnml`, `template.pnml`, the
`code-templates-*` family, `recolour.pnml`, `dummy/`, `align/`), then graphics
templates, then the module's `<mod>-list.pnml`, then a
`sort (FEAT_TRAINS, [...])` block. Differences:

- **Header.** Combined uses `src/header.pnml`; each module has its own
  `<mod>-header.pnml` exposing only the GRF parameters relevant to it.
- **Graphics templates.** A standalone module includes only the
  `graph-templates-*` files its vehicles need (emu pulls the MU templates but
  not the loco ones, and so on).
- **`src/disable-origin.pnml`** (disables all original TTD trains) is included
  only by the combined build.
- **`src/definition-cross.pnml`** (cross-module consist rules: TEP70BS+ES1/ES2G,
  2M62U+DR1A) is included by the combined build and by the modules involved.
- **Bleed-over includes.** A standalone module pulls the minimum slice of a
  foreign module it needs, e.g. emu includes `../wagons/cargoes-all.pnml` and
  a couple of capacity files.
- **subway** is the odd one out: it uses `../template_v2.pnml` instead of
  `../template.pnml` and borrows `../emu/emu-code-templates-calc.pnml`.

## GRF parameters (combined set, `src/header.pnml`)

| Param | Names | Meaning |
|---|---|---|
| 1 | `speed_penalty_percent` | 0–50, default 15 |
| 2 | `disable_steamer`, `disable_diesel`, `disable_{ac,dc,acdc}electric`, `disable_dmu`, `disable_{ac,dc,acdc}emu`, `disable_subway`, `disable_wagon`, `disable_car` | bit flags, bits 0–11 |
| 3 / 4 | `costs_multiplier` / `rcosts_multiplier` | 1–10000, default 100; applied inside the `purchase_menu()` / `cost_factor` macros |
| 5 | `new_cargo_ageing` | 1–10000, default 100 |
| 6 | `disable_groups`, `disable_wagon_groups`, `disable_icons`, `disable_long_names` | bit flags |

Every vehicle file ends with an `allow_<class>(name)` macro
(`src/code-templates.pnml`) that re-opens the item and sets
`climates_available: NO_CLIMATE` when the matching `disable_*` bit is set.
Module headers declare only their own subset of these bits.

## Compatibility checks

- `src/check-main.pnml` — requires OpenTTD >= 1.14.0.
- `src/check.pnml` — requires `dynamic_engines`; detects foreign GRFs (YETI,
  ECS Chemicals II / Machinery, SETS, the xUSSR rails GRF) via
  `grf_current_status`/`grf_future_status` and sets global flags (`yeti_on`,
  `ecs_chem_ii_on`, `xUSSR_rails_not_board`, …) consumed by wagon capacity
  code and the addon's railtype logic.
- `src/rails/check.pnml` — load-order check for the rails set.

## Versioning and releases

- `versions/<name>.ver` holds each GRF's integer revision, committed to git.
  `./compile.sh -b <module>` bumps it. `REPO_REVISION` = that number;
  `min_compatible_version` is 1 for modules and 496 for the combined set
  (`XUSSR_MODULE_MIN_REV` in `compile.sh`, mirrored in
  `.github/workflows/release.yaml`).
- The human-readable version string comes from `git describe --tags`.
- CI (`.github/workflows/release.yaml`) builds all GRFs on every push with the
  `Nik-mmzd/nml-compile@v5` action (which pins the nmlc toolchain), uploads
  artifacts, publishes a `nightly/<branch>` prerelease, and a real release on
  tags.

## Vehicle ID space

`item (FEAT_TRAINS, <identifier>, <numeric_id>)`:

- 0–115 — original TTD vehicles (disabled by `src/disable-origin.pnml`);
- 116–123 — `dummy1..dummy8`, invisible articulated spacers
  (`src/dummy/dummy.pnml`);
- 124+ — the set's vehicles and group items, registered in `src/IDs_usage`
  (see [strings-and-ids.md](strings-and-ids.md)).

Identifiers can't start with a digit, so numeric series get a leading
underscore: `_2te10`, `_3m62u_m`.

## Purchase-list order

Order in the buy menu is set by `sort (FEAT_TRAINS, [...])`, not by IDs or
include order. Each module has a `<mod>-sort-order.pnml` **fragment** — a bare
comma-separated identifier list (group parent first, then members) that is
`#include`d inside the `sort()` array of both the standalone and the combined
assembly. Never wrap a fragment in `sort()` yourself; the only exception is
`src/addon/sort-order.pnml`, which contains the full statement because the
addon is its sole consumer.
