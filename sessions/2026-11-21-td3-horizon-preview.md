# TD3 Course Editor - Horizon preview fidelity

**Date:** 2026-11-21
**Workspace:** /home/rob/opencode/td3-analysis-docs/repos/testdrive3-course-editor

## Request
Compare Test Drive III Course Editor horizon display vs in-game behavior for SCENE03 leg 1 values:
- Day, sky accent 0x7B, horizon color 0x6868 (slot 0x23), gradient 0x0203
- In-game: bottom green, middle black, top blue (unexpected black band)
- Editor preview did not show the black band.

## Findings
- Editor preview (`src/browser/editor.ts`) used a simple RGB linear gradient from sky accent to fog/remap color; it did not simulate VGA palette stepping.
- In-game `sub_11408` (VGA mode 13h) uses accent as solid sky pair and steps the sky gradient by `+0x0101` in banded groups near the horizon; fog/horizon comes from `word_285B0` remap lookup. Palette-index stepping can cross `0x00`, producing a black band.
- Evidence: `disassembly/functions/seg006_35B8_sub.asm`, `spec/map-format.md`, `spec/color_remap.md`, `src/shared/palette.ts`.

## Changes (Option 4: faithful preview + warning + safe presets)
- `src/browser/editor.ts`
  - Added `getHorizonGradientWarning()` (48-step +0x0101 window) for black-wrap detection.
  - Replaced smooth RGB preview with banded palette-stepped preview approximation.
  - Added preview warning text and apply-time warning feedback.
  - Added `applyHorizonPreset()` with presets: Day clear, Night clear, Dusk warm.
  - Wired preset button handler.
- `editor.html`
  - Added `#horizon-preview-warning`.
  - Added safe preset select/button.
  - Updated horizon help text.
- `src/browser/editor.css`
  - Added warning/preset styling.

## Caveats
- Preview is a pragmatic approximation of `sub_11408`, not a full emulator; exact band geometry can vary.
- `npm run build`/`tsc` could not be run in sandbox (node/npm not installed).
