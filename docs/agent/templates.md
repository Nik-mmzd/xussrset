# The template system

Everything is built on the GNU C preprocessor (`gcc -E` runs before `nmlc`).
Consequences that shape the code style:

- A "template" is a `#define` whose body is one huge continued line — every
  source line ends with `\`, right-aligned in a column. That alignment is
  maintained by `scripts/MonaLisa.pl`.
- Identifier construction uses token pasting: `name##_sprites_left`,
  `s##n##_template`, `PROP_##name0##_CF`.
- C macros have no useful varargs here, so arity is encoded in the macro name:
  `_template2`, `_template4m2`, `MU_power_template12` — hence the huge families
  of near-identical macros.
- Sprite paths are passed via `#define IMAGEFILE "..."` / `#undef IMAGEFILE`
  around the template call.
- The generated `.nml` is unreadable one-liners; both build flows re-insert
  newlines afterwards (`sed` in `compile.sh`, `scripts/Change.pl` in the
  `.bat` flow).

## Sprite layout templates

- **`src/template.pnml`** — the active layout table: NML `template` blocks
  `sN_template(x, y, shift)` / `sN_r_template(...)` for lengths N = 3…17
  (`_r` = reversed), plus `new_purchase_template*`.
- **`src/template_v2.pnml`** — a +1px-width revision used **only by
  `src/subway/`**; commented out in `src/xussr.pnml`. Do not unify the two
  without re-checking every sprite sheet against the new widths.
- Alignment/debug sprites: `src/align/align.pnml` defines
  `align_N_sprites` (magenta grids, the mandatory default branch of every
  sprite switch). Blank artist reference sheets, one per body-template layout,
  are in `src/align/templates/`.

## `src/definition.pnml`

Global constants and property-block macros (~150 defines):

- named magic numbers (`INVALID_ENGINE`, `MAXSPEED`, `ENGINE_WAGON_CF`),
  cargo-ageing scale (`CAP_001_NORMAL`, …), loading physics (`WAGON_DOOR`,
  `DOUBLE_DOOR`, `*_HATCH`, `*_FALL`);
- era constants: `USSRSTART 1918`, `USSREND 1991`, `MAIDAN 2014`,
  `GREAT_CHANGE_YEAR 1972`, `PID_YEAR 2009`;
- computed helpers: `advanced_check_year()`, `calculate_capacity()`,
  `calculate_loading_speed()`;
- property-block families: `vehicle_dates`, `vehicle_steam`, `vehicle_diesel`,
  `vehicle_emu_c`, `vehicle_wagon`, `vehicle_pass()`, `vehicle_group*`,
  `group_steam/electric/wagon`, and `purchase_menu*` (which applies
  `costs_multiplier` / `rcosts_multiplier`).
- **pricing is callback-only**: `vehicle_dates`/`vehicle_no_dates` hard-code
  the `cost_factor` *property* to 1 (the argument is commented out); the real
  purchase price lives solely in the CB36 callback emitted by
  `purchase_menu()`. A vehicle whose callback chain is missing or broken
  shows the engine base price × 1 (~124k credits) in the buy menu — that
  price is the diagnostic signature of a lost `cost_factor:` wiring.

`src/definition-cross.pnml` is narrow: hard-coded cross-module consist
validation for real hybrid trains (TEP70BS+ES1/ES2G, 2M62U+DR1A), with vehicle
IDs as constants.

## `code-templates-*.pnml` (logic switches)

