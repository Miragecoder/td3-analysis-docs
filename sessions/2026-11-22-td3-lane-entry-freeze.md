# TD3 Enhanced - Lane entry search freeze fix

**Date:** 2026-11-22
**Workspace:** /home/rob/opencode/td3-analysis-docs/repos/testdrive3-enhanced-analysis

## Request
Investigate a freeze in Test Drive III Enhanced (DOS-faithful reverse-engineered codebase). User reported: after speeding past a police car and getting it engaged on some curvy road tiles, the game can freeze. User provided a GDB core dump showing the main thread stuck in `objects_update()`.

## Findings
- Stack: `objects_update()` -> `moving_objects_update()` -> `vehicle_follow_lane()` (`src/game/sim_traffic.c`). No debug symbols, so GDB reported the outer function.
- `vehicle_follow_lane()` searches for a matching lane-entry waypoint when a vehicle exits a cell. If not found in the adjacent cell, it falls back to searching the original cell for `0xC0 | (exit_edge & 3)`.
- The existing comment already noted the problem: "Faithful: a waypoint count of 0 loops 256 times, and bad data can loop forever as in the original." If the fallback search also fails, the code reset the same fallback key/cell and looped forever.
- This matches the reported trigger: police engagement/U-turn on curvy road tiles can produce a lane transition with no matching entry waypoint.

## Changes
- `src/game/sim_traffic.c`
  - Added an `ENH` guard in the lane-entry search fallback. If the fallback pass has already been tried and still finds no matching entry, bail out with an in-place U-turn/wait (shuttle-style reversal) instead of looping forever.
  - Preserves normal behavior: the guard only activates in the previously infinite-loop/bad lane-data case.

## Validation
- Code-path diagnosis matched the reported symptoms and stack; core file was on the user's machine and not inspected directly in the sandbox.
- User tested usual hotspot races after the fix and reported no freezes or visible regressions.

## Commit
- `d3e8558` (local master) — `Avoid infinite loop in lane entry search`
- Author: `Rob Wijkstra <rob.wijkstra@gmail.com>`

## Caveats
- Recovery behavior for bad/unmatched lane transitions is an arbitrary but safe U-turn/wait; it should only appear where the original game would have frozen.
- Not built in this sandbox (no C toolchain configured in PATH for a quick rebuild check). User should rebuild/retest in their normal environment.
