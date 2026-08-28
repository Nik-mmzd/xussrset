# Adding or modifying a vehicle

Reference example used throughout: `src/emu/rvz/er1_2/er2_h.pnml` (ER2 EMU head
car). Copy the nearest existing sibling of the same traction type instead of
starting blank — macro combinations are not interchangeable across traction
types.

## Where files live

```
src/<module>/<manufacturer>/<name>.pnml   # vehicle code
src/<module>/<manufacturer>/<name>.png    # sprite sheet, next to the pnml
```

Vehicles without a manufacturer subdir sit flat in the module root
(`src/emu/es1.pnml`, `src/steam/l.pnml`). Family subdirs may nest one level
deeper (`src/emu/rvz/er1_2/`). All filenames are lowercase (enforced by
`scripts/clean-lng.pl`, which renames offenders).

### Filename suffix conventions

| Suffix | Meaning |
|---|---|
| `<name>-partN.png` | sprite sheet split across several PNGs |
| `<name>_use.pnml` | reuses another vehicle's sprites, has no own PNG |
| `<name>-group.pnml` | `variant_group` parent item shown in the purchase list |
| `<name>-subgroup-<factory>.pnml` | second-level variant group |
| `<name>-pre.pnml` | prototype / pre-series unit |
| `<name>-type<YEAR>.pnml` | build-year variant of a series |
| `_h / _m / _c / _hb / _hmp / …` | MU car roles: head / motor / trailer / head-buffet / head-motor-pantograph |

## Vehicle file skeleton

Every vehicle file follows the same fixed sequence:

### 1. Property constants

```pnml
#define PROP_er2_h_CF  2 * 10   // cost factor
#define PROP_er2_h_RC  78       // running cost
#define PROP_er2_h_SD  130      // max speed, km/h
#define PROP_er2_h_WT  40.9     // weight, t
#define PROP_er2_h_TE  0        // tractive effort
#define PROP_er2_h_PR  0        // power
#define PROP_er2_h_CC  84       // cargo capacity
```

Strictly `PROP_<id>_<CF|RC|SD|WT|TE|PR|CC>`; wagons add `_LC/_AC/_VC`
(load/area/volume capacity), coaches `_FC`. The same symbols feed the
`property {}` block, `purchase_menu()` and the running-cost switch — change one
`#define` and everything follows. The numbers themselves come from the design
spreadsheet `docs/xUSSR Set.xlsx`.

### 2. Sprite sheets

```pnml
#define IMAGEFILE  "src/emu/rvz/er1_2/er2_h-part1.png"
purchase_sprites_with_icon(er2_h_v1, 18, 0, 3dc)
MU_head_sprites(12, er2_h_v1_mu, 32, 40)
#undef IMAGEFILE
```

- `IMAGEFILE` paths are **repo-root-relative**, even though the PNG sits next
  to the pnml. Always `#define` / `#undef` in pairs, one pair per PNG —
  `scripts/clean-lng.pl` scrapes exactly this form.
- Sheet geometry must match the blank artist template in
  `src/align/templates/` for the macro being called (`mu_head.png`,
  `steamer.png`, `diesel1.png`, `wagon-tanker.png`, …). MU sheets stack
  268-px paint bands at y = 40 / 308 / 576 / 844 (purchase strip at y 0–17).
- Inside each sprite cell, align new art with the SAME family's existing
  sheets, not with other families: ground-row conventions differ per family
  (e.g. Ivolga cars sit at cell rows 18–39 while ЭД4М sits at 19–40), and a
  1-px offset against the vehicle's own siblings renders as a visible step
  between cars of one consist.
- The first macro argument of body templates is the vehicle length `n`, mapped
  to `sN_template` slots from `src/template.pnml`.

### 3. Livery / year switch chains

Two-level chain: `cargo_subtype` (livery, `LV_*` constants from
`src/code-templates-lv.pnml`) selects a livery, then `check_year(...)` selects
the era variant. Era constants (`GREAT_CHANGE_YEAR`, `USSREND`, `PID_YEAR`, …)
live in `src/definition.pnml`.

**The default branch is always `align_<n>_sprites`** — the magenta alignment
grid from `src/align/align.pnml`. An unhandled case renders as a visible
checkerboard instead of silently falling back.

