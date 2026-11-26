# TD3 AI Opponent Routing

## Context

Implemented an opt-in TD3-enhanced opponent AI route-generation feature that produces valid 10-byte branch scripts at leg load, while keeping the legacy script format for DOS/course-editor compatibility. Work landed in `testdrive3-enhanced-analysis` commit `5f32aa0`.

## Problem / Findings

- SCENE02 shipped opponent scripts are bad for some legs; SCENE01 scripts are usable. Original and custom courses must still work.
- Engine finish trigger is `DS_surface_under_car == 0x1F` (`race_run`/`surf`), but static w3 0x1F-material cells are **not** a clean intended-finish identifier: many decoy 0x1F tiles exist (e.g. SCENE02/LEG05 `0x73` gas-station tiles), and designated finish-line tiles `0x59` (SCENE01/03) / `0x51` (SCENE02) do not appear as simple w3 0x1F cells in static scans.
- Agreed strategy: for SCENE02 use the unique `0x51` finish-line cell when present; for other scenes trace original opponent branch scripts through the lane graph to their termination cell and require agreement. If unresolved, keep original scripts. Custom courses should provide proper legacy branch scripts.

## Implementation

### Finish resolution and generation

- `ai_trace_script_finish()` traces an original opponent script with the same lane/control-point/exit semantics used by the route probe.
- `ai_resolve_finish_cell()`: SCENE02 prefers unique `0x51` cell with lane waypoints, falls through to script trace if absent; other scenes trace both opponents and accept only an agreed valid termination cell.
- `ai_probe_routes()` BFS-probes valid lane routes to the resolved finish cell and writes `DS_opp_scripts` in engine format. Syncs object behavior nibble in `DS_obj_flags`.
- Visited-state dedup by `(cell, wp, beh, initial)` prevents loop/detour routes (e.g. SCENE01/LEG05 detour that exited and re-entered an intersection).

### Modes

- Replaced `--ai-route-finding 0|1` bool with `--ai-opponent-routing legacy|deterministic|random` (`0|1|2`), kept `--ai-route-finding` as CLI alias.
- Launcher: **AI Opponent Routing** choice (`Legacy`, `Modern: Deterministic`, `Modern: Random`). Setting `AiOpponentRouting`; migrates old `AiRouteFinding=1` to Random.
- `Legacy`: scene scripts.
- `Modern: Random`: independent random valid route per opponent.
- `Modern: Deterministic`: stable routes; for two opponents prefers earliest physical divergence, then routes matching original scripts, then lower total steps, giving separate routes when possible.

### Debug aid

- `TD3_DEBUG_KEYS=1` / launcher Debug keys: enables dormant debug keys and makes car invulnerable; F8 ends current leg (`sim_controls.c`).

## Files

- `td3-analysis-docs/repos/testdrive3-enhanced-analysis/src/game/flow_race.c`
- `td3-analysis-docs/repos/testdrive3-enhanced-analysis/src/portcfg.h`
- `td3-analysis-docs/repos/testdrive3-enhanced-analysis/src/portcfg.c`
- `td3-analysis-docs/repos/testdrive3-enhanced-analysis/src/main.c`
- `td3-analysis-docs/repos/testdrive3-enhanced-analysis/src/game/sim_controls.c`
- `td3-analysis-docs/repos/testdrive3-enhanced-analysis/launcher/*`
- `td3-analysis-docs/repos/testdrive3-enhanced-analysis/README.md`
- `td3-analysis-docs/repos/testdrive3-enhanced-analysis/ENHANCED.md`
- `td3-analysis-docs/repos/testdrive3-enhanced-analysis/CHANGELOG.md`
