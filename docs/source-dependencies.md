# C source files and dependencies

In C, `.c` files are usually called **source files** or **translation units**.
Each is compiled separately into an object file and then linked into the
executable. They are not classes, and calling them “scripts” would be unusual
for compiled C code.

The diagram below is a simplified **logical/runtime dependency map** of the
main source files, not a complete `#include` graph or function call graph.
Arrows show major startup, ownership, or runtime relationships. C modules
generally share declarations through headers (`game/game.h`,
`platform/platform.h`, `host.h`, etc.); one `.c` file does not normally
`#include` another `.c` file. CMake compiles the source files together into
`testdrive3-enhanced`.

```mermaid
flowchart TD
    main["src/main.c"]
    mem["src/mem.c<br/>DOS memory + EXE loader"]
    host["src/host.c<br/>SDL host services"]
    vga["src/platform/vga.c<br/>VGA mode 13h model"]
    modules["src/modules.c<br/>module registration"]
    gameflow["src/game/flow.c<br/>game state machine"]
    race["src/game/flow_race.c<br/>race / leg loop"]

    subgraph platform["Platform source files"]
        dos["platform/dos.c"]
        gfx["platform/gfx.c"]
        input["platform/kbd.c + joy_mouse.c"]
        pal["platform/pal.c + pic.c + text.c"]
        timer["platform/timer.c + view.c"]
    end

    subgraph game["Gameplay source files"]
        menus["game/flow_menus.c + flow_screens.c"]
        files["game/flow_files.c + flow_scores.c"]
        sim["game/sim_controls.c + sim_physics.c"]
        traffic["game/sim_traffic.c + sim_collide.c"]
        replay["game/sim_replay.c"]
        render["game/render.c + render_world.c"]
        renderparts["game/render_object.c + render_sprite.c<br/>render_mirror.c + render_raster.c + render_overlay.c"]
        hud["game/hud.c + hud_topbar.c + hud_overlay.c"]
    end

    subgraph sound["Sound source files"]
        sounddriver["sound/driver.c + api.c + sfx.c"]
        sounddevices["sound/adlib.c + speaker.c + parked.c"]
    end

    subgraph enhanced["Enhanced renderer source files"]
        enh["enhanced/enhanced.c"]
        enhparts["enhanced/enh_scene.c + enh_raster.c<br/>enh_sprite.c + enh_far.c + enh_keys.c"]
    end

    main -->|"load and validate"| mem
    main -->|"initialize SDL"| host
    main -->|"initialize VGA"| vga
    main -->|"initialize handlers"| modules
    main -->|"configure enhancements"| enh
    main -->|"enter game"| gameflow

    modules --> platform
    modules --> sound
    modules --> sim

    gameflow --> menus
    gameflow --> files
    gameflow --> race
    race --> sim
    race --> traffic
    race --> replay
    race --> render
    render --> renderparts
    race --> hud
    race --> timer

    platform --> host
    vga --> mem
    vga --> host
    sound --> host
    gameflow --> dos
    gameflow --> gfx
    gameflow --> input
    gameflow --> pal

    race --> enh
    render --> enh
    hud --> enh
    enh --> enhparts
    vga --> enh
    enh --> host
```

The broad boxes group related `.c` files to keep the graph readable. They do
not mean those files are a single compilation unit. For a strict dependency
audit, inspect the `#include` directives and function declarations in the
source and headers; a header dependency alone does not mean every function in
that header is called.
