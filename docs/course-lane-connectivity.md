# Test Drive III lane connectivity

Generated from `SCENETTP.BIN` embedded in `DATAB.DAT` and the supplied Pacific/Cape Cod map files.
The game can load a scene-specific `<scene>P.BIN` instead; none was supplied in this workspace, so this
report describes the shared bank and marks that limitation where relevant.

## Lane-bank format and connection rule

The shared bank has 123 tile-index entries (tile IDs 0x00–0x7A) and 96 tile IDs with waypoints
(80 distinct lane records: 214 entry ports and 224 exit ports).
The 27 indexed tile IDs with no waypoints are: 0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x19, 0x1a, 0x1b, 0x1c, 0x1d, 0x1e, 0x2a, 0x2b, 0x2c, 0x2e, 0x5d, 0x5e, 0x5f, 0x60, 0x65, 0x66, 0x6e.
Each record has an 8-byte waypoint: 16-bit flags, 16-bit local height, signed 16-bit local X and Z.
The lane header low 7 bits give its waypoint count; the high bit is retained as a lane/header flag.

Flags with `(flags & 0xC0) == 0x80` are exits; `(flags & 0xC0) == 0xC0` are entries. Local sides
0–3 are north/east/south/west. A cell rotation adds to the local side modulo 4. For an exit side `d`
and adjacent cell rotation `r`, the required entry flag is
`0xC0 | (((d ^ 2) - r) & 3)`. This is the exact key the game searches for in `sim_traffic.c`.

The simulation wraps adjacent map-cell indices at the 32×16 map edges. It matches entry flags only:
it does not reject a link because lane heights differ. The target height is
`(map elevation * 256 + waypoint height) & 0xFFFF` (then scaled by 8 for runtime); the catalog
shows stored waypoint words as signed two’s-complement values for readability. A nonzero height delta is a
grade/vertical discontinuity to inspect, not a failed link by the game. If no entry key exists, the
traffic routine searches back in the current tile for the opposite entry and turns around there.

## Tile port catalog

Sides below are in the tile-local orientation; rotate them using the rule above. `index@side:height`
identifies a waypoint record. Interior waypoint heights are included because they expose ramps and
overpasses even when their ports are at matching elevations.

