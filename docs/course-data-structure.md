# Course data structure

## Course, leg, and route levels

The two supplied courses are `SCENE01` (Pacific Coast) and `SCENE02` (Cape Cod).
Each course has a scene index file (`SCENE01.LST` / `SCENE02.LST`) and a data
archive (`SCENE01.DAT` / `SCENE02.DAT`). The LST is a directory/configuration
file, not the 3D world itself.

Each scene LST is 1,638 bytes. It stores the course name, a leg/map count
(5 in both files), per-leg configuration and labels, route names, default
route-selection values, and a resource directory. The first 20 route-name
entries are 5 legs × 4 route choices; the remaining entries are unused
placeholders according to the current LST specification.

At race setup, `flow_race.c` loads the selected leg's map resource using a
name of the form `SCENE01A.DAT` through `SCENE01E.DAT` (or `SCENE02A.DAT`
through `SCENE02E.DAT`). These are resources indexed within the course's DAT
archive, not five separate top-level files. `flow_files.c` reads the leg count
and route metadata from the LST, while `leg_map_load()` in `flow_race.c` loads
the selected map into the game's data segment.

## One map/leg blob

Each map is `0x2137` bytes (8,503 bytes). Its principal regions are:

| Map offset | Size | Meaning |
|---|---:|---|
| `0x0000–0x00DE` | `0xDF` | Header/parameters; includes sky/color parameters and the day/dark color-mode byte. Many header bytes remain unknown. |
| `0x00DF–0x04DE` | `0x400` | 32 × 16 grid, with 2 bytes per tile. |
| `0x04DF–0x0EE0` | `0xA02` | Secondary entity pool; some chunks are used for entity type/state and coordinates, other chunks are not yet understood. |
| `0x0EE1–0x0EE6` | 6 | Three word counts used by collision, AI/behavior, and LOD/visibility passes. |
| `0x0EE7–0x1F26` | `0x1040` | 13 parallel arrays of 160 words each, describing placed/runtime entities and their fields. |
| `0x1F27–0x2126` | `0x200` | 256-entry, 16-bit color/material remap table. |
| `0x2127–0x2136` | `0x10` | Secondary 16-byte remap table. |

### Tile grid

Each 2-byte cell is encoded as:

```text
byte 0: tile ID
byte 1: bits 7..6 = quarter-turn rotation; bits 5..0 = elevation
```

The data-format project's current reconstruction uses a 4,096-unit spacing
between cells and interprets elevation as `height × 256`; the spacing is
supported by disassembly, while the exact elevation scale is still a tooling
interpretation. Its current tile-bank split is IDs `0x00–0x3F` from the shared
tile set and `0x40–0xFF` from the course-local set (subtract `0x40` for the
local index). The format notes caution that the original engine resolves IDs
through descriptor tables rather than a proven inline range test.

The entity arrays after the grid complement the repeated tile geometry with
placed objects and moving/behavioral state. The map also has a separate
secondary entity pool. They are not just a tile bitmap.

The main entity table appears to contain the player start record, moving
traffic, and props; tile-selected lane templates supply AI routes, while
special tile surface materials trigger weather and the finish. See
[Map entities and gameplay](map-entities-and-gameplay.md) for the runtime
evidence and remaining uncertainties.

## Shared and course-specific assets

Both courses reuse meshes from banks instead of embedding full mesh copies in
each map:

| Asset | Pacific Coast (`SCENE01`) | Cape Cod (`SCENE02`) |
|---|---|---|
| Scene tile meshes | 59-entry bank in `SCENE01.DAT`, shared by its five maps. | 64-entry bank in `SCENE02.DAT`, shared by its five maps. |
| Shared tile meshes | 64-entry bank in `DATAB.DAT`. | Same shared bank in `DATAB.DAT`. |
| 3D scene-object meshes | Uses the shared object bank in `DATAB.DAT` (`SCENETT0.BIN`). | Has a 64-entry scene-object bank in `SCENE02.DAT`. |
| 2D billboard sprite bank | 128 sprites catalogued in `SCENE01.DAT`. | 110 sprites catalogued in `SCENE02.DAT`. |
| Backdrops/palettes | Three sub-scene palettes and three ground/sky strip pairs shared across the five maps. | Same arrangement: three palettes and three ground/sky strip pairs. |

The scene-object banks contain varied scenery and vehicle objects, including
traffic/police-related meshes; those are distinct from the selectable car
models. The player and opponent cars are loaded separately from per-car
`.POB` files chosen through the PLAYDISK/car-slot data. The game source loads
the course tile bank (`<scene>T.BIN`) and, when the course requires it, its
own object/lane banks (`<scene>O.BIN` / `<scene>P.BIN`); otherwise it uses
shared sets. Thus “the course has its own traffic cars” is broadly right for
Cape Cod's scene bank, but Pacific Coast's object bank is shared.

Trees and other roadside items can be billboard sprites rather than 3D
meshes. The asset inventory counts the sprite banks per course; the game also
selects sprite/resource data during each leg load via the LST resource
directory.

## Day/night (dark/light) data

