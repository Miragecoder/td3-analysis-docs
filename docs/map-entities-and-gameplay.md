# Map entities and gameplay

The visible tile grid is only part of a leg's behavior. The grid selects
reusable tile meshes; the map's entity table places the player, traffic, and
scenery; lane data supplies routes for AI; and special polygon surface
materials drive weather and finish events.

## What is stored in each map

The map begins its main entity table at `+0x0EE1`: three 16-bit pass counts,
then 13 parallel arrays of 160 words each beginning at `+0x0EE7`. The map
format notes identify the counts with collision, AI/behavior, and LOD/visibility
processing. The arrays hold initial entity flags, coordinates, orientation,
movement and cell/waypoint state, plus fields used or recomputed at runtime.
The map also contains a separate, less-understood entity pool before this
table.

One useful check against the shipped data: every SCENE01 leg begins with an
entity in slot 0 whose flags are `0x2001` (model `0x01`). That slot's position
and heading are present per leg. The map also has sprite-0 X/Z/Y words at
`+0x0761`, `+0x09E1`, and `+0x0C61`; in the shipped maps, X/Z match the position
of entity slot 0. Because the map is loaded directly into the game data
segment, these words are `DS_sprite_x/z/y`. `car_leg_reset()` initializes the
player's position from those sprite coordinates, while reading the heading
from object 0. Thus the main entity record and the separate sprite-0 start
coordinates both need updating when moving the player start.

For the experimental reversed Pacific course, racer headings and formation
offsets are adjusted by the difference between the start- and finish-road
rotations. That points racers along the new leg while keeping their relative
spacing. The stored heading word is rotated per byte, matching its paired
heading components. Leg 3 starts on an intersection; its rotation alone does
not identify the direction of travel, so the current transform leaves that
leg's racer heading and formation unchanged pending route verification. Racer
entity Y values and the separate player sprite Y value are also adjusted for
the elevation change at the new start.

The map header bytes `+0x18..+0x1C` should therefore not be treated as a
confirmed start-position field.

The `Test-Drive-3-Maps` viewer reads the first count as its record count and
can display object positions, but draws them as static markers. It does not run
the game's moving-entity logic, so a rendered object listing is not a
simulation of the leg.

## Weather and finish are surface events

`render_raster.c` derives the surface under the car from the polygon face
material/type (`DS_face_type >> 3`). The race loop responds to these surface
codes:

| Surface code | Effect |
|---|---|
| `0x1C` | Wind zone; ramps wind level. |
| `0x1D` | Rain zone; ramps rain and updates the color remap. |
| `0x1E` | Snow zone; ramps snow and updates the color remap. |
| `0x1F` | Finish; the race state advances. |

These are not a separate list of weather-trigger cells. The special material
is part of tile geometry, and the player's current surface changes as the car
crosses the relevant polygons. Each transition onto rain material `0x1D`
changes the rain level by 4, ramping between 0 and 8; ordinary road does not
itself clear that level. Snow material `0x1E` clears rain and changes the snow
level by 4, while wind material `0x1C` changes wind by 2 between 0 and 4.
These are trigger crossings/state steps, not continuous “rain is on while
inside this area” flags. `hud_topbar.c` independently searches tile geometry
for material `0x1F`, described in the source as the gas-station finish tile.

## Traffic routes and moving entities

AI routes are not described by the visible tile grid alone. `sim_traffic.c`
uses the map cell's tile ID to look up a lane entry in a `P.BIN` lane bank.
That entry contains waypoint records (flags, height, X, and Z); the cell's
rotation bits transform their local coordinates and exits into the placed
tile's orientation. In short, the tile identifies the route template, while
the lane bank provides the path geometry and connectivity.

`flow_race.c` loads a course-specific `<scene>P.BIN` when configured, or the
shared `SCENETTP.BIN` lane bank otherwise. The final byte of the scene LST's
10-byte leg-data field selects this: `0` loads the scene's own object/lane
banks, while nonzero keeps the shared banks. In the supplied files
`SCENE01.LST` and experimental `SCENE03.LST` select the shared bank;
`SCENE02.LST` selects `SCENE02P.BIN`, an archived resource at offset `0x1D11A`
inside `SCENE02.DAT`. No loose P.BIN is required. Lane waypoint flags also
carry control information used by traffic behavior, such as branches and
stopping controls. Map entity records supply the individual vehicles' model,
behavior, position, and current lane/waypoint state.