| Tile | Header | Waypoints | Entries | Exits | Interior heights |
|---|---:|---:|---|---|---|
| 0x0a | 0x04 | 4 | 0@W:3, 2@E:3 | 1@E:3, 3@W:3 | — |
| 0x0b | 0x0a | 10 | 0@W:3, 5@E:3 | 4@E:3, 9@W:2 | 67, 3 |
| 0x0c | 0x0a | 10 | 0@E:3, 5@S:3 | 4@S:3, 9@E:3 | 3 |
| 0x0d | 0x0a | 10 | 0@W:3, 2@E:3, 4@N:2, 7@S:2 | 1@E:3, 3@W:3, 6@S:2, 9@N:2 | 2 |
| 0x0e | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x0f | 0x0c | 12 | 0@N:258, 6@S:2 | 5@S:2, 11@N:258 | 130, 162 |
| 0x10 | 0x10 | 16 | 0@N:258, 8@S:2 | 7@S:2, 15@N:258 | 274, 242, 50, 98 |
| 0x11 | 0x10 | 16 | 0@N:258, 8@S:2 | 7@S:2, 15@N:258 | 258, 162, 146, 50 |
| 0x12 | 0x10 | 16 | 0@W:130, 8@E:130 | 7@E:130, 15@W:130 | 130, 114, 98 |
| 0x13 | 0x0a | 10 | 0@W:2, 5@E:130 | 4@E:130, 9@W:2 | 2, 18, 82 |
| 0x14 | 0x0c | 12 | 0@W:130, 6@E:2 | 5@E:2, 11@W:130 | 130, 66, 34 |
| 0x15 | 0x0c | 12 | 0@N:2, 6@S:2, 10@E:2 | 3@S:2, 5@E:2, 9@N:2 | 2 |
| 0x16 | 0x1c | 28 | 0@N:2, 7@S:2, 14@E:2, 21@W:2 | 3@S:2, 6@E:2, 10@N:2, 13@W:2, 17@W:2, 20@S:2, 24@E:2, 27@N:2 | 2 |
| 0x17 | 0x0e | 14 | 0@N:258, 7@S:130 | 6@S:130, 13@N:258 | 226, 194, 66, 50 |
| 0x18 | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x1f | 0x0e | 14 | 0@N:2, 7@S:2 | 6@S:2, 13@N:2 | -18, -98, -114, -34 |
| 0x20 | 0x11 | 17 | 0@N:2, 9@S:2 | 8@S:2, 16@N:2 | 2, 34, 50, -14, 18 |
| 0x21 | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x22 | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x23 | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x24 | 0x12 | 18 | 0@N:2, 9@S:2, 15@E:2 | 5@S:2, 8@E:2, 14@N:2 | 98, 18, -50, 2, 10, 34 |
| 0x25 | 0x22 | 34 | 0@N:2, 8@S:2, 17@E:2, 26@W:2 | 4@S:2, 7@E:2, 12@N:2, 16@W:2, 21@W:2, 25@S:2, 30@E:2, 33@N:2 | 138, 114, 34, 130 |
| 0x26 | 0x0c | 12 | 0@E:2, 6@S:2 | 5@S:2, 11@E:2 | 2, -50, -18 |
| 0x27 | 0x10 | 16 | 0@N:2, 8@S:2 | 7@S:2, 15@N:2 | 82, 2, -66, -50 |
| 0x28 | 0x12 | 18 | 0@N:2, 9@S:2 | 8@S:2, 17@N:2 | 34, 50, 18 |
| 0x29 | 0x0e | 14 | 0@N:2, 7@S:2 | 6@S:2, 13@N:2 | -18, -98, -114, -34 |
| 0x2d | 0x04 | 4 | 0@N:146, 2@S:146 | 1@S:146, 3@N:146 | — |
| 0x2f | 0x0c | 12 | 0@N:146, 6@S:2 | 5@S:2, 11@N:146 | 130, 98, 34 |
| 0x30 | 0x03 | 3 | 0@S:2 | 2@S:2 | 2 |
| 0x31 | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x32 | 0x0c | 12 | 0@E:2, 6@S:2 | 5@S:2, 11@E:2 | 2 |
| 0x33 | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x34 | 0x06 | 6 | 0@N:2, 2@S:2 | 1@S:2, 5@N:2 | 2 |
| 0x35 | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x36 | 0x06 | 6 | 0@N:2, 2@S:2 | 1@S:2, 5@N:2 | 2 |
| 0x37 | 0x0c | 12 | 0@N:2, 6@S:2 | 5@S:2, 11@N:2 | 2, 34 |
| 0x38 | 0x10 | 16 | 0@N:2, 2@S:2, 4@W:2, 10@E:2 | 1@S:2, 3@N:2, 9@E:2, 15@W:2 | 2, 146 |
| 0x39 | 0x0a | 10 | 0@N:2, 2@S:2, 4@W:2, 7@E:2 | 1@S:2, 3@N:2, 6@W:2, 9@E:2 | 2 |
| 0x3a | 0x10 | 16 | 0@N:2, 2@S:2, 4@W:2, 10@E:2 | 1@S:2, 3@N:2, 9@E:2, 15@W:2 | 2, -146 |
| 0x3b | 0x10 | 16 | 0@W:3, 2@E:3, 4@N:2, 10@S:2 | 1@E:3, 3@W:3, 9@S:2, 15@N:2 | 2, -146 |
| 0x3c | 0x0c | 12 | 0@S:2 | 11@S:2 | 2 |
| 0x3d | 0x88 | 8 | 0@N:2, 4@S:2 | 3@S:2, 7@N:2 | 2 |
| 0x3e | 0x88 | 8 | 0@N:2, 4@S:2 | 3@S:2, 7@N:2 | 2, 50 |
| 0x3f | 0x8c | 12 | 0@N:2, 6@S:2 | 5@S:2, 11@N:2 | 2 |
| 0x40 | 0x0c | 12 | 0@N:2, 6@S:2 | 5@S:2, 11@N:2 | 2 |
| 0x41 | 0x0a | 10 | 0@N:2, 5@S:2 | 4@S:2, 9@N:2 | 2 |
| 0x42 | 0x0c | 12 | 0@E:2, 6@S:2 | 5@S:2, 11@E:2 | 2 |
| 0x43 | 0x0c | 12 | 0@N:2, 6@S:2 | 5@S:2, 11@N:2 | 2, 50 |
| 0x44 | 0x0a | 10 | 0@N:2, 5@S:2 | 4@S:2, 9@N:2 | 66, 2 |
| 0x45 | 0x06 | 6 | 0@N:2, 3@S:2 | 2@N:2, 5@S:2 | 2 |
| 0x46 | 0x0c | 12 | 0@E:2, 6@S:2 | 5@S:2, 11@E:2 | 2 |
| 0x47 | 0x0e | 14 | 0@N:258, 7@S:2 | 6@S:2, 13@N:258 | 226, 130, 82, 98, 2 |
| 0x48 | 0x0c | 12 | 0@N:2, 6@S:258 | 5@S:258, 11@N:2 | 2, 178, 210 |
| 0x49 | 0x0e | 14 | 0@N:258, 7@S:258 | 6@S:258, 13@N:258 | 258, 226, 242, 322 |
| 0x4a | 0x04 | 4 | 0@N:258, 2@S:258 | 1@S:258, 3@N:258 | — |
| 0x4b | 0x0c | 12 | 0@N:2, 6@E:2 | 5@E:2, 11@N:2 | 2 |
| 0x4c | 0x0c | 12 | 0@N:258, 6@E:258 | 5@E:258, 11@N:258 | 258 |
| 0x4d | 0x0a | 10 | 0@N:2, 5@S:2 | 4@S:2, 9@N:2 | 2 |
| 0x4e | 0x0e | 14 | 0@N:2, 7@S:258 | 6@S:258, 13@N:2 | 50, 130, 178, 210, 258 |
| 0x4f | 0x0c | 12 | 0@N:2, 6@S:2 | 5@S:2, 11@N:2 | 2, -34 |
| 0x50 | 0x0e | 14 | 0@N:2, 7@S:2 | 6@S:2, 13@N:2 | 2, -50, 50 |
| 0x51 | 0x0c | 12 | 0@E:2, 6@S:2 | 5@S:2, 11@E:2 | 2 |
| 0x52 | 0x17 | 23 | 0@N:258, 6@S:258, 12@W:2, 18@E:2 | 5@S:258, 11@N:258, 17@E:2, 22@W:2 | 258, 354, 274, 2 |
| 0x53 | 0x11 | 17 | 0@N:258, 3@S:258, 6@W:2, 12@E:2 | 2@N:258, 5@S:258, 11@E:2, 16@W:2 | 258, 2 |
| 0x54 | 0x84 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x55 | 0x8a | 10 | 0@N:2, 5@S:2 | 4@S:2, 9@N:2 | 66, 2 |
| 0x56 | 0x84 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x57 | 0x8d | 13 | 0@N:2, 2@S:2, 8@E:2 | 1@S:2, 4@N:2, 7@E:2, 12@N:2 | 2 |
| 0x58 | 0x0d | 13 | 0@N:2, 2@S:2, 8@E:2 | 1@S:2, 4@N:2, 7@E:2, 12@N:2 | 2 |
| 0x59 | 0x06 | 6 | 0@N:2, 3@S:2 | 2@S:2, 5@N:2 | 2 |
| 0x5a | 0x04 | 4 | 0@W:322, 2@E:322 | 1@E:322, 3@W:322 | — |
| 0x5b | 0x0a | 10 | 0@N:258, 5@E:322 | 4@E:322, 9@N:258 | 258, 290, 322 |
| 0x5c | 0x0c | 12 | 0@W:322, 6@E:258 | 5@E:258, 11@W:322 | 322, 370, 258 |
| 0x61 | 0x0e | 14 | 0@N:2, 7@S:2 | 6@S:2, 13@N:2 | 50, 98, 2 |
| 0x62 | 0x06 | 6 | 0@N:2, 3@S:2 | 2@N:2, 5@S:2 | 2 |
| 0x63 | 0x90 | 16 | 0@N:2, 8@S:2 | 7@S:2, 15@N:2 | 2, 18, 34 |
| 0x64 | 0x06 | 6 | 0@N:2, 3@S:2 | 2@N:2, 5@S:2 | 2 |
| 0x67 | 0x0c | 12 | 0@E:2, 6@S:2 | 5@S:2, 11@E:2 | 2 |
| 0x68 | 0x0a | 10 | 0@N:2, 5@S:2 | 4@S:2, 9@N:2 | 2 |
| 0x69 | 0x12 | 18 | 0@N:2, 9@S:2 | 8@S:2, 17@N:2 | 2, -18, -34 |
| 0x6a | 0x0e | 14 | 0@N:2, 7@S:2 | 6@S:2, 13@N:2 | 34, -34, -50 |
| 0x6b | 0x06 | 6 | 0@N:2, 3@S:2 | 2@N:2, 5@S:2 | 2 |
| 0x6c | 0x92 | 18 | 0@N:2, 9@S:2 | 8@S:2, 17@N:2 | 98, 114, 130, 66 |
| 0x6d | 0x06 | 6 | 0@N:2, 3@S:2 | 2@N:2, 5@S:2 | 2 |
| 0x6f | 0x0c | 12 | 0@E:130, 6@S:130 | 5@S:130, 11@E:130 | 130, 82 |
| 0x70 | 0x0a | 10 | 0@N:2, 5@S:2 | 4@S:2, 9@E:2 | 2 |
| 0x71 | 0x02 | 2 | 0@E:2 | 1@S:2 | — |
| 0x72 | 0x02 | 2 | 0@E:2 | 1@S:2 | — |
| 0x73 | 0x0a | 10 | 0@E:2, 5@S:2 | 4@W:2, 9@E:2 | 2 |
| 0x74 | 0x8a | 10 | 0@N:2, 5@S:2 | 4@S:2, 9@N:2 | 2, 50, 66, 18 |
| 0x75 | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x76 | 0x0e | 14 | 0@N:2, 6@S:2 | 5@S:2, 13@N:2 | 2, -82, 98, 114 |
| 0x77 | 0x04 | 4 | 0@N:2, 2@S:2 | 1@S:2, 3@N:2 | — |
| 0x78 | 0x10 | 16 | 0@N:2, 2@S:2, 4@W:75, 10@E:75 | 1@S:2, 3@N:2, 9@E:75, 15@W:75 | 130, 146 |
| 0x79 | 0x0a | 10 | 0@N:2, 2@S:2, 4@W:75, 7@E:75 | 1@S:2, 3@N:2, 6@W:75, 9@E:75 | 82 |
| 0x7a | 0x06 | 6 | 0@W:66, 3@E:2 | 2@E:2, 5@W:66 | 2 |

