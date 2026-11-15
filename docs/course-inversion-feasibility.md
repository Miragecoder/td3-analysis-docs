# Reverse-course feasibility

## Proposed transformation

For each destination leg, use source legs in reverse order (`5 → 1`, `4 → 2`,
`3 → 3`, `2 → 4`, `1 → 5`), invert the day/night setting, rotate the tilemap
180 degrees, move the racers from the old finish to the new start, and place a
finish at the old start.

This combines several distinct data transformations. Reversing the list of
legs is not the same as reversing the direction of traffic within a leg.

## Confidence by part

| Part | Confidence | Evidence / remaining work |
|---|---|---|
| Reverse the five map resources and update day/night | High for the binary transform; sky tuning remains experimental | Map offsets and mode byte are known. Inversion applies reversible per-byte offsets to the four sky-color values using the engine's debug-toggle presets; the resulting opposite-mode sky still needs an in-game visual check. |
| Rotate the grid 180 degrees | High for the visible terrain | Move each cell `(x, y)` to `(31-x, 15-y)`, add two quarter-turns to its stored rotation, and preserve elevation. Tile-attached sprite children inherit the transformed placement. |
| Rotate placed entities | Medium-high | The entity arrays and world coordinates are known. Non-racer entity headings rotate by 180 degrees with the map. Racer headings are adjusted by the difference between the start- and finish-road rotations; the leg-3 intersection is an exception because its rotation does not encode the approach direction. Counts, parallel fields, cell indices, and object-state bits must all be preserved. |
| Move the player/opponent starts | High for position and elevation; medium for direction | The runtime player position comes from the map's sprite-0 X/Z/Y arrays at `+0x0761`, `+0x09E1`, and `+0x0C61`; `car_leg_reset()` consumes those fields. The inversion moves these and the racer entity records as a group, rotates their formation with the endpoint road alignment, and adjusts racer height for the destination elevation. The player and opponents still need an in-game check. |
| Relocate the finish | Medium-high | Finish is surface material `0x1F` on tile geometry, not a standalone map marker. The inversion exchanges the start/finish tile IDs while preserving each destination cell's rotated road orientation and elevation. Leg 3's finish moves to the adjacent road cell beyond the preserved intersection; this placement still needs an in-game check. |
| Make AI drive the reversed direction | Low until lane data is decoded | AI uses `P.BIN` waypoint networks and per-opponent branch scripts. Rotating cells and moving starts alone does not reverse waypoint order or guarantee the lane exits connect in the desired direction. |

## Current editor experiment

The editor's Pacific inversion creates a modified copy of the DAT: it reverses
the five map resources, rotates each grid and its placed entities, swaps the
start/finish locations, and flips each map's lighting mode. The four sky-color
bytes are adjusted by the difference between the engine's night preset
(`09 08 72 13`) and day preset (`0B 07 79 19`). Applying the opposite delta
when toggling back preserves each map's deviations from those presets instead
of overwriting them with one fixed set of values. When exchanging the start
and finish tiles, their IDs move while each destination keeps its own rotated
orientation and elevation. The rotation belongs to the road at that map
location, not to the start/finish role being exchanged. On leg 3, the source
start is an intersection, so the finish moves to the rotated counterpart of
the ground-road tile `0x30` at source cell `(31,13)`. This leaves the
intersection at `(31,2)`, preserves the overpass tile at `(30,2)`, and puts
the finish tile at `(0,2)`, one cell beyond the intersection on the other
road exit. The old finish's ground-road approach tile is reused at the new
start, keeping the cell-ID changes reversible.

The new-start tile `0x30` has only three waypoint records, unlike the usual
four-waypoint straight road. The two opponent waypoint states are therefore
changed from `0x43` (index 3, out of range for tile `0x30`) to `0x40` (index 0)
on this inverted leg. Applying the transform again restores the original
`0x43` state. The first exit from waypoint 0 reaches waypoint 2 and has a
matching entry on tile `0x58` with zero height delta in the shared lane bank.

Non-racer entity overlays use the map's cell-center coordinates, and inversion
reflects their coordinates across the full map period. This keeps placed
entities aligned with the rotated tile grid rather than shifting them by half
a cell or one whole cell. The player's startup position is also stored in
sprite-0's X/Z/Y arrays at map offsets `+0x0761`, `+0x09E1`, and `+0x0C61`.
The game loads these directly into `DS_sprite_x/z/y`; `car_leg_reset()` uses
them to initialize the car. The inversion relocates these runtime start
coordinates and the main racer entity records, adjusts their Y values for the
destination terrain elevation, and rotates their offsets by the difference
between the source start- and finish-road orientations. Racer headings receive
the corresponding quarter-turn in each byte of the stored heading word. The
leg-3 start is an intersection, whose rotation does not identify which exit the
race takes, so its heading and formation are left unrotated pending route
verification. The player's formation remains behind the opponents, but the
AI's waypoint sequence and route behavior are not reversed; only the
stage-3 start index is corrected for the new tile's shorter lane record.

