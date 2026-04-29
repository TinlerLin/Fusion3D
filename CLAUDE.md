# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Fusion3D is a single-page web app that converts parallel-eye (平行眼) Side-by-Side (SBS) stereo pair images into red-cyan anaglyph 3D images (viewable with red-cyan glasses) and wiggle animations (glasses-free). The entire application lives in a single file.

## Development

- No build step, no package manager — open `index.html` directly in a browser
- Tailwind CSS loaded via CDN (`cdn.tailwindcss.com`), no local CSS build
- All JS is vanilla, no framework dependencies

## Architecture

**Single-file structure** (`index.html`):
- **CSS block** (inline `<style>`, lines ~10-136): Custom scrollbar, card/button styling, slider appearance, modal overlay styles
- **HTML body** (lines ~139-335): Header with action buttons, upload zone (drag & drop + URL input), control bar (main sliders + collapsible advanced panel + output mode tabs), 3-panel preview (SBS original | L/R split | output), usage instructions, gallery strip
- **Modal HTML** (lines ~337-344): Full-viewport overlay with large canvas for detailed output viewing
- **JS block** (IIFE, lines ~346-end): The entire application logic

**Key JS components**:

- `PanZoomCanvas` class: Reusable canvas controller managing an image with mouse/touch drag-to-pan and scroll-to-zoom. Used for all preview panes and the modal viewer. Handles resize observation and clamping offsets to prevent panning out of bounds.

- Algorithm pipeline (5-step standardized process):
  1. **`validateSBS(img)`** — Checks aspect ratio (1.5~2.5) to confirm valid SBS format
  2. **`splitSBSImage(img, ratio)`** — Precise split at ratio with ±1px guard zone to prevent pixel bleed at the split boundary
  3. **`detectDepthDirection(leftCV, rightCV)`** — Downsampled cross-correlation to detect depth reversal; returns `{ reversed, confidence }`
  4. **`composeAnaglyph(leftCV, rightCV, config)`** — Fixed channel mapping: Left view → Red, Right view → Cyan (Green+Blue). Depth strength controls horizontal offset between views
  5. **`postProcess(canvas, config)`** — Chain: brightness equalization → saturation adjustment → contrast adjustment → edge smoothing (anti-crosstalk)

- **`runPipeline(img)`** — Master pipeline orchestrator that runs all 5 steps and returns `{ anaglyph, lc, rc, depthReversed }`

- Wiggle player: Canvas-based frame alternation between left/right views at configurable speed (80-300ms)

- Gallery system: Batch upload with thumbnail strip, active image selection, delete support

## Algorithm Specification & Parameter Tuning Guide

### Standard Pipeline

```
Input SBS Image
    │
    ▼
[1] SBS Validation ─── check aspect ratio 1.5~2.5
    │
    ▼
[2] Precise L/R Split ─── split at SBS_SPLIT_RATIO with ±1px guard zone
    │
    ▼
[3] Auto Depth Detection ─── cross-correlation, swap L/R if reversed
    │
    ▼
[4] Channel Mapping ─── L→Red, R→Cyan(G+B) fixed mapping
    │                    depthOffset based on DEPTH_STRENGTH
    ▼
[5] Post-Processing ─── equalizeBrightness → adjustSaturation
                         → adjustContrast → edgeSmooth(anti-crosstalk)
    │
    ▼
Output Anaglyph Canvas
```

### Default Parameters

| Parameter | Default | Range | Description |
|---|---|---|---|
| `SBS_SPLIT_RATIO` | 0.5 | 0.45~0.55 | Horizontal split position |
| `DEPTH_STRENGTH` | 0.5 | 0.3~0.7 (max 0.8) | Stereo depth intensity, controls pixel offset |
| `SATURATION_ADJUST` | -0.20 | -0.15~-0.30 | Saturation suppression to reduce color rivalry |
| `CONTRAST_ADJUST` | 0.10 | 0.08~0.15 | Contrast enhancement for clarity |
| `EDGE_SMOOTH` | 0.25 | 0.20~0.30 | Edge softening strength (blend ratio) |
| `CROSSTALK_LEVEL` | `"medium"` | `low` / `medium` / `high` | Anti-crosstalk kernel radius (1/2/3 px) |
| `AUTO_DEPTH_CORRECT` | `true` | bool | Auto-detect and fix reversed stereo depth |
| `LEFT_VIEW_TO_RED` | `true` | bool | Map left view to red channel |
| `RIGHT_VIEW_TO_CYAN` | `true` | bool | Map right view to cyan (G+B) channels |

### Algorithm Details