## Course-map edge audit

Counts are for every exit waypoint in each shipped map, including unused branches and dead-end spurs.
They are not a judgment that every missing edge is a design error.

Tile IDs `0x7B`–`0x7F` have no index entries in this shared bank. Their map cells are excluded
as source lanes, and links pointing to them are reported as unresolved rather than claimed unmatched.

| Course | Leg | Indexed exit records | Matching neighbor entry | No matching entry | Unindexed source cells | Unindexed target | Nonzero height pairs |
|---|---:|---:|---:|---:|---:|---:|---:|
| Pacific Coast | 1 | 265 | 265 | 0 | 0 | 0 | 10 |
| Pacific Coast | 2 | 239 | 239 | 0 | 0 | 0 | 6 |
| Pacific Coast | 3 | 286 | 286 | 0 | 0 | 0 | 7 |
| Pacific Coast | 4 | 305 | 303 | 2 | 0 | 0 | 4 |
| Pacific Coast | 5 | 293 | 292 | 1 | 0 | 0 | 8 |
| Cape Cod | 1 | 258 | 186 | 70 | 1 | 2 | 23 |
| Cape Cod | 2 | 528 | 424 | 89 | 10 | 15 | 64 |
| Cape Cod | 3 | 387 | 306 | 81 | 0 | 0 | 18 |
| Cape Cod | 4 | 462 | 346 | 93 | 13 | 23 | 39 |
| Cape Cod | 5 | 383 | 304 | 65 | 12 | 14 | 56 |


