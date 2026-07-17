# Runtime-verified findings (from the WaveRace64-Recomp PC port)

Facts below were established by **runtime instrumentation** of the recompiled
PC port ([ramich/WaveRace64-Recomp](https://github.com/ramich/WaveRace64-Recomp),
`docs/RE-NOTES.md` has the full trail): live RDRAM peeking/poking during real
races, RT64 display-list telemetry, and A/B screenshot verification. They are
offered here as ground truth for naming/matching work — static decomp can
rarely prove "the game re-reads this struct every frame", but the port did.

## Water rendering pipeline

- **`func_8008FB74` (code_43DA0.c) is the wave-mesh display-list builder** —
  the per-frame generator of the detail-water grid (the water surface with
  foam). Call chain each frame: `func_8008FB74 → func_800949B8 (per-course) →
  Draw_WaterEffects → overlay scene draw`. It begins by copying a 12-byte
  config block from `D_800DA8B4` to its stack and calls into the water
  simulation (`func_8004C998`, `func_8004C1D0` — water_69D0.c) for wave
  heights.
- **`D_800DA8B4` is the wave-grid config block**:
  `{ u32 flag(=1), u32 gridRows(=19), u32 gridCols(=35), then Vp viewport
  structs — vscale/vtrans (640,480,511,0) = full-screen, followed by the
  splitscreen viewport variants (0x02800108 / 0x028002C0 vtrans values) }`.
  The builder re-reads rows/cols **every frame**: live-poking them instantly
  resizes the rendered water grid (verified stable up to 27x63; the PC port
  ships 23x55 for widescreen coverage). 19x35 covers exactly the original
  4:3 view rect — this grid is why the detail water ends in a screen-space
  rectangle.
- The camera-facing grid + per-frame rebuild means the "wave rectangle" seen
  in widescreen hacks is grid sizing, not scissoring or culling.

## Camera

- **Camera struct offset `unk88` is fovy in degrees** (the `= 45.0f` writes
  throughout code_52CD0.c). Confirmed by patching individual `45.0f`
  instruction constants and watching the projection matrix (m11 = cot(fovy/2))
  in RT64 telemetry.
- **`func_800A52D8` sets up the actual in-race camera** — of all the
  inline-45.0f sites, only patching the constant at `0x800A54E4` (inside this
  function) changes the live race projection. Its three sibling 45.0f sites
  (`0x800A5648`, `0x800A57AC`, `0x800A57B8`) are believed to be the other
  C-button view modes (untested).
- The dispatch battery `func_8009B910/..BA14/..BAE0/..BB98/..BCA4` sites and
  the 16-entry stride-0x30 setter army (`0x8009BEBC..0x8009C1C4`) never
  affected the projection in attract demos **or** live races — they set
  something else (camera angles per course/mode, or dead paths).
- The game applies **camera bob by translating its full-size RSP viewport**
  per frame (vscale stays (640,480); vtrans wanders, e.g. (504,296)); the
  viewport is never sized to the inner view rect.

## Screen tint / atmosphere overlay

- **`func_801E7C58` (wr64_fade.c, already matched) draws the fullscreen
  atmosphere tint** as a texrect; `func_801E7908` calls it with the view rect
  as immediates — two rect variants: `(8,20)-(310,218)` (stack bottom 0xDA)
  and `(8,12)-(310,228)` (stack bottom 0xE4), colors from four s16 globals
  around `0x80228A20` (clamped to 255 inside).

## View-rect / clipping notes (open)

- The game **CPU-clips its large ground polygons (beach/shore strip, banner
  cloth) to the original view rect** before building the DL — with every
  GPU-side scissor verified full-frame, those polys still cut at fb x≈9.
  The clip constants are NOT: any s16 (8,310)/(20,218)-style pair in RDRAM
  (all live candidates poked, no effect), integer immediates 310/218/302/198
  (only the tint builder uses them), or float immediates 310.0f/218.0f (no
  hits). Finding this clipper (likely in the course-geometry draw path,
  `func_800949B8` and the ovl_i* scene overlays) is the main open item.
- The guFrustum-feeding struct array at `0x801D7B70` (stride 0x24, active
  when +0x00 != 0) is **fully inactive during attract demos and real races**
  on Sunny Beach — whatever uses it, it is not the race view.
- `gDPSetScissor(8,20,310,218)`-building functions: `func_80093DBC`
  (code_4C750.c, matched) plus per-scene overlay copies; the wave-mesh passes
  set the same scissor again from branched sub-DLs.
