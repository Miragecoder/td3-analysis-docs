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
| Reverse the five map resources and update day/night | High | Map offsets and LST resource references are known; mode and sky seeds have already been changed successfully in the SCENE03 experiment. |
| Rotate the grid 180 degrees | High for the visible terrain | Move each cell `(x, y)` to `(31-x, 15-y)`, add two quarter-turns to its stored rotation, and preserve elevation. Tile-attached sprite children inherit the transformed placement. |
| Rotate placed entities | Medium-high | The entity arrays and world coordinates are known, and entity headings have a 180-degree flip operation. Counts, parallel fields, cell indices, and object-state bits must all be preserved. |
| Move the player/opponent starts | Medium | Slot 0 is strongly supported as the player start and models `0x02`/`0x03` are the opponents. The exact loader that copies slot-0 coordinates into player runtime position is not yet traced. |
| Relocate the finish | Medium-low | Finish is surface material `0x1F` on tile geometry, not a standalone map marker. The target tile/face at the old start must be identified or safely duplicated. |
| Make AI drive the reversed direction | Low until lane data is decoded | AI uses `P.BIN` waypoint networks and per-opponent branch scripts. Rotating cells and moving starts alone does not reverse waypoint order or guarantee the lane exits connect in the desired direction. |

## Why AI is the main risk

Each tile ID selects a lane template from a `P.BIN` bank. Waypoints carry
local coordinates and exit flags, and map-cell rotation transforms their
positions and exits. The two opponents also have map-header branch scripts;
their scripts differ on Pacific legs 1, 2, 4, and 5, and match on leg 3.
Those scripts choose branch modes at certain lane markers, not direct world
coordinates.

A 180-degree grid transform rotates the lane templates' geometry, but it does
not by itself reverse the sequence of waypoints. A playable reverse course
needs either proof that the rotated lane network is traversable from the new
start, or a reversed/adjusted lane network and matching opponent branch
scripts. The editor currently does not load the course `P.BIN` banks needed to
draw or validate those paths.

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