Unmatched-edge examples for Pacific Coast:

- Pacific Coast leg 4: tile 0x3e at (18,10) exits S to tile 0x05 at (18,11); no entry 0xc0.
- Pacific Coast leg 4: tile 0x3e at (18,10) exits N to tile 0x08 at (18,9); no entry 0xc1.
- Pacific Coast leg 5: tile 0x0f at (31,2) exits E to tile 0x5e at (0,2); no entry 0xc3.

Unmatched-edge examples for Cape Cod:

- Cape Cod leg 1: tile 0x0a at (5,0) exits E to tile 0x00 at (6,0); no entry 0xc3.
- Cape Cod leg 1: tile 0x0a at (5,0) exits W to tile 0x00 at (4,0); no entry 0xc1.
- Cape Cod leg 1: tile 0x0b at (7,0) exits S to tile 0x74 at (7,1); no entry 0xc3.
- Cape Cod leg 1: tile 0x0a at (11,0) exits S to tile 0x6e at (11,1); no entry 0xc1.
- Cape Cod leg 1: tile 0x4e at (14,0) exits S to tile 0x75 at (14,1); no entry 0xc3.
- Cape Cod leg 2: tile 0x51 at (1,1) exits S to tile 0x04 at (1,2); no entry 0xc0.
- Cape Cod leg 2: tile 0x15 at (2,1) exits W to tile 0x51 at (1,1); no entry 0xc0.
- Cape Cod leg 2: tile 0x16 at (6,1) exits S to tile 0x66 at (6,2); no entry 0xc0.
- Cape Cod leg 2: tile 0x16 at (6,1) exits S to tile 0x66 at (6,2); no entry 0xc0.
- Cape Cod leg 2: tile 0x52 at (10,1) exits W to tile 0x00 at (9,1); no entry 0xc1.
- Cape Cod leg 3: tile 0x52 at (0,0) exits E to tile 0x51 at (1,0); no entry 0xc3.
- Cape Cod leg 3: tile 0x51 at (1,0) exits E to tile 0x08 at (2,0); no entry 0xc2.
- Cape Cod leg 3: tile 0x13 at (8,0) exits S to tile 0x5e at (8,1); no entry 0xc0.
- Cape Cod leg 3: tile 0x4f at (20,0) exits S to tile 0x40 at (20,1); no entry 0xc3.
- Cape Cod leg 3: tile 0x4f at (22,0) exits W to tile 0x2e at (21,0); no entry 0xc2.
- Cape Cod leg 4: tile 0x53 at (2,0) exits N to tile 0x00 at (2,15); no entry 0xc2.
- Cape Cod leg 4: tile 0x53 at (2,0) exits E to tile 0x00 at (3,0); no entry 0xc3.
- Cape Cod leg 4: tile 0x53 at (2,0) exits W to tile 0x00 at (1,0); no entry 0xc1.
- Cape Cod leg 4: tile 0x4f at (18,0) exits S to tile 0x40 at (18,1); no entry 0xc3.
- Cape Cod leg 4: tile 0x4f at (23,0) exits W to tile 0x2e at (22,0); no entry 0xc2.
- Cape Cod leg 5: tile 0x4e at (12,1) exits W to tile 0x5d at (11,1); no entry 0xc0.
- Cape Cod leg 5: tile 0x4e at (13,1) exits E to tile 0x5d at (14,1); no entry 0xc0.
- Cape Cod leg 5: tile 0x4e at (10,2) exits N to tile 0x5d at (10,1); no entry 0xc0.
- Cape Cod leg 5: tile 0x67 at (11,2) exits W to tile 0x4e at (10,2); no entry 0xc3.
- Cape Cod leg 5: tile 0x67 at (11,2) exits S to tile 0x29 at (11,3); no entry 0xc1.