The two computer opponents have additional branch scripts in the map header.
The port labels their storage `DS_opp_scripts` at map offset `+0x05`: two
10-byte sequences, assigned to entity model IDs `0x02` and `0x03`. The first
byte is the initial behavior mode. Further bytes are consumed at marked
branch/control waypoints, not at every tile. Modes `2`, `3`, and `4` select
different branch strategies; a `0` ends that opponent's run. These scripts
select among paths, while the lane bank supplies the actual waypoint
coordinates and cell-to-cell connections.

The editor traces opponent waypoint sequences on both course layouts. It
detects each map's finish tile by its surface material, reads the scene LST's
lane-bank flag, extracts an archived `<scene>P.BIN` from its scene DAT when
required, and uses the shared bank otherwise. It follows branch scripts,
rotated cell connections, and the engine's same-cell U-turn fallback; it does
not simulate steering or vehicle timing.

Against the shared bank, both opponents reach the finish without a fallback
U-turn on all five original Pacific legs. On the inverted map, the trace
predicts a same-cell U-turn for both racers on leg 1; on legs 2–5 it finds no
such U-turn, but the route scripts end before those opponents reach the finish
(leg 2: one opponent; leg 3: both; leg 4: one; leg 5: both). The supplied
`SCENE03.LST` selects the shared bank, so a private `SCENE03P.BIN` is not
expected for this inversion. The user confirmed the workspace's
`TD3_SCENE03/SCENE03.DAT` is the latest playtested file. Comparing it with a
fresh output of the current inversion code reveals that its leg-3 start and
finish layout comes from an earlier transform; see
`course-inversion-feasibility.md`. Reviewing the rendered paths, the user
confirmed that the opponents turn back before the apparent
dead ends and then head in the correct direction on inverted legs 1, 2, 4, and
5. Leg 3's rendered route does not turn back; the user drove to leg 3 and
confirmed that its opponents immediately head in the correct direction. This
resolves the reported all-legs U-turn discrepancy. The available playtested
`SCENE03.DAT` differs from the current inverter's output in its leg-3 start
and finish layout, so the current export still needs its own in-game check
(details in `course-inversion-feasibility.md`). The trace's U-turn count is
narrower: it records only the engine's same-cell fallback when an exit has no
matching neighbor entry. A visual reversal along connected waypoints need not
use that fallback. The route trace still does not simulate steering or vehicle
timing.

The shared `SCENETTP.BIN` lane index embedded in `DATAB.DAT` contains 123
entries (`0x00`–`0x7A`); IDs `0x7B`–`0x7F` are not indexed in that shared bank.
The generated [lane-connectivity report](course-lane-connectivity.md) catalogs
the indexed tile ports and audits their rotated, elevation-aware links across
the original maps. All five inverted Pacific opponents have a valid first
neighbor entry with zero height delta in this shared bank, so the reported
U-turns are not explained by a missing first edge. Later waypoints, branch
choices, headings, or a scene-specific lane bank remain to be checked.

The five Pacific maps confirm the behavior you noticed:

| Pacific leg | Opponent 1 script | Opponent 2 script | Same branch script? |
|---|---|---|---|
| 1 | `2 → 3 → 2 → 0` | `3 → 2 → 2 → 2 → 0` | No |
| 2 | `2 → 4 → 2 → 0` | `3 → 2 → 2 → 0` | No |
| 3 | `2 → 2 → 3 → 2 → 0` | `2 → 2 → 3 → 2 → 0` | Yes |
| 4 | `2 → 3 → 2 → 0` | `4 → 2 → 2 → 2 → 2 → 0` | No |
| 5 | `2 → 2 → 2 → 2 → 0` | `3 → 3 → 2 → 0` | No |

These are behavior/branch codes, not route names or a complete drawn path.
The scene LST's named route choices belong to the player's route-selection
system and should not be confused with these two opponent scripts. The editor
extracts scene-specific `P.BIN` lane banks from the scene DAT archive when
available, but does not automatically generate or rewrite route scripts.
Cape Cod's `SCENE02P.BIN` is embedded in `SCENE02.DAT` and is resolved through
the LST archive directory. Its 127-entry index is now used for the Cape route
traces, and traces for the first three legs still terminate at script end before
reaching the detected finish:

| Cape leg | Opponent 1 trace endpoint | Opponent 2 trace endpoint |
|---|---|---|
| 1 | script ended after 214 steps at (3,1), tile `0x68` | script ended after 174 steps at (19,1), tile `0x44` |
| 2 | script ended after 214 steps at (2,1), tile `0x15` | script ended after 199 steps at (2,1), tile `0x15` |
| 3 | script ended after 229 steps at (1,1), tile `0x15` | script ended after 208 steps at (1,1), tile `0x15` |

