# Test Drive III Analysis Notes

This repository records findings about the neighboring Test Drive III projects:

- `repos\testdrive3-enhanced-analysis`: native C/SDL3 port and enhanced renderer for
  *Test Drive III: The Passion* (1990).
- `repos\testdrive3-course-editor`: TypeScript extraction tools and browser viewer/editor for
  maps and assets from the same game.

These are Git submodules. Clone with `--recurse-submodules`, or initialize
them after cloning with:

```sh
git submodule update --init --recursive
```

## Notes

- [Codebase overview](docs/codebase-overview.md): components, startup, and frame flow.
- [Real-mode memory model](docs/real-mode-memory.md): how the C port addresses DOS-era state.
- [Source dependencies](docs/source-dependencies.md): Mermaid map of major source-file relationships.
- [Course data structure](docs/course-data-structure.md): course/leg files, maps, assets, and lighting mode.
- [Map entities and gameplay](docs/map-entities-and-gameplay.md): weather surfaces, finish detection, object records, traffic lanes, and police behavior.
- [Course inversion feasibility](docs/course-inversion-feasibility.md): which parts of a 180-degree, reverse-leg course are understood and what remains risky.
- [Initial walkthrough](sessions/2026-11-14_1630+0100.md): scope and findings.
- [Course-data walkthrough](sessions/2026-11-14_1655+0100.md): evidence and open questions for course files.
- [Pacific night / SCENE03 walkthrough](sessions/2026-11-14_1703+0100.md): leg 5 color data and clone feasibility.
- [Tile previews and gameplay-data investigation](sessions/2026-11-14_1847+0100.md): rotated tile thumbnails and findings about map-driven game logic.
- [Entity overlay and AI route scripts](sessions/2026-11-14_1906+0100.md): map-entity inspection and Pacific opponent branch-script comparison.
- [Tile vegetation, inversion, and repository layout](sessions/2026-11-14_1931+0100.md): tile-child sprites, reverse-course risks, and submodule prerequisites.

Add a timestamped Markdown file under `sessions/` for each walkthrough, and
update the human-readable documents under `docs/` when findings are confirmed.