## Pacific opponent start-port check

The two opponent records in the Pacific maps start with waypoint state `0x43` in the supplied DAT.
The low six bits select the initial waypoint index. The table follows that waypoint to its first lane
exit and checks the neighboring entry using the engine’s edge key; it does not simulate later branches.

| Pacific stage | Start tile / rotation | Start waypoint (opp 1 / 2) | First exit | Neighbor / required entry | Result |
|---:|---|---|---|---|---|
| Original 1 | 0x31 / 1 | 0x0043 / 0x0043 (index 3) | 3@E h2 | 0x58 at (3,11), entry 0xc2 | match; height delta 0 |
| Original 2 | 0x33 / 1 | 0x0043 / 0x0043 (index 3) | 3@E h2 | 0x27 at (2,1), entry 0xc2 | match; height delta 0 |
| Original 3 | 0x7a / 2 | 0x0043 / 0x0043 (index 3) | 5@E h66 | 0x78 at (1,13), entry 0xc3 | match; height delta 9 |
| Original 4 | 0x33 / 1 | 0x0043 / 0x0043 (index 3) | 3@E h2 | 0x24 at (2,13), entry 0xc0 | match; height delta 0 |
| Original 5 | 0x31 / 1 | 0x0043 / 0x0043 (index 3) | 3@E h2 | 0x16 at (3,4), entry 0xc3 | match; height delta 0 |
| Inverted 1 | 0x31 / 3 | 0x0043 / 0x0043 (index 3) | 3@W h2 | 0x0f at (3,13), entry 0xc0 | match; height delta 0 |
| Inverted 2 | 0x33 / 3 | 0x0043 / 0x0043 (index 3) | 3@W h2 | 0x3c at (0,11), entry 0xc2 | match; height delta 0 |
| Inverted 3 | 0x30 / 3 | 0x0040 / 0x0040 (index 0) | 2@E h2 | 0x58 at (3,14), entry 0xc0 | match; height delta 0 |
| Inverted 4 | 0x33 / 0 | 0x0043 / 0x0043 (index 3) | 3@N h2 | 0x3c at (2,15), entry 0xc2 | match; height delta 0 |
| Inverted 5 | 0x31 / 1 | 0x0043 / 0x0043 (index 3) | 3@E h2 | 0x15 at (2,13), entry 0xc0 | match; height delta 0 |