The strongest runtime link is map byte `0x35`: the C port loads each map at
`DS:9592`, and `DS_colour_mode` is at `DS:95C7` (`map_base + 0x35`). Rendering
code treats mode `0` as dark/night and nonzero as day/light; the enhanced
renderer makes the same distinction. The map-format notes also identify this
byte as controlling a color-table darkening pass.

Reading the ten shipped map blobs confirms that this value varies per leg:

| Course | Leg 1 | Leg 2 | Leg 3 | Leg 4 | Leg 5 |
|---|---:|---:|---:|---:|---:|
| Pacific Coast (`SCENE01`) | 0 | 1 | 1 | 0 | 0 |
| Cape Cod (`SCENE02`) | 1 | 0 | 0 | 1 | 1 |

Here `0` is the game's dark/night mode and `1` its day/light mode. This is
consistent with your understanding that the map data carries a per-leg
setting, with the qualification that it is expressed as a rendering/color
mode rather than a clearly named standalone “night” structure.

The adjacent byte at map offset `0x36` is a separate field. The format notes
associate it with choosing the sky-base seed for a sky/horizon lookup. It
should not be confused with the `0x35` color-mode byte; `0x36` is zero in all
ten sampled on-disk maps, and its full runtime role is less settled.

### Is Pacific leg 5 darker?

It does not have a separate, deeper night-mode flag: legs 1, 4, and 5 all
have `map[0x35] = 0`, and their night sky seed at `map[0x2F]` is `0x08`.
However, leg 5 is not a byte-for-byte copy of the other dark legs:

- The sky/fog remap entry at index `0x23` is `0x6767` for leg 1, `0x6666`
  for leg 4, and `0x6565` for leg 5. In dark mode the engine's remap pass
  lowers these pairs by `0x0202`, giving `0x6565`, `0x6464`, and `0x6363`.
- The scene LST's palette suffixes are `1, 3, 3, 3, 5` for legs 1–5, so
  leg 5 uses its own palette/backdrop group (palette C), unlike leg 4
  (palette B). Thus a one-step-lower palette index does not by itself prove a
  darker rendered color. In the supplied data, the raw palette C is not
  globally darker than palette B; color usage and the landscape strips also
  affect the impression.

So a slightly different sky/fog treatment is present, but the data does not
show leg 5 using a distinct “extra-dark night” mode. A direct visual comparison
in the game would be needed to settle the subjective impression.

## Generated `SCENE03` day/night-flipped copy

A test copy has been generated at `TD3_SCENE03` in the workspace; the supplied
`TD3` directory remains unchanged. It contains a copy of the original game-data
folder plus:

- `SCENE03.DAT` and `SCENE03.LST`, cloned from `SCENE01`; all 29 scene-resource
  hashes in the LST directory now use the `SCENE03` prefix.
- The five map mode bytes at `map_base + 0x35` inverted from `0, 1, 1, 0, 0`
  to `1, 0, 0, 1, 1`. The sky-color seeds at `map_base + 0x2E..0x31` are also
  set to the engine's night values (`09 08 72 13`) or day values
  (`0B 07 79 19`) to match each flipped mode; the per-map gradient seed at
  `+0x32` is preserved. All other DAT bytes are unchanged.
- A `SCENE03` entry in PLAYDISK's third scene slot and an updated active scene
  count of three.

No engine changes are needed: the enhanced port's `--course` selection uses
the PLAYDISK slots. The map mode byte does not itself choose the sky seeds,
which explains why the first playtest showed no sky-color change. The seed
values follow the existing day/night toggle in `sim_controls.c`.
`TD3_SCENE03/README.md` has usage notes. File sizes, resource hashes,
slot/count fields, and changed map bytes were checked; the updated package
still needs a runtime check.

## References and confidence

- `Test-Drive-3-Maps/spec/map-format.md`: map layout, tile cells, entity pools,
  and map-header field analysis.
- `Test-Drive-3-Maps/spec/dat-file-layouts.md`: course archive offsets, mesh
  banks, sprite banks, palette/backdrop groups.
- `Test-Drive-3-Maps/spec/lst-file-format.md`: scene LST and route structure.
- `testdrive3-enhanced/src/game/flow_files.c`: LST loading and scene archive
  lookup; `src/game/flow_race.c`: per-leg resource and map loading.
- `testdrive3-enhanced/src/game/render.c`: mode `0` dark-leg color remap;
  `src/enhanced/enhanced.c`: enhanced renderer's day/dark interpretation.
- `testdrive3-enhanced/src/game/sim_traffic.c`,
  `sim_collide.c`, `render_raster.c`, and `hud_topbar.c`: map-entity, lane,
  surface-trigger, collision, police, and finish behavior.
- `TD3/SCENE01.DAT`, `SCENE01.LST`, and `PLAYDISK.DAT`: supplied original data;
  the LST's scene palette suffixes and available scene slots were checked.

The map format is mostly confirmed, but parts of its header and secondary
entity block remain empirical or unresolved; the tooling's tile-height
interpretation is not an authoritative engine specification.