| File | Role |
|---|---|
| `code-templates.pnml` | core misc: start/stop callbacks, tenders/articulation, default cargo classes, `allow_*` disable gates, `map_sprites`, `use_yeti` |
| `code-templates-attach.pnml` | `can_attach_wagon` rules for MU consist shapes; the name encodes topology (`h02ch` = head + 0..2 cars + head, `m2` = 2 middle types) |
| `code-templates-directions.pnml` | left/right cab orientation per unit, deterministic or seeded from `prev_vehicle_randombits()` |
| `code-templates-powers.pnml` | swaps `_sprites` ↔ `_notpowered_sprites` depending on power availability in the consist |
| `code-templates-effects.pnml` | visual FX: pantograph sparks, diesel exhaust, steam plumes (`engine_<type><count>[r]_..._effect`) |
| `code-templates-prop.pnml` | property callbacks — speed/power/capacity/cost varying by year, refit, position, current type |
| `code-templates-prop-rc.pnml` | the running-cost (fuel) model: `RC_RACE`, `RC_STOP`, `RC_head_check*`; design notes in the header |
| `code-templates-groups.pnml` | purchase-list grouping tree (`name_in_group*`, `group_props*`, `group_CBs`), honours `disable_*groups` params |
| `code-templates-strings.pnml` | purchase-hint / additional-text builders (`hint_MU*`, `hint_engine_in`, `hint_change_after*`, year-dependent `fact_*()` factory names) |
| `code-templates-lv.pnml` | the livery ID registry: `LV_*` numeric constants for `cargo_subtype`; `0xE0–0xFF` reserved for paid refits |
| `code-templates-lv-new.pnml` | the newer livery menu builder: `livery_subtemplate_long`, `livery_template_base_listN` |

Module-local counterparts follow the same conventions:
`src/<mod>/<mod>-code-templates*.pnml`, notably
`<mod>-code-templates-lv-pred.pnml` (pre-baked livery sets) and
`src/emu/emu-code-templates-calc.pnml` (consist calculations, also used by
subway).

## `graph-templates-*.pnml` (spritesets + selection trees)

These emit `spriteset`/`spritegroup` plus the switch tree selecting between
them, consuming `sN_template` slots.

| File | Role |
|---|---|
| `graph-templates.pnml` | purchase-menu sprites incl. electrification icon overlay (`purchase_sprites*`), generic cargo layout entry point |
| `graph-templates-locos.pnml` | locomotive bodies: `steam_sprites`, `tender_sprites`, `diesel1_sprites`, `electric1_2_sprites`, … |
| `graph-templates-mu.pnml` | EMU/DMU car bodies (`MU_head_sprites`, `EMU_motor1_sprites`); four sprite states: `_notpowered_`, `_normal_` (pantograph down), `_middle_` (lights off), plain |
| `graph-templates-mu-icons.pnml` | overlay icons on the last car showing what may still be attached |
| `graph-templates-wagons.pnml` | freight/passenger wagon bodies and load layers (`pass_wagon_sprites`, `tanker_layout_template`, `combo_layout_boxcar_template`) |
| `graph-templates-containers.pnml` | container stacks on platforms, fill thresholds from `cargo_count * 100 / cargo_capacity` |

## Other shared files

| File | Role |
|---|---|
| `src/cargotable.pnml` | master cargo label list (~150: vanilla + ECS + FIRS + YETI), sectioned with `// ---` banners that `scripts/Adjust2Master.pl` uses as sync anchors |
| `src/railtypetable.pnml` | railtype table `T_R*`/`T_A*`/`T_D*` with fallback chains ending in vanilla `RAIL`/`ELRL` — compatibility with other rail sets |
| `src/recolour.pnml` | company-colour recolour sprite bank (`ttdall_cc = reserve_sprites(215)`); source data in `docs/ral.xlsx` |
| `src/basecost.pnml` | one live line (`PR_BUILD_VEHICLE_WAGON: 2`); the rest is a superseded legacy scheme |
| `src/disable-origin.pnml` | disables all original TTD trains (combined build only) |
| `src/car-attach.pnml` | hand-rolled attach chain for mail cars 61-4504/05 |
| `src/dummy/` | invisible articulated spacers `dummy1..8` (IDs 116–123) + purchase icons + MU consist-preview icon system (`types.pnml`) |
| `src/override/` | the standalone stationratings mini-GRF; unrelated to vehicles |