These initial exits are only a first-step check. A successful entry match rules out the game’s
same-cell dead-end fallback at that first edge, but not a later route/script branch, a visually
misleading turn, or a difference in a scene-specific `P.BIN`. The workspace contains the shared
`SCENETTP.BIN` bank but not the scene-specific lane banks loaded by `stage_load_objects()` when present.

## Leg 3 endpoint example

Pacific leg 3 starts on tile `0x7A`, rotation 2, elevation 1. Its ground-level exit waypoint 2
(local side 1, height 2) rotates to world side 3 and connects to tile `0x30` at `(31,13)`, entry
side 2, height 2. Its other exit waypoint 5 (local side 3, height 66) rotates to world side 1 and
connects to tile `0x78` at `(1,13)`, entry side 3, height 75: the nine-unit rise is the overpass.
After inversion, the ground-level neighbor is `(0,2)` and the overpass remains at `(30,2)`, with
the intersection at `(31,2)`. The finish belongs on the ground-level exit, not the overpass.

## References

- `../../testdrive3-enhanced/src/game/sim_traffic.c`: `lane_list_for_cell()` and `vehicle_follow_lane()`.
- `../../testdrive3-enhanced/src/game/flow_race.c`: per-scene `P.BIN` loading.
- `../repos/testdrive3-course-editor/src/shared/lane-connectivity.ts`: parser and edge-key helpers.