These are trace results, not a confirmed report of the in-game routes. They
leave the mismatch with the expected finish-reaching routes on Cape Cod
unresolved.

The high byte of an entity's flags word includes a behavior value. In
`sim_traffic.c`, behaviors below 2 and behavior 9 use shuttle motion: the
entity reverses at a waypoint endpoint. Other values select different traffic
behaviors, including branch choices and a slower train behavior. Train models
`0x12`–`0x14` use the same lane-following routine; behavior 8 halves their
speed. Thus trains and cars share the waypoint-based movement mechanism,
although a particular railroad tile's route still depends on the selected lane
bank and placed train entity. Back-and-forth motion is AI behavior attached to
an entity following its lane, not an animation inferred from its tile alone.

## Trees and tile-attached sprites

Trees and roadside details can be attached to a tile model as child sprites.
In `render_world.c`, `model_place()` reads a child count after the tile's
vertices and faces, then reads each child's sprite ID and local X/Z/Y
position. When the map cell places that tile, the game rotates those local
positions using the cell's rotation bits and adds the tile's world position.
So the map cell chooses the tile, while the tile model carries the child
sprite placements.

The child list is not necessarily a fixed set of visible trees: the game
uses a vegetation seed to probabilistically omit children whose sprite IDs
are `0x12` or higher. There are also separate entity-table records for
independently placed 3D scenery. A complete placement view therefore needs
both the entity table and the tile models' child-sprite data.

The current map viewer/editor parses tile geometry but skips the child-sprite
records, and its top-down tile thumbnails show only that geometry. This is why
some vegetation may be absent from the editor preview even though it appears
in-game. The engine's tile-child format is clear enough to extract placement
IDs and local coordinates; matching every ID to the correct per-leg sprite
image still needs the corresponding sprite resources.

## Birds and the chicken-like objects

The source has a collision special case for model `0x39` when it hits face
type `0x1B`. The handler changes the entity flags to `0x3039` and clears its
velocity and waypoint state; its comment calls the event “seagull takes off.”
The object-bank notes also tentatively label nearby entries as birds. This
could be the moving bird/chicken behavior seen in-game, but the model identity
is not conclusively matched, and the collision handler itself does not show an
explicit vertical-flip operation. That visual detail remains open.

## Police

`sim_traffic.c` identifies police meshes by model IDs `0x11` and `0x30`.
Parked police can be awakened when the player passes at sufficient speed; the
police entity is switched into pursuit behavior, with a U-turn when needed.
This matches the stationary-police-then-chase pattern, although the exact
thresholds and pursuit state are game logic rather than placement data.

One port-specific caveat: the source documents an intentional change from the
original game. The original starts pursuit when the police are close while
still ahead; this enhanced port waits until the player has passed them. Keep
that distinction in mind when treating the port as a literal specification of
the 1990 executable.

## What is known and what remains open

- **Confirmed:** special material codes drive wind, rain, snow, and finish
  events; AI waypoints come from lane banks selected by tile ID and rotated
  with each map cell; map entity records contain placed and moving entities;
  Pacific's opponent branch scripts are identical only on leg 3.
- **Strongly supported:** entity slot 0 is the per-leg player start record.
  Its coordinates are present in every sampled SCENE01 map, and its heading
  is explicitly used by `car_leg_reset()`. The coordinate-copy path still
  needs tracing.
- **Unresolved:** whether model `0x39` is the user's chicken-like object and
  which rendering/state mechanism produces the reported vertical flip.
- **Port caveat:** police pursuit timing differs intentionally from the
  original game.

## References

- `Test-Drive-3-Maps/spec/map-format.md`: map header, entity arrays, and
  secondary entity pool.
- `Test-Drive-3-Maps/src/browser/main.ts` and `src/shared/mapgen.ts`: current
  static object visualization.
- `Test-Drive-3-Maps/src/shared/extract.ts`: tile geometry extraction currently
  skips the sprite-child records.
- `testdrive3-enhanced/src/game/flow_race.c`: leg setup, reset, map loading,
  weather transitions, and lane-bank selection.
- `testdrive3-enhanced/src/game/render_world.c` and `render_sprite.c`:
  tile-attached sprite children and per-frame billboard rendering.
- `testdrive3-enhanced/src/game/render_raster.c`: surface type under the car.
- `testdrive3-enhanced/src/game/hud_topbar.c`: finish-material lookup.
- `testdrive3-enhanced/src/game/sim_traffic.c`: lane lookup, waypoint motion,
  opponent branch scripts, shuttle behavior, and police pursuit.
- `testdrive3-enhanced/src/game/sim_collide.c`: special model `0x39` collision.
