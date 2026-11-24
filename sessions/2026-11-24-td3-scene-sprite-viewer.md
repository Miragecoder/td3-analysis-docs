# TD3 Scene Sprite Viewer / Drop-in Replace Prototype

## Context

Investigated whether the Test Drive III course editor could expose world sprites (trees, shrubs, roadside billboards) in addition to existing Scene UI bitmap slots (title/icon/landA–F). Work landed in `testdrive3-course-editor` commit `c3ca3a8`.

## Findings

- **UI bitmaps vs world sprites are different formats.** UI bitmaps are hardcoded `SceneImageSlot`s decoded/encoded as TD3 LZW+RLE indexed images (`src/shared/scene-dat-images.ts`, `src/shared/td3-image.ts`), with editor overrides recomposed into DAT+LST on export. Scene sprites live in 2D render-descriptor banks (`SCENE01.DAT @ 0x15490`, `SCENE02.DAT @ 0x1229A`, `DATAB.DAT SCENETT1 @ 0x1ED2C`) with a different source/prescale format.
- **Enhanced renderer is the ground truth.** `td3-enhanced-analysis/src/game/render_sprite.c` shows the real prescale path: `sprites_prescale` / `sprite_prescale_one` read source sprite records and generate distance copies. Mode 0 builds per-variant copies from `step_to_size`/`CHILD_STATES`; mode 1 builds one fixed-size copy used for all distances; mode 2/3 alias `id-1`/`id-2` copies. This means distance variants are generated at load time from source records, not stored as separate bitmaps in the DAT.
- **Course-editor sprite spec was cautious.** `spec/scene-render-descriptor-bank-format.md` describes the bank as partially identified/approximate; useful for context, but the enhanced recompilation is the stronger reference for behavior.

## Implementation

### Scene sprites panel (read-only + export)

- Added `Scene sprites` panel in `editor.html` and wiring in `src/browser/editor.ts`.
- Decodes the loaded SCENE01/SCENE02 sprite bank via `extractSceneSprites`, renders with default palette + OTWCOL (`DATAC.DAT @ 0x00495`, slots 16..127), index 0 transparent.
- Added `Mode` toggle: **Raw decoded (explore)** builds mode 2 roots per-variant for exploration; **Engine aliases (accurate)** treats mode 2/3 as `id-1`/`id-2` aliases.
- Added extraction log capture/display (`[spriteextract]` warnings) for debugging rejected/corrupt records.
- Fixed extraction overflow by using a 32-bit-pointer internal workspace and larger buffer; added duplicate-leaf rollback to save space.

### Constrained drop-in replace prototype

- Added `getSceneSpriteSourceInfo()` and `replaceSceneSpriteSourcePixels()` in `src/shared/scene-sprites.ts`.
- Replacement is in-place over source row pixel bytes only: same `w,h` and same per-row active width (`m`) layout. The editor accepts an exact-dimension PNG, quantizes it to the sprite palette, writes bytes into `workingData`, refreshes the view, and marks DAT dirty.
- Because TD3 prescales mode 0 source sprites at load time, replacing the source pixels updates downscaled distance variants automatically.

## Caveats / Next steps

- Replace is constrained: same dimensions/layout, no bank repacking/resizing, uses selected sprite’s first `rXX` source ref.
- Full sprite editing would need a proper source-bank encoder/packer, round-trip validation against the enhanced/original prescaler, and careful handling of mode/alias semantics.
- SCENETT1 (`DATAB.DAT`) is not yet exposed in the editor panel.

## Files

- `td3-analysis-docs/repos/testdrive3-course-editor/editor.html`
- `td3-analysis-docs/repos/testdrive3-course-editor/src/browser/editor.ts`
- `td3-analysis-docs/repos/testdrive3-course-editor/src/shared/scene-sprites.ts`
- Reference: `td3-analysis-docs/repos/testdrive3-enhanced-analysis/src/game/render_sprite.c`
