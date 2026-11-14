# Codebase overview

## Scope

`..\testdrive3-enhanced` is an SDL3-native C11 port of Accolade's *Test Drive
III: The Passion* (1990), loading the user's original `TDIII.EXE` or alternate
`TD3.EXE` plus its data files. It is not a DOS emulator and does not include
the original game files. The neighboring `..\Test-Drive-3-Maps` is a separate
TypeScript/Vite extractor and viewer that documents and visualizes the game's
maps, objects, and image resources.

The enhanced engine's faithful core is ported from the `td3port/` area of an
upstream reverse-engineering repository. Its own `ENGINE.md` points to
specifications, symbols, and disassembly tools there. That upstream repository
was not present in the workspace examined for this walkthrough, so the notes
here rely on the checked-out enhanced source and its documentation.

## Main components

| Area | Role |
|---|---|
| `src/main.c` | Parses options, finds and loads the original executable, initializes host/VGA/modules, then calls `game_main()`. `--check` validates the executable without starting the game. |
| `src/mem.h`, `src/mem.c`, `src/symbols.h` | Simulated real-mode address space, executable unpacking/relocation, typed memory accessors, far pointers, divide-error behavior, and generated names for original symbols. |
| `src/host.*` | SDL3 host: window, frame presentation, 145.652 Hz timer scheduling, VGA retrace pacing, XT keyboard input, mouse/gamepad, audio, file lookup, and worker pool. |
| `src/modules.c` | Registers and initializes platform handlers plus code-pointer mappings used by the game. |
| `src/platform/` | DOS-facing services, VGA mode 13h, pages and drawing, palette/pictures, keyboard, mouse/joystick, timer, and view presentation. |
| `src/sound/` | Original sound driver and effects, AdLib/OPL2 and PC speaker devices. OPL emulation is provided by `third_party/nuked-opl3/`. |
| `src/game/flow*.c` | Startup, configuration, menus, race/leg loop, results, scoring, and files. |
| `src/game/sim*.c` | Driving physics, controls, collisions, traffic, and replay. |
| `src/game/render*.c` | World construction, projection, faces, sprites, mirror, and sky. |
| `src/game/hud*.c` | Cockpit and per-frame instruments/overlays. |
| `src/enhanced/` | Optional high-resolution, smooth renderer, sampled from game frames and composed over the VGA image; detailed design is in `ENHANCED.md`. |
| `launcher/` | wxWidgets launcher and configuration UI. |

`CMakeLists.txt` builds the C11 executable, SDL3 dependency, bundled OPL2
library, and wxWidgets launcher. `--classic` disables the enhanced view.

## Startup and frame flow

1. `main()` parses display, audio, car/course/skill, and enhancement options.
2. It locates `TDIII.EXE` (falling back to `TD3.EXE`) and calls
   `mem_load_exe()`. The loader handles MZ/EXEPACK data, applies relocations,
   verifies the expected image size and identifying strings, and puts the
   image at segment `0x1000`.
3. Unless `--check` was requested, it configures the enhanced renderer, starts
   SDL host services, initializes the VGA model, and calls `modules_init()`.
4. `game_main()` initializes original-game state and enters its state machine:
   title/screens and menus lead into `race_run()`, which loads a leg, repeatedly
   updates simulation/render/HUD, and processes results.
5. Timer ticks and platform input are delivered through host callbacks while
   game busy-waits pump `host_pump()`. The tick rate is 145.652 Hz; the VGA
   retrace model is 70.086 Hz. Race frames are paced by timer ticks.
6. The original renderer draws into VGA memory (`A000:0000`) and its RAM page.
   The VGA module composes the display image. With enhancements active,
   `src/enhanced/` uses snapshots of game frames to draw a smooth, higher
   resolution view and lays it over the original picture.

## How to explore

- Start with `README.md` for build/run instructions and command-line options.
- Read `ENGINE.md` for faithful-port constraints, module responsibilities,
  timing, input, and memory rules.
- Read `ENHANCED.md` for how the second renderer snapshots and re-renders the
  world and where it hooks into the game.
- Trace startup through `src/main.c`, `src/modules.c`, and
  `src/game/flow.c`; follow race execution in `src/game/flow_race.c`.
- For the memory model, begin at `src/mem.h` and `src/mem.c`; for subsystems,
  consult `src/platform/platform.h`, `src/platform/vga.h`, and the headers
  under `src/game/`, `src/sound/`, and `src/enhanced/`.
- The second repository's `spec/` contains independent TD3 file-format
  documentation; its README describes its parsers and viewer.

## Build/check

The project README documents a CMake/Ninja build with SDL3, a C11 compiler,
and optionally wxWidgets 3.2. The focused executable check is:

```text
testdrive3-enhanced.exe --game-dir Game --check
```

This checks that the original executable loads and matches the expected build;
it does not run the game or validate every subsystem.