The reported black band is on inverted stage 5, which uses original leg 1's
map. `flow_race()` applies the engine's `0x0202` darkening pass to all
remap-table entries only when the loaded map's mode byte at `+0x35` is zero;
the inverted daytime stage skips that pass. The sky/fog seed remains `0x0203`,
which selects remap entry `0x23`. Original leg 1 stores `0x6767` there, while
the daytime Pacific maps 2 and 3 store `0x6868`. The inverter now moves that
selected pair one palette step (`0x0101`) toward the target mode on each
day/night flip. This makes inverted stage 5's horizon pair `0x6868`; flipping
back subtracts the same step, preserving byte-exact reversibility. An
in-game check is still needed to confirm that this removes the black band.

The transformation passes focused tests and the editor build; applying it
twice to the supplied Pacific `SCENE01.DAT` restores every byte. On leg 5, the
old start cell is at elevation 1 while the finish cell and its neighboring
road are at elevation 5. The first playtest showed that the opponents were
missing on the reversed stage; their entity Y values were still at the old
elevation. The inversion now applies the same elevation delta used for the
player, scaled to the entity-Y units. The generated stage-1 racers now have
their Y values raised by 8192 for the four-level change. This is a data-level
check, not an in-game confirmation.

The latest playtest also reports immediate AI U-turns on all inverted stages.
The heading and formation adjustment cannot by itself reverse waypoint order
or repair lane exits; the route remains unresolved. For stage 3, the finish is
now placed on the ground-road exit beyond the preserved intersection, rather
than on the overpass. The exact finish side and drive-through behavior still
need an in-game check. The stage-5 horizon remap pair is generated as `0x6868`,
but its visual correction also needs an in-game check. None of these
data-level checks establishes complete visual correctness or playability.

## Why AI is the main risk

Each tile ID selects a lane template from a `P.BIN` bank. The shared bank
embedded in `DATAB.DAT` has 123 index entries (`0x00`–`0x7A`); map tile IDs
`0x7B`–`0x7F` are not indexed there. Waypoints carry local coordinates and
exit flags, and map-cell rotation transforms their positions and exits. The
two opponents also have map-header branch scripts; their scripts differ on
Pacific legs 1, 2, 4, and 5, and match on leg 3. Those scripts choose branch
modes at certain lane markers, not direct world coordinates.

A 180-degree grid transform rotates the lane templates' geometry, but it does
not by itself reverse the sequence of waypoints. A playable reverse course
needs either proof that the rotated lane network is traversable from the new
start, or a reversed/adjusted lane network and matching opponent branch
scripts. The shared-bank audit finds that all five inverted Pacific opponent
starts have a matching first exit with zero height delta; the first-edge
same-cell U-turn fallback therefore does not explain the reported U-turns
under this bank. This does not rule out a later unmatched edge, route-script
branch, heading behavior, or a difference in the tested map data. The editor
now traces opponent routes from their stored start waypoints to the finish
tile, detected from its surface material. It follows the scene LST's bank
selection and loads `<scene>P.BIN` when available. Both opponents reach the
finish on all five original Pacific legs. The supplied `SCENE03.LST` selects
the shared bank, so this inverted experiment does not require
`SCENE03P.BIN`. The inverted trace predicts a same-cell U-turn on leg 1; the
remaining legs have no same-cell fallback, but the rendered route paths turn
back on legs 2, 4, and 5 before continuing in the correct direction. The user
confirmed that leg 3 has no such reversal in the route rendering and that the
opponents in-game immediately head the right way there. This resolves the
apparent mismatch between the fallback counter and the observed route turns:
the counter covers only a dead-end fallback, not every geometric reversal in
the route. Some existing scripts still end before the detected finish:
both scripts end on inverted legs 3 and 5, and one ends early on legs 2 and 4.
In particular, the revised leg-3 ground-road finish at `(0,2)` has a
height-matched connection from the preserved intersection, but both opponent
scripts terminate before reaching it. That is a route-script issue, not
evidence of an elevation mismatch at the finish. See
`course-lane-connectivity.md` for the tile port catalog and per-leg map audit.

## Safe implementation sequence

1. Make a separate experimental course; do not overwrite the supplied scene.
2. Apply the known leg-order, lighting, grid, cell-rotation, and entity-position
   transforms while keeping an audit of every changed record.
3. Identify the existing finish-material tile and the old start location; use
   a known-good tile rather than inventing a material record.
4. Decode or obtain the lane bank, then validate the reversed route network and
   adjust opponent branch scripts.
5. Test the player route and AI behavior in-game before treating the export as
   a finished course.

The first two steps are well-supported and suitable for an experimental
generator. A fully reliable, AI-correct inverted course should wait until the
lane-bank direction and start/finish placement are verified.
