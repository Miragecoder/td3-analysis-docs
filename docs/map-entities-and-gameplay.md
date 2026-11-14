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
is present per leg, and its heading is stored there. `car_leg_reset()` reads
object 0's heading, and resets the car using sprite coordinates already
present in runtime state. This makes slot 0 the strongly supported player-start
record; the precise loader step that copies its coordinates into those sprite
fields remains to be traced.

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
shared `SCENETTP.BIN` lane bank otherwise. Lane waypoint flags also carry
control information used by traffic behavior, such as branches and stopping
controls. Map entity records supply the individual vehicles' model, behavior,
position, and current lane/waypoint state.

The two computer opponents have additional branch scripts in the map header.
The port labels their storage `DS_opp_scripts` at map offset `+0x05`: two
10-byte sequences, assigned to entity model IDs `0x02` and `0x03`. The first
byte is the initial behavior mode. Further bytes are consumed at marked
branch/control waypoints, not at every tile. Modes `2`, `3`, and `4` select
different branch strategies; a `0` ends that opponent's run. These scripts
select among paths, while the lane bank supplies the actual waypoint
coordinates and cell-to-cell connections.

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
system and should not be confused with these two opponent scripts. The current
browser viewer does not load `P.BIN`, so it can inspect the scripts but cannot
yet draw the opponents' full road paths.

The high byte of an entity's flags word includes a behavior value. In
`sim_traffic.c`, behaviors below 2 and behavior 9 use shuttle motion: the
entity reverses at a waypoint endpoint. Other values select different traffic
behaviors, including branch choices and a slower train behavior. Thus the
back-and-forth motion is AI behavior attached to an entity following its lane,
not an animation inferred from its tile alone.

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
