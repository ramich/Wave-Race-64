# Wave Race 64 Decompilation — Remaining Work (DECOMP-RESUME)

> ## 📌 Project Pivot: PC Recompilation
>
> **As of 2026-05-27, the primary effort has pivoted to PC Recompilation** using the [N64Recomp](https://github.com/N64Recomp/N64Recomp) static recompilation approach. The recomp project lives at **[WACOMalt/WaveRace64-Recomp](https://github.com/WACOMalt/WaveRace64-Recomp)**.
>
> **Static recompilation is complete** (Phase 2) — all 1,228 functions have been recompiled from MIPS binary to native C code. The next step is runtime integration (Phase 3).
>
> The decomp work documented below **remains valuable** for:
> - **Symbol names** — 889 named functions feed directly into the recomp symbol table
> - **Struct definitions** — Player, audio, controller, and game state structs inform runtime integration
> - **Overlay mappings** — All 19 overlays are fully mapped and used in recomp configuration
> - **Mod support** — Named functions enable function-level hooking in the recompiled binary
>
> However, completing the remaining 476 functions to 100% is **no longer the critical path** to a playable PC port. The recomp approach handles undecompiled functions automatically by recompiling directly from the ROM binary.

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Critical Path Items (Must-Do First)](#2-critical-path-items-must-do-first)
3. [Audio Engine — 5 Functions Remaining](#3-audio-engine--5-functions-remaining-97--100)
4. [System Layer — 6 Functions Remaining](#4-system-layer--6-functions-remaining-92--100)
5. [Game Logic — ~148 Functions Remaining](#5-game-logic--148-functions-remaining-50-60--100)
6. [Codeseg — ~176 Functions Remaining](#6-codeseg--176-functions-remaining-15--100)
7. [Overlays — ~131 Functions Remaining](#7-overlays--131-functions-remaining-25--100)
8. [Miscellaneous Segments — ~11 Functions Remaining](#8-miscellaneous-segments--11-functions-remaining)
9. [NON_MATCHING Functions — 7 Functions Need Byte-Matching](#9-non_matching-functions--7-functions-need-byte-matching)
10. [Asset Pipeline — Currently <1% Complete](#10-asset-pipeline--currently-1-complete)
11. [Build System & Infrastructure Improvements](#11-build-system--infrastructure-improvements)
12. [Recommended Work Order (Critical Path)](#12-recommended-work-order-critical-path)
13. [Estimated Total Remaining Effort](#13-estimated-total-remaining-effort)

---

## 1. Executive Summary

- **Current state:** 889/1365 functions (65.71%)
- **Target state:** 1365/1365 functions (100%) + full asset pipeline + clean build
- **Estimated remaining:** ~476 functions across all subsystems
- **Key blockers:** Asset pipeline (unstarted), codeseg camera/math (hardest functions), struct naming

---

## 2. Critical Path Items (Must-Do First)

These items block other work:

### 2.1 Fix decomp.yaml Project Name

- **File:** `decomp.yaml` line 1 — change "Wonder Project J2" to "Wave Race 64"
- **Priority:** Trivial fix, do immediately
- **Difficulty:** ⬜ Trivial

### 2.2 Struct Reverse Engineering

- **File:** `include/structs.h` — 40+ structs with `UnkStruct_XXXXXXXX` naming
- The massive `UnkStruct_80192690` (100 lines, likely Player/Rider state) must be properly named and documented
- `UnkStruct_801C3C50` (complex nested struct) needs investigation
- `UnkStruct_8004B0F8` (water simulation object) — critical for `water_69D0.c` decompilation
- **Priority:** HIGH — blocks meaningful decompilation of game logic
- **Difficulty:** 🟥 Hard (requires extensive runtime analysis / memory dumps)

### 2.3 Complete Symbol Naming

- Most functions still have auto-generated names (`func_800XXXXX`)
- Audio symbols are well-named (SM64-derived), but game/codeseg/overlay functions are mostly unnamed
- **Priority:** MEDIUM — improves readability and decompilation speed
- **Difficulty:** 🟧 Medium (ongoing effort throughout project)

---

## 3. Audio Engine — 5 Functions Remaining (~97% → 100%)

### 3.1 audio_general.c — 5 GLOBAL_ASM stubs

- `func_800C010C` (line 408) — "chonker" — very large function, decomp.me scratch: https://decomp.me/scratch/cOjnQ
- `func_800C37F4` (line 1429) — "chonker" — very large function
- `func_800C3EF4` (line 1466) — "problematic switch" — switch statement mismatch with IDO
- `func_800BFD3C` (line ~668) — decomp.me scratch: https://decomp.me/scratch/io8AQ
- 1 additional unnamed stub
- **Difficulty:** 🟧 Medium-Hard (large functions, switch statement compiler quirks)
- **Estimated effort:** 2–4 days

---

## 4. System Layer — 6 Functions Remaining (~92% → 100%)

### 4.1 sys_main.c — 1 GLOBAL_ASM stub

- `SysMain_Thread` (line 106) — Main thread entry point
- **Difficulty:** 🟧 Medium
- **Estimated effort:** 0.5–1 day

### 4.2 sys_utils.c — 5 GLOBAL_ASM stubs

- System utility functions
- **Difficulty:** 🟧 Medium
- **Estimated effort:** 1–2 days

---

## 5. Game Logic — ~148 Functions Remaining (~50-60% → 100%)

This is the bulk of remaining work. Organized by file from most to fewest remaining stubs:

### 5.1 code_52CD0.c — 34 GLOBAL_ASM stubs (Race Logic/AI)

- Largest game logic file (2,196+ lines)
- Has `#ifndef NON_MATCHING` guards on 3 functions: `func_8009C2CC`, `func_800ADC8C`, `func_800ADF90`
- **Difficulty:** 🟥 Hard (AI logic, race state management)
- **Dependencies:** Struct naming (`UnkStruct_80192690`)
- **Estimated effort:** 2–3 weeks

### 5.2 code_C6C0.c — 30 GLOBAL_ASM stubs (Core Gameplay/Jet-Ski Physics)

- 1,827 lines, uses `LERP` macro
- Has `#ifndef NON_MATCHING` guard on `func_800534AC`
- **Difficulty:** 🟥 Hard (floating-point physics, vehicle dynamics)
- **Dependencies:** Water simulation structs, player state struct
- **Estimated effort:** 2–3 weeks

### 5.3 water_69D0.c — 19 GLOBAL_ASM stubs (Water Simulation)

- Only 1/20 functions decompiled (`func_8004E548`)
- Uses `TWO_OVER_SQRT_3` and `DIRLIGHT_DIRECTION` constants
- **Difficulty:** 🟥🟥 Very Hard (wave physics, fluid dynamics math)
- **Dependencies:** `UnkStruct_8004B0F8` naming
- **Estimated effort:** 2–4 weeks

### 5.4 wr64_save.c — 14 GLOBAL_ASM stubs (EEPROM Save/Load)

- ~70% done, EEPROM read/write logic
- **Difficulty:** 🟧 Medium (I/O patterns are well-documented in N64 SDK)
- **Estimated effort:** 1 week

### 5.5 code_43DA0.c — 11 GLOBAL_ASM stubs

- Has `#ifndef NON_MATCHING` guard on `func_80091DBC`
- **Difficulty:** 🟧 Medium
- **Estimated effort:** 1 week

### 5.6 code_24B00.c — 7 GLOBAL_ASM stubs

- **Difficulty:** 🟧 Medium
- **Estimated effort:** 3–5 days

### 5.7 code_4F850.c — 6 GLOBAL_ASM stubs

- Contains `unk_game_load` stub
- **Difficulty:** 🟧 Medium
- **Estimated effort:** 3–5 days

### 5.8 code_2FB10.c — 6 GLOBAL_ASM stubs

- **Difficulty:** 🟧 Medium
- **Estimated effort:** 3–5 days

### 5.9 code_68A10.c — 5 GLOBAL_ASM stubs

- **Difficulty:** 🟧 Medium
- **Estimated effort:** 2–3 days

### 5.10 code_2C670.c — 5 GLOBAL_ASM stubs

- **Difficulty:** 🟧 Medium
- **Estimated effort:** 2–3 days

### 5.11 code_5480.c — 3 GLOBAL_ASM stubs

- **Difficulty:** 🟨 Easy-Medium
- **Estimated effort:** 1–2 days

### 5.12 code_383C0.c — 2 GLOBAL_ASM stubs

- **Difficulty:** 🟨 Easy-Medium
- **Estimated effort:** 1 day

### 5.13 code_3A580.c — 2 GLOBAL_ASM stubs

- **Difficulty:** 🟨 Easy-Medium
- **Estimated effort:** 1 day

### 5.14 code_43A60.c — 2 GLOBAL_ASM stubs

- Has `#ifndef NON_MATCHING` guard on `func_80088EA8`
- **Difficulty:** 🟧 Medium
- **Estimated effort:** 1–2 days

### 5.15 code_3AC00.c — 1 GLOBAL_ASM stub

- Mostly decompiled (1,405+ lines), 1 stubborn function
- **Difficulty:** 🟧 Medium
- **Estimated effort:** 0.5–1 day

---

## 6. Codeseg — ~176 Functions Remaining (~15% → 100%)

### 6.1 A95D0.c — ~70 GLOBAL_ASM stubs (Camera/3D Math) ⚠️ HARDEST FILE

- Camera system and 3D transformation math
- Local structs defined but nearly all functions are ASM
- **Difficulty:** 🟥🟥 Very Hard (matrix math, camera logic, quaternions likely)
- **Dependencies:** `camera.h` needs expansion, struct naming
- **Estimated effort:** 4–8 weeks

### 6.2 AF8C0.c — ~37 GLOBAL_ASM stubs (includes ~32 data stubs)

- ~5 actual function stubs + ~32 `D_8022XXXX` data symbol stubs
- **Difficulty:** 🟧 Medium (data stubs are mechanical)
- **Estimated effort:** 1–2 weeks

### 6.3 B97B0.c — ~35 GLOBAL_ASM stubs

- Second-worst codeseg file
- Some C between ASM blocks
- **Difficulty:** 🟥 Hard
- **Estimated effort:** 3–5 weeks

### 6.4 B66E0.c — ~18 GLOBAL_ASM stubs

- Nearly all ASM
- **Difficulty:** 🟥 Hard
- **Estimated effort:** 1–2 weeks

### 6.5 B3AA0.c — 6 GLOBAL_ASM stubs

- Some decompiled code between stubs
- **Difficulty:** 🟧 Medium
- **Estimated effort:** 3–5 days

### 6.6 C94E0.c — 3 GLOBAL_ASM stubs

- **Difficulty:** 🟧 Medium
- **Estimated effort:** 1–2 days

### 6.7 B17D0.c — 2 GLOBAL_ASM stubs

- **Difficulty:** 🟨 Easy-Medium
- **Estimated effort:** 1 day

### 6.8 B3290.c — 2 GLOBAL_ASM stubs

- **Difficulty:** 🟨 Easy-Medium
- **Estimated effort:** 1 day

### 6.9 B2510.c — 1 GLOBAL_ASM stub

- Single function stub
- **Difficulty:** 🟨 Easy-Medium
- **Estimated effort:** 0.5 day

### 6.10 B2A70.c — 1 GLOBAL_ASM stub

- Comment: "Needs more context" — depends on surrounding code
- **Difficulty:** 🟧 Medium
- **Dependencies:** Adjacent file decompilation
- **Estimated effort:** 1–2 days

### 6.11 wr64_fade.c — 1 GLOBAL_ASM stub

- `func_801E7908` (line 299) — 86.02% on decomp.me (https://decomp.me/scratch/udOjy)
- **Difficulty:** 🟨 Easy (already 86% matched)
- **Estimated effort:** 0.5 day

---

## 7. Overlays — ~131 Functions Remaining (~25% → 100%)

Per-overlay breakdown:

| Overlay | File | ASM Stubs | Difficulty | Est. Effort |
|---------|------|-----------|------------|-------------|
| ovl_i0 | `ovl_1B3EC0.c` | 4 | 🟧 Medium | 2–3 days |
| ovl_i1 | `ovl_1B55A0.c` | 13 | 🟧 Medium | 1 week |
| ovl_i2 | `ovl_1B9440.c` | 13 | 🟧 Medium | 1 week |
| ovl_i3 | `ovl_1BC890.c` | 4 | 🟧 Medium | 2–3 days |
| ovl_i4 | `ovl_1BE0B0.c` | 7 | 🟧 Medium | 3–5 days |
| ovl_i5 | `ovl_1BFF50.c` + `ovl_1C17E0.c` | 6 + 10 | 🟧 Medium | 1–2 weeks |
| ovl_i6 | `ovl_1C2250.c` | 4 | 🟧 Medium | 2–3 days |
| ovl_i7 | `ovl_1C43F0.c` | 1 | 🟨 Easy | 0.5 day |
| ovl_i8 | `ovl_1C49A0.c` | 8 | 🟧 Medium | 3–5 days |
| ovl_i9 | `ovl_1C66D0.c` | 7 | 🟧 Medium | 3–5 days |
| ovl_i10 | `ovl_1C9150.c` | 4 | 🟧 Medium | 2–3 days |
| ovl_i11 | `ovl_1CA480.c` | 2 | 🟨 Easy | 1 day |
| ovl_i12 | `ovl_1CAE40.c` | 2 | 🟨 Easy | 1 day |
| ovl_i13 | `ovl_1CBAF0.c` | 20 | 🟥 Hard | 2–3 weeks |
| ovl_i14 | `ovl_1CF180.c` | 2 | 🟨 Easy | 1 day |
| ovl_i15 | `ovl_1CFB60.c` | 4 | 🟧 Medium | 2–3 days |

> **Note:** ovl_i0's `func_i0_802C6AE4` is "Matched but needs data migration"

---

## 8. Miscellaneous Segments — ~11 Functions Remaining

### 8.1 unused_code_1B1FB0.c — 10 GLOBAL_ASM stubs

- Unused code at VRAM `0x802C5800` (same as overlay base)
- 3 C functions already done, 10 ASM stubs remain
- **Difficulty:** 🟧 Medium (but low priority — code is unused)
- **Estimated effort:** 3–5 days

### 8.2 seg_1C3D00.c — 1 GLOBAL_ASM stub

- Small segment, mostly decompiled
- **Difficulty:** 🟨 Easy
- **Estimated effort:** 0.5 day

---

## 9. NON_MATCHING Functions — 7 Functions Need Byte-Matching

These functions are decompiled but don't produce byte-identical output with IDO 5.3:

| Function | File | Notes |
|----------|------|-------|
| `func_800534AC` | `code_C6C0.c:671` | Core gameplay |
| `func_8009C2CC` | `code_52CD0.c:1318` | Race logic |
| `func_800ADC8C` | `code_52CD0.c:2054` | Race logic |
| `func_800ADF90` | `code_52CD0.c:2196` | Race logic |
| `func_80088EA8` | `code_43A60.c:71` | Game code |
| `func_80091DBC` | `code_43DA0.c:431` | Game code |
| `func_i1_802C8E70` | `ovl_1B55A0.c:165` | Overlay i1 |

- **Difficulty:** 🟥 Hard (requires IDO compiler quirk knowledge, register allocation tricks)
- **Estimated effort:** 1–2 days per function

---

## 10. Asset Pipeline — Currently <1% Complete

This is the largest infrastructure gap in the project:

### 10.1 Asset Discovery & Cataloging

- Only 1 GFX symbol defined in `ast_1E5860.yaml` (`D_1002DC0`)
- Need to identify ALL assets in the ROM: textures, models, course data, audio banks, animations
- **Difficulty:** 🟥 Hard (requires ROM analysis tools, binary inspection)
- **Estimated effort:** 2–4 weeks

### 10.2 Texture Extraction

- CI4, CI8, RGBA16, RGBA32, I4, I8, IA4, IA8, IA16 formats expected
- Need YAML definitions for every texture
- **Difficulty:** 🟧 Medium (tooling exists, identification is the hard part)
- **Estimated effort:** 2–3 weeks

### 10.3 Model/Display List Extraction

- F3D microcode display lists
- Torch (HarbourMasters) integration is stub-level
- Need complete GFX symbol coverage
- **Difficulty:** 🟥 Hard
- **Estimated effort:** 3–4 weeks

### 10.4 Course Data

- 9 courses defined in `course.h` (`SUNNY_BEACH` through `SOUTHERN_ISLAND`)
- Track geometry, checkpoint data, spline paths, buoy positions all need extraction
- No course data structures exist yet
- **Difficulty:** 🟥🟥 Very Hard (custom formats, need reverse engineering)
- **Estimated effort:** 4–8 weeks

### 10.5 Audio Assets

- Sound banks, sequence data, instrument definitions
- Audio code is nearly done but audio data assets need extraction
- **Difficulty:** 🟧 Medium (audio format is well-understood from SM64)
- **Estimated effort:** 1–2 weeks

### 10.6 MIO0 Compressed Segments

- Multiple MIO0-compressed segments in ROM
- Tools exist (`libmio0.c`, `mio0_decompress.py`) but pipeline isn't automated
- Need to identify and decompress all MIO0 blocks
- **Difficulty:** 🟧 Medium (tooling exists)
- **Estimated effort:** 1 week

---

## 11. Build System & Infrastructure Improvements

### 11.1 CI/CD Pipeline

- `.github/` directory exists but CI configuration status is unknown
- Need GitHub Actions for automated build verification
- **Difficulty:** 🟨 Easy-Medium
- **Estimated effort:** 1–2 days

### 11.2 Progress Tracking

- `decomp.yaml` progress categories: game, audio, codeseg, sys, overlays, libultra
- `tools/progress.py` exists but may need updating
- **Difficulty:** 🟨 Easy
- **Estimated effort:** 0.5 day

### 11.3 Documentation

- Header files need documentation comments
- Function purposes need documenting as they're decompiled
- **Difficulty:** 🟨 Easy (ongoing)

---

## 12. Recommended Work Order (Critical Path)

1. **Phase 1 — Quick Wins** (1–2 weeks)
   - Fix `decomp.yaml` name
   - `wr64_fade.c` last function (86% done)
   - `ovl_i7` (1 stub), `ovl_i11` (2 stubs), `ovl_i12` (2 stubs), `ovl_i14` (2 stubs)
   - `seg_1C3D00.c` (1 stub)
   - `code_3AC00.c` (1 stub)

2. **Phase 2 — Audio Completion** (1–2 weeks)
   - `audio_general.c` (5 stubs) — finish the audio engine to 100%

3. **Phase 3 — System Layer Completion** (1 week)
   - `sys_main.c` (1 stub) + `sys_utils.c` (5 stubs)

4. **Phase 4 — Easy-Medium Game Functions** (3–4 weeks)
   - `code_383C0.c`, `code_3A580.c`, `code_5480.c`, `code_43A60.c`
   - `code_2C670.c`, `code_68A10.c`, `code_2FB10.c`, `code_24B00.c`
   - `wr64_save.c`

5. **Phase 5 — Struct Naming & Documentation** (2–3 weeks, parallel with Phase 4)
   - Reverse engineer `UnkStruct_80192690` (player state)
   - Name all fields in critical structs
   - Document struct purposes

6. **Phase 6 — Hard Game Functions** (4–8 weeks)
   - `code_C6C0.c` (physics), `code_52CD0.c` (race/AI), `water_69D0.c` (water sim)
   - `code_43DA0.c`, `code_4F850.c`

7. **Phase 7 — Codeseg** (8–16 weeks)
   - Start with `wr64_fade.c`, `C94E0.c`, `B17D0.c`, `B3290.c` (easy)
   - Progress to `B3AA0.c`, `B2A70.c`, `AF8C0.c` (medium)
   - Tackle `B66E0.c`, `B97B0.c` (hard)
   - `A95D0.c` last (hardest file in project — camera/3D math)

8. **Phase 8 — Overlays** (4–8 weeks, can parallel with Phase 7)
   - Easy overlays first: i7, i11, i12, i14
   - Medium: i0, i3, i6, i10, i15, i4, i8, i9
   - Hard: i1, i2, i5, i13

9. **Phase 9 — NON_MATCHING Resolution** (1–2 weeks)
   - Fix all 7 `NON_MATCHING` functions

10. **Phase 10 — Asset Pipeline** (8–16 weeks)
    - MIO0 decompression automation
    - Texture extraction and YAML definitions
    - Model/display list extraction
    - Course data reverse engineering
    - Audio asset extraction

11. **Phase 11 — Final Integration** (2–4 weeks)
    - Build system cleanup
    - CI/CD verification
    - Full ROM comparison verification
    - Documentation completion

---

## 13. Estimated Total Remaining Effort

| Area | Functions | Estimated Weeks |
|------|-----------|-----------------|
| Audio | 5 | 1–2 |
| System | 6 | 1–2 |
| Game Logic | ~148 | 8–16 |
| Codeseg | ~176 | 8–16 |
| Overlays | ~131 | 4–8 |
| NON_MATCHING | 7 | 1–2 |
| Misc Segments | 11 | 1 |
| Asset Pipeline | N/A | 8–16 |
| Struct/Symbol Naming | N/A | 2–3 |
| Build/CI/Docs | N/A | 1–2 |
| **TOTAL** | **~476** | **~35–67 weeks** |

For a solo developer: **8–16 months**
For a small team (3–4): **3–6 months**