### 4. Direction / articulation / consist logic

`engine_direction_template*`, `MU_attach_wagon_icon_template*`,
`long_vehicle*`, `EMU_*_can_attach_wagon_*`, `EMU_attach_calculation_*` — see
[templates.md](templates.md). These encode which car may follow which and draw
the consist-preview icons from `src/dummy/types.pnml`.

### 5. Livery list, capacity, running cost

`livery_template_base_listN(...)`, `engine_capacity_*`, `RC_head_check*` plus a
hand-written `running_cost_factor` switch that stores cost components
(engines/crew/wear/maintenance/…) in temp registers. The trailing comment with
the computed result must equal `PROP_<id>_RC`.

### 6. Name callback

`name_in_group_subgroup(id, ..., string(STR_NAME_...))` — variants in
`src/code-templates-groups.pnml`, honouring the `disable_groups` /
`disable_long_names` params.

### 7. Purchase-window hint

`hint_MU(...)` / `hint_engine_in(...)` / `hint_wagon_coach(...)` from
`src/code-templates-strings.pnml`, plus `fact_*()` macros that return a
year-dependent factory name.

### 8. The `item` block

```pnml
item (FEAT_TRAINS, er2_h, 222) {
  property {
    name: string(STR_NAME_ER2_TYPE1962_H);
    vehicle_dates(1962, 1974, 30, 10, 8, PROP_er2_h_CF)
    vehicle_emu_c(dc, PROP_er2_h_WT, PROP_er2_h_CC, 2 * DOUBLE_DOOR, )
    vehicle_group(group_er2)
  }
  graphics {
    purchase_menu(PROP_er2_h_CF, PROP_er2_h_RC, PROP_er2_h_SD, PROP_er2_h_WT, PROP_er2_h_TE, PROP_er2_h_PR, PROP_er2_h_CC)
    additional_text: er2_h_additional_text;
    ...
  }
}
```

The numeric ID comes from `src/IDs_usage` (next free slot; see
[strings-and-ids.md](strings-and-ids.md)). `graphics {}` keys are
alphabetically sorted and colon-aligned — that is the house style. Property
macro families (`vehicle_dates`, `vehicle_steam`, `vehicle_emu_c`,
`vehicle_wagon`, `vehicle_group*`, `purchase_menu*`) live in
`src/definition.pnml`.

### 9. Tail

```pnml
long_name_template(er2_h, STR_LONGNAME_ER2_TYPE1962_H)
allow_dcemu(er2_h)
```

Both re-open the item: the first overrides `name` when long names are enabled,
the second disables the vehicle when its `disable_*` param bit is set. Wagons
and coaches also call `models_default_cargo_template_*`.

### 10. Group item (separate file)

A series gets a `<series>-group.pnml` with its own numeric ID:
`group_props*` + an `item` with `group_<traction>(...)` and `group_CBs(...)`.
Group files must be included **after** all member vehicles in
`<mod>-list.pnml`.

## Checklist

1. Draw the sprite sheet against the matching `src/align/templates/*.png`;
   save as lowercase `src/<module>/<factory>/<name>.png`.
2. Create the `.pnml` next to it, following the skeleton above (copy the
   nearest sibling of the same traction type).
3. Take the next free numeric ID from `src/IDs_usage` (empty right-hand side =
   free; if there are no gaps, use max+1).
4. Add `STR_NAME_<ID>` + `STR_LONGNAME_<ID>` to `lang/english.lng` **and**
   `lang/russian.lng`, in the correct banner section, colon-aligned.
5. `#include` the file in `src/<module>/<module>-list.pnml`, before the
   series' `-group.pnml`.
6. Add the identifier to `src/<module>/<module>-sort-order.pnml` at the right
   place in the series.
7. New series → also create `<series>-group.pnml` and reference it via
   `vehicle_group(...)`.
8. Run `perl scripts/clean-lng.pl` from the repo root: regenerates the
   `*_usage` registries, catches ID collisions, orphan PNGs and missing
   strings. Commit the regenerated files.
9. Build: `./compile.sh <module>` and `./compile.sh combined`; output lands in
   `build/`.
