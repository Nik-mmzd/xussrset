# Wagon cargo system

Root: `src/wagons/`. Wagon bodies live in per-type dirs (`boxcars/`,
`gondolas/`, `flatbeds/`, `hoppers/`, `tankers/`, `refrigerators/`,
`special/`, `vehicles/`); shared cargo logic and load graphics live in the
`cargoes*` files described below.

## `cargoesN/` directories: N is the body length

**`N` is the sprite-template size class (the `N` of `sN_template`), not a
generation.** A 9-length wagon uses `cargo9_*` load sprites from `cargoes9/`.
Present: `cargoes5` … `cargoes14`; 5/6/8/9 have the fullest coverage. The
hybrid `cargoes9/cargoes7_9/` holds 7-length loads drawn on 9-length bodies.
`container/` subdirs exist under `cargoes8`, `cargoes9`, `cargoes12`. Within a
directory the split is by body type: `cargoes.pnml`, `cargoes-gondola.pnml`,
`cargoes-flatbed.pnml`, `cargoes-crates.pnml`, `container/containers.pnml`.

## Cargo taxonomy — `cargoes-all.pnml`

Two macro families:

- `#define CARGO_<LABEL> 0x<palette>` — recolour index for bulk loads;
- `cargo_all_<group>list()` / `cargo_check_<group>list(function)` — group
  membership lists spliced into switches. Groups: clays/ores/generic bulk,
  wood, steel, piece goods, large boxes, grain hopper, containers, and tanker
  sub-classes (oil / petrol / food / chem general / chem dangerous /
  chem heated / gas). The taxonomy diagram is `docs/cargoes.vsd` /
  `docs/tankers.vsd`.

## The physical model: temp registers + density

A wagon stores its physical envelope in temp registers, then chains into a
shared per-body-type switch:

```
Register 0 — carrying capacity, t
Register 1 — volume, m³
Register 2 — floor area, m²
Register 3 — pallet slots
Register 4 — length (template N)
```

### Capacity — `cargoes-capacity.pnml` + `-box/-gondola/-flatbed/-hopper/-ref/-tank/-cont`

One giant switch per body type over `cargo_type_in_veh` returns
`calculate_capacity(LOAD_TEMP(0), LOAD_TEMP(1), <density_kg_m3>, cargo_unit_weight)`
— capacity derives from real cargo density (sources cited in the file
headers). Labels not handled by a body type are kept as `// LABEL,` comment
lines preserving `cargotable.pnml` order — that ordering is maintained by
`scripts/Adjust2Master.pl` (`ct2cl` mode). Container capacity re-stores the
registers scaled by TEU count and chains into the box/ref/tank switches.

### Load speed — `cargoes-loadspeed.pnml` + per-body variants

Registers: 0/1 = ticks to load/unload (drawn stages), 2/3 = door/hatch
throughput per tick. Values built from `definition.pnml` constants
(`NORMAL_HATCH`, `WAGON_DOOR`, `MEDIUM_FALL`, …).

### Refit cost — `cargoes-refit.pnml`

Cleaning cost on cargo-class change: same class → free autorefit
(`return 0 | CB_RESULT_AUTOREFIT`), otherwise a cost of 1–7 depending on how
dirty the previous class was.

### Weight and ageing — `cargoes-weight.pnml`, `cargoes-cap-cont.pnml`

Container tare/gross weights (era-scaled via `date_of_last_service`), and
cargo ageing periods for containerised cargo.

## Load graphics — `cargoes-open-templates.pnml`

For open wagons (gondolas, flatbeds). Macros named `wagon_cargoV_S` where
V = number of random visual variants and S = number of fill stages
(`1_1`, `1_3`, `2_5`, `5_3`, `5_5`). The fill-stage threshold is computed from
cargo **density**, so equal tonnage of coal and iron ore shows different pile
heights.

## Foreign-GRF interplay

`src/check.pnml` sets flags (`yeti_on`, `ecs_chem_ii_on`, `ecs_mach_on`,
`otis_on`) that the capacity files use to handle cargo labels defined by those
sets (YETI LVST, ECS OIL_/PETR/RFPR densities, VEHI, …).