#### Step 1: SBS Validation
- Checks image aspect ratio against expected range (1.5~2.5) for standard SBS stereo pairs
- Non-valid images still process but show a warning badge in the pipeline status bar

#### Step 2: Precise L/R Split (Anti Pixel-Bleed)
- Split point calculated as `Math.round(img.width × SBS_SPLIT_RATIO)`
- ±1px guard zone on each side of the split boundary prevents pixel leakage
- Left view: pixels `[0, splitX - 1]`, Right view: pixels `[splitX + 1, img.width - 1]`
- If guard zone would make either view empty, falls back to no-guard split

#### Step 3: Auto Depth Direction Detection
- Downsamples both views to ~100px wide thumbnails
- Extracts center-row luminance profiles (skipping top/bottom 25%)
- Cross-correlation over ±10px horizontal shift range
- `bestShift > 1` indicates right-view features are to the right of left-view → depth is reversed (cross-view format detected)
- Swaps L/R if `reversed && confidence > 0.3`
- Displays status in pipeline badge: "方向OK" or "已修正(反转)"

#### Step 4: Channel Mapping & Composition
- **Prohibited**: Single-image pseudo-3D via channel shifting (old Fusion3D approach)
- **Required**: Always split SBS into L/R views first, then compose
- Fixed mapping: `outR = leftR`, `outG = rightG`, `outB = rightB`
- Depth offset: `pxShift = (DEPTH_STRENGTH - 0.5) / 0.2 × (width × 0.03)`
  - At DEPTH_STRENGTH=0.5, offset is zero (views combined as-is)
  - Positive offset shifts red right/cyan left (enhances depth)
  - Negative offset shifts red left/cyan right (reduces depth)

#### Step 5a: Dual-Channel Brightness Equalization
- Computes mean luminance of Red channel (from left view) vs Cyan channel average (from right view)
- If brightness difference > 5%, scales the darker channel up to match
- Scale capped at 1.05× to prevent over-exposure

#### Step 5b: Saturation Adjustment
- Uses standard luminance weights: `gray = R×0.299 + G×0.587 + B×0.114`
- Applies factor `1 + SATURATION_ADJUST` (default 0.8 = 20% reduction)
- Reduces color rivalry between red and cyan channels

#### Step 5c: Contrast Adjustment
- Classic contrast formula: `factor = (259 × (amount×255 + 255)) / (255 × (259 - amount×255))`
- Applies to all three channels independently

#### Step 5d: Edge Smoothing (Anti-Crosstalk)
- **Critical design**: Only softens edge regions, NOT global blur
- Simplified Sobel operator on luminance to detect edges (threshold: gradient magnitude > 40)
- Box blur applied ONLY to pixels where edge mask = 1
- Kernel radius controlled by `CROSSTALK_LEVEL`: low=1px, medium=2px, high=3px
- Blend ratio controlled by `EDGE_SMOOTH / 0.3` (0.67~1.0)
- This prevents ghosting/crosstalk while preserving fine detail in non-edge areas

### Acceptance Criteria

- [x] 3D effect matches original parallel-eye viewing when wearing left-red right-cyan glasses
- [x] No ghosting/double-image artifacts (controlled by edge smoothing + crosstalk suppression)
- [x] No depth reversal (auto-detection swaps L/R if needed)
- [x] No detail loss (edge-only smoothing preserves texture in flat regions)
- [x] Brightness balanced between channels (≤5% difference after equalization)
- [x] No pixel bleeding at split boundary (±1px guard zone)

### UI Controls

**Main Controls:**
- Split position slider: 45%-55%, step 0.5%
- Depth strength slider: 0.30-0.70, step 0.01
- Output mode tabs: Anaglyph / Wiggle
- Wiggle speed slider: 80-300ms (shown only in wiggle mode)

**Advanced Panel (collapsible):**
- Saturation suppression: -15% to -30%
- Contrast enhancement: +8% to +15%
- Edge smoothing: 0.20-0.30
- Anti-crosstalk: low / medium / high
- Auto depth correction: on / off toggle

**Pipeline Status Bar:**
- Three badges showing real-time status of: 校验 (validation), 方向 (direction), 后处理 (post-processing)
- Green = OK, Yellow = warning/auto-corrected

### Output Modes

1. **Anaglyph 红蓝立体** (primary): Red channel from left view, Cyan (G+B) from right view. View with standard red-cyan 3D glasses (left red, right blue/cyan).
2. **Wiggle 抖动动画** (glasses-free): Rapidly alternates left/right views at configurable speed. Brain fuses motion parallax into depth perception.

## Git

- Commit style uses Chinese commit messages describing feature additions and UI refinements
- The `index.html` file is the only source file
