# Wave Race 64 — Technical Specification

> **Decompilation Project · US Rev 1 (V1.1)**
> SHA-1: `508dfc2d4caa42b6f6de5263d0aed5e44ac7966a`

---

## Table of Contents

1. [Project Overview](#1-project-overview)
   - 1.1 [Target ROM Information](#11-target-rom-information)
   - 1.2 [Project Goals](#12-project-goals)
   - 1.3 [Current Status Summary](#13-current-status-summary)
2. [Build System](#2-build-system)
   - 2.1 [Toolchain Requirements](#21-toolchain-requirements)
   - 2.2 [Compiler Configuration](#22-compiler-configuration)
   - 2.3 [Build Pipeline](#23-build-pipeline)
   - 2.4 [Build Targets](#24-build-targets)
   - 2.5 [Compiler Flags & ISA Variants](#25-compiler-flags--isa-variants)
3. [Architecture](#3-architecture)
   - 3.1 [Memory Map](#31-memory-map)
   - 3.2 [Thread Model](#32-thread-model)
   - 3.3 [Overlay System](#33-overlay-system)
   - 3.4 [Game Initialization Flow](#34-game-initialization-flow)
4. [Decompilation Progress by Subsystem](#4-decompilation-progress-by-subsystem)
   - 4.1 [Audio Engine](#41-audio-engine)
   - 4.2 [System Layer](#42-system-layer)
   - 4.3 [Game Logic](#43-game-logic)
   - 4.4 [Codeseg](#44-codeseg)
   - 4.5 [Overlays](#45-overlays)
   - 4.6 [libultra](#46-libultra)
   - 4.7 [Miscellaneous Segments](#47-miscellaneous-segments)
5. [Data Structures](#5-data-structures)
   - 5.1 [Core Game Structs](#51-core-game-structs)
   - 5.2 [Audio Structs](#52-audio-structs)
   - 5.3 [Game Enums](#53-game-enums)
   - 5.4 [Key Type Definitions](#54-key-type-definitions)
6. [Symbol Coverage](#6-symbol-coverage)
   - 6.1 [Named vs Unnamed Functions](#61-named-vs-unnamed-functions)
   - 6.2 [Symbol Files](#62-symbol-files)
   - 6.3 [Best-Covered Areas](#63-best-covered-areas)
   - 6.4 [Worst-Covered Areas](#64-worst-covered-areas)
7. [Asset Pipeline](#7-asset-pipeline)
   - 7.1 [MIO0 Compression](#71-mio0-compression)
   - 7.2 [Torch/F3D Integration](#72-torchf3d-integration)
   - 7.3 [Current Asset Definitions](#73-current-asset-definitions)
   - 7.4 [Asset Tools](#74-asset-tools)
8. [Known Issues & Technical Debt](#8-known-issues--technical-debt)
   - 8.1 [NON_MATCHING Guards](#81-non_matching-guards)
   - 8.2 [decomp.yaml Vestigial Name](#82-decompyaml-vestigial-name)
   - 8.3 [Dual Overlay Table](#83-dual-overlay-table)
   - 8.4 [Data Migration Issues](#84-data-migration-issues)
   - 8.5 [decomp.me Scratch Links](#85-decompme-scratch-links)
9. [Tools & Utilities](#9-tools--utilities)
   - 9.1 [asm-differ](#91-asm-differ)
   - 9.2 [asm-processor](#92-asm-processor)
   - 9.3 [splat](#93-splat)
   - 9.4 [MIO0 Tools](#94-mio0-tools)
   - 9.5 [Code Formatting](#95-code-formatting)

---

## 1. Project Overview

### 1.1 Target ROM Information

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| **Game**           | Wave Race 64                                               |
| **Platform**       | Nintendo 64 (NUS-NWRE-USA)                                 |
| **Region/Version** | US, Revision 1 (V1.1)                                      |
| **SHA-1**          | `508dfc2d4caa42b6f6de5263d0aed5e44ac7966a`                 |
| **Baserom file**   | `baserom.us.rev1.z64`                                      |
| **Output ROM**     | `build/waverace64.us.rev1.z64`                             |
| **GBI Version**    | Fast3D (F3D_OLD), defined via [`-DF3D_OLD`](Makefile:267)  |
| **CPU**            | NEC VR4300 (MIPS III subset, compiled as MIPS II)          |
| **Resolution**     | 320×240, defined in [`macros.h`](include/macros.h:86)      |

Other ROM versions are partially stubbed for future work:

| Version | Status            | Rev    | Build Define     |
|---------|-------------------|--------|------------------|
| `jp`    | Planned           | rev1   | `VERSION_JP`     |
| `us`    | **Active target** | rev1   | `VERSION_US`     |
| `eu`    | Planned           | rev0   | `VERSION_EU`     |
| `au`    | Planned           | rev0   | `VERSION_AU`     |
| `ln`    | Planned           | rev0   | `VERSION_LN`     |

### 1.2 Project Goals

1. **Byte-matched decompilation** — Produce a ROM binary identical to the original US Rev 1 retail ROM, verified via SHA-1 checksum comparison.
2. **Readable C source** — Translate all MIPS assembly back into human-readable C compiled with the original IDO 5.3 toolchain.
3. **Documentation** — Recover function names, struct layouts, enum semantics, and game logic through community reverse-engineering.
4. **Modding foundation** — Enable future mod support through the Torch asset pipeline and overlay architecture.

> **📌 PC Recompilation Project:** Based on the analysis documented here, a PC port effort has been started using N64Recomp static recompilation. The recomp project lives at [WACOMalt/WaveRace64-Recomp](https://github.com/WACOMalt/WaveRace64-Recomp). As of 2026-06-01, **static recompilation is complete** — all 1,228 functions have been recompiled from MIPS binary to C. The decomp work here provides the symbol names, struct definitions, and overlay mappings that make the recomp possible. See [PC-RECOMP-PLAN.md](PC-RECOMP-PLAN.md) for details.

### 1.3 Current Status Summary

```
Overall Progress: 65.71% — 889 / 1,365 functions decompiled
                  476 GLOBAL_ASM stubs remaining
```

| Subsystem   | Est. Completion | Remaining ASM Stubs |
|-------------|-----------------|---------------------|
| Audio       | ~97%            | 5                   |
| System      | ~92%            | 6                   |
| Game Logic  | ~50–60%         | ~148                |
| Codeseg     | ~15%            | ~176                |
| Overlays    | ~25%            | ~131                |
| Misc/Other  | ~30%            | ~10                 |

---

## 2. Build System

### 2.1 Toolchain Requirements

| Component              | Requirement                             | Path / Notes                                                   |
|------------------------|-----------------------------------------|----------------------------------------------------------------|
| **C Compiler**         | IDO 5.3 (statically recompiled)         | [`tools/ido-static-recomp/build/5.3/out/cc`](Makefile:204)    |
| **Assembler**          | `mips-linux-gnu-as` (GNU binutils)      | Auto-detected prefix in [`Makefile`](Makefile:39)              |
| **Linker**             | `mips-linux-gnu-ld`                     | [`Makefile:206`](Makefile:206)                                 |
| **objcopy**            | `mips-linux-gnu-objcopy`                | [`Makefile:207`](Makefile:207)                                 |
| **Python**             | Python 3 (with venv support)            | [`Makefile:59`](Makefile:59)                                   |
| **C Preprocessor**     | `clang` (preferred) or `cpp`            | [`Makefile:215`](Makefile:215)                                 |
| **GCC (alt compiler)** | `mips-linux-gnu-gcc` (experimental)     | Activated via `COMPILER=gcc`                                   |
| **Splat**              | Python-based ROM splitter               | [`tools/splat/split.py`](Makefile:226)                         |
| **n64crc**             | N64 ROM CRC fixer                       | [`tools/n64crc`](Makefile:212)                                 |
| **Torch**              | Asset extraction/import tool            | [`tools/Torch/cmake-build-release/torch`](Makefile:211)        |

Supported host toolchain prefixes (auto-detected):
- `mips-linux-gnu-`
- `mips64-linux-gnu-`
- `mips64-elf-`

### 2.2 Compiler Configuration

**Primary compiler: IDO 5.3** — The original Silicon Graphics compiler, recompiled to run natively on modern Linux via `ido-static-recomp`.

```makefile
# IDO compiler invocation (wrapped through asm-processor)
CC := python3 tools/asm-processor/build.py \
      --input-enc=utf-8 --output-enc=euc-jp --convert-statics=global-with-filename \
      tools/ido-static-recomp/build/5.3/out/cc -- \
      mips-linux-gnu-as -march=vr4300 -32 -G0 --
```

**Experimental GCC support** — Enabled via `COMPILER=gcc`. Sets `NON_MATCHING=1` automatically since GCC cannot reproduce IDO's exact code generation. Relevant flags from [`Makefile:93`](Makefile:93):

```
-G 0 -ffast-math -fno-unsafe-math-optimizations -march=vr4300 -mfix4300
-mabi=32 -mno-abicalls -mdivide-breaks -fno-zero-initialized-in-bss
-fno-toplevel-reorder -ffreestanding -fno-common -fno-merge-constants
-mno-explicit-relocs -mno-split-addresses -funsigned-char
```

### 2.3 Build Pipeline

```
┌─────────┐    ┌───────┐    ┌───────────────┐    ┌──────┐    ┌─────────┐    ┌───────┐
│ baserom │───>│ splat │───>│ asm/ + bin/    │    │      │    │         │    │       │
│  .z64   │    │       │    │ (extracted)    │    │      │    │         │    │       │
└─────────┘    └───────┘    └───────┬───────┘    │      │    │         │    │       │
                                    │            │      │    │         │    │       │
                            ┌───────▼───────┐    │      │    │         │    │       │
                            │  .s / .c / .bin│──>│  IDO │──>│   ld    │──>│objcopy│──>n64crc──> .z64
                            │  source files │    │ 5.3  │    │ (link) │    │(bin)  │
                            └───────────────┘    │      │    │         │    │       │
                                                 │      │    │         │    │       │
                            ┌───────────────┐    │      │    │         │    │       │
                            │ linker scripts│───>│      │───>│         │    │       │
                            │  (.ld files)  │    └──────┘    └─────────┘    └───────┘
                            └───────────────┘
```

**Step-by-step:**

1. **Extract** (`make extract`) — Runs `splat` on the YAML config to split the ROM into assembly (`.s`), binary (`.bin`), and generates linker scripts.
2. **Preprocess** — C preprocessor (`clang -E` or `cpp`) processes assembly files and linker scripts.
3. **Compile** — IDO 5.3 (via `asm-processor`) compiles `.c` files; GNU `as` assembles `.s` files; `objcopy` converts `.bin` to ELF objects.
4. **Link** — `mips-linux-gnu-ld` links all objects using the generated linker script plus symbol resolution files.
5. **Convert** — `objcopy -O binary` converts ELF to raw ROM binary.
6. **Fix CRC** — `n64crc` patches the ROM header checksums.
7. **Verify** — SHA-1 comparison against `waverace64.us.rev1.sha1`.

### 2.4 Build Targets

| Target          | Command              | Description                                        |
|-----------------|----------------------|----------------------------------------------------|
| `all` / `finalrom` | `make`            | Full matching build with SHA-1 verification        |
| `extract`       | `make extract`       | Split ROM via splat + MIO0 extraction              |
| `clean`         | `make clean`         | Remove all build artifacts                         |
| `init`          | `make init`          | Clean → extract → build (full from scratch)        |
| `expected`      | `make expected`      | Copy current build to `expected/` for asm-differ   |
| `format`        | `make format`        | Run clang-format via [`tools/format.py`](tools/format.py) |
| `checkformat`   | `make checkformat`   | Verify formatting compliance                       |
| `context`       | `make context FILE`  | Generate `ctx.c` for decomp.me context             |
| `assets`        | `make assets`        | Extract assets via Torch                           |
| `torch`         | `make torch`         | Build the Torch tool                               |
| `disasm`        | `make disasm`        | Full disassembly of all segments                   |
| `toolchain`     | `make toolchain`     | Build just the toolchain components                |
| `dependencies`  | `make dependencies`  | Install Python deps and build tools                |

### 2.5 Compiler Flags & ISA Variants

**Default IDO flags** (from [`Makefile:97`](Makefile:97)):

```
-G 0 -non_shared -fullwarn -verbose -Xcpluscomm -nostdinc
-Wab,-r4300_mul -woff 649,838,712,516
-mips2 -32 -O2
```

**Per-directory optimization overrides:**

| Path Pattern                     | `-O`  | ISA        |
|----------------------------------|-------|------------|
| `src/libultra/os/*.c`            | `-O1` | `-mips2`   |
| `src/libultra/gu/*.c`            | `-O3` | `-mips2`   |
| `src/libultra/io/*.c`            | `-O1` | `-mips2`   |
| `src/libultra/audio/*.c`         | `-O3` | `-mips2`   |
| `src/libultra/libc/*.c`          | `-O1` | `-mips2`   |
| `src/libultra/libc/ll.c`         | `-O1 -g0` | `-mips3 -32` |
| `src/libultra/io/pfs*.c`         | `-O2` | `-mips1`   |
| `src/libultra/io/contram*.c`     | `-O2` | `-mips1`   |
| `src/libultra/io/controller.c`   | `-O2` | `-mips1`   |
| Game/audio/codeseg/overlay code  | `-O2` | `-mips2`   |

**Key C defines** (from [`Makefile:266`](Makefile:266)):

```c
-D_FINALROM -DCOMPILING_LIBULTRA -DTARGET_N64
-DSSSV -DWIN32 -DLANGUAGE_C -D_LANGUAGE_C -DNDEBUG
-DF3D_OLD  // Fast3D microcode variant
```

**ASM-processor flags** (from [`Makefile:224`](Makefile:224)):

```
--input-enc=utf-8 --output-enc=euc-jp --convert-statics=global-with-filename
```

The `--convert-statics=global-with-filename` flag is critical for handling IDO's static variable emission correctly during incremental decompilation.

---

## 3. Architecture

### 3.1 Memory Map

```
┌──────────────────────────────────────────────────────┐
│  N64 Virtual Address Space (KSEG0: 0x80000000+)     │
├──────────────────────────────────────────────────────┤
│  0x80000000 ┌──────────────────────────────────┐     │
│             │  System / Boot code              │     │
│  0x80005480 ├──────────────────────────────────┤     │
│             │  Main Segment (TEXT)             │     │
│             │  Game code: code_5480.c ...      │     │
│             │  Audio: audio_effects.c ...      │     │
│             │  System: main.c, sys_*.c         │     │
│  0x801D0FA0 ├──────────────────────────────────┤     │
│             │  Codeseg (A95D0.c ... C94E0.c)   │     │
│             │  Camera, 3D math, rendering      │     │
│  0x801FC4D4+├──────────────────────────────────┤     │
│             │  libultra (precompiled ASM)       │     │
│  ...        ├──────────────────────────────────┤     │
│  0x802C5800 │  ████ OVERLAY VRAM ████          │     │
│             │  (19 overlays, shared region)     │     │
│  0x802C9700 │  (approx end of largest ovl)     │     │
│             ├──────────────────────────────────┤     │
│             │  BSS / Stack / Heap              │     │
└──────────────────────────────────────────────────────┘
```

Key addresses from [`src/ovl_table.c`](src/ovl_table.c:28):

| Segment                | ROM Start    | ROM End      | VRAM Start   | VRAM End     |
|------------------------|-------------|-------------|-------------|-------------|
| `segment_1B1FB0`       | `0x001B1FB0` | `0x001B3EC0` | `0x802C5800` | `0x802C7730` |
| `ovl_i0` (boot up)     | `0x001B3EC0` | `0x001B55A0` | `0x802C5800` | `0x802C6F20` |
| `ovl_i1` (main menu)   | `0x001B55A0` | `0x001B9440` | `0x802C5800` | `0x802C96E0` |
| `ovl_i13` (audio opts) | `0x001CBAF0` | `0x001CF180` | `0x802C5800` | `0x802C8EC0` |

All 19 overlays share the same VRAM base address: **`0x802C5800`**.

### 3.2 Thread Model

Wave Race 64 uses the standard N64 multi-threaded architecture via libultra's cooperative threading:

```
┌─────────────────────────────────────────────┐
│              THREAD HIERARCHY               │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────────┐   Priority: Lowest        │
│  │  Idle Thread  │   Bootstraps system,      │
│  │  (Thread 1)   │   launches Main thread    │
│  └──────┬───────┘                           │
│         │                                   │
│  ┌──────▼───────┐   Priority: Medium        │
│  │  Main Thread  │   Game logic, rendering,  │
│  │  (Thread 3)   │   overlay management      │
│  │  SysMain_Thread() in sys_main.c          │
│  └──────┬───────┘                           │
│         │                                   │
│  ┌──────▼───────┐   Priority: High          │
│  │ Audio Thread  │   Synthesizes audio,      │
│  │  (Thread 4)   │   processes audio cmds    │
│  │  AudioThread_Init() → audio_thread.c     │
│  └──────────────┘                           │
│                                             │
│  ┌──────────────┐   Priority: Highest       │
│  │ Graphics /    │   VI retrace handler,     │
│  │ Scheduler     │   task scheduling         │
│  └──────────────┘                           │
│                                             │
└─────────────────────────────────────────────┘
```

Key entry points:

- **Idle/Boot**: Initial entry → sets up OS, spawns main thread
- **Main Thread**: [`SysMain_Thread()`](src/sys/sys_main.c:105) — Game loop, state machine, overlay dispatch
- **Audio Thread**: [`AudioThread_Init()`](src/audio/audio_thread.c) → [`AudioThread_CreateTask()`](include/wr64audio.h:936)
- **Graphics**: Task submission via [`GfxPool`](include/structs.h:356) double-buffering

### 3.3 Overlay System

The game uses **19 overlays** loaded into a shared VRAM region at **`0x802C5800`**. Only one overlay is active at a time. The overlay table is defined in [`src/ovl_table.c`](src/ovl_table.c:28) as `gOverlayTable[]`.

**Overlay ID enum** (from [`include/wr64ovl.h`](include/wr64ovl.h:4)):

| ID  | Enum Name                      | Description              | Overlay Dir |
|-----|--------------------------------|--------------------------|-------------|
| 0   | `OVL_BOOT_UP`                 | Boot/startup             | `ovl_i0`    |
| 1   | `OVL_TITLE_SCREEN`            | Title screen             | `ovl_i1`    |
| 2   | `OVL_RIDER_SELECT`            | Rider selection          | `ovl_i2`    |
| 3   | `OVL_COURSE_OVERVIEW`         | Course overview          | `ovl_i3`    |
| 4   | `OVL_COURSE_SELECT`           | Course selection         | `ovl_i4`    |
| 5   | `OVL_RACE_RESULTS`            | Race results screen      | `ovl_i5`    |
| 6   | `OVL_6`                       | Unknown                  | `ovl_i6`    |
| 7   | `OVL_TIME_TRIALS_RESULTS`     | Time trial results       | `ovl_i7`    |
| 8   | `OVL_STUNT_MODE_RESULTS`      | Stunt mode results       | `ovl_i8`    |
| 9   | `OVL_OPTIONS_MENU`            | Options menu             | `ovl_i9`    |
| 10  | `OVL_OPTIONS_CHANGE_NAMES`    | Change names             | `ovl_i10`   |
| 11  | `OVL_OPTIONS_VIEW_RECORDS`    | View records             | `ovl_i11`   |
| 12  | `OVL_OPTIONS_CHANGE_CONDITIONS` | Change conditions      | `ovl_i12`   |
| 13  | `OVL_OPTIONS_AUDIO`           | Audio settings           | `ovl_i13`   |
| 14  | `OVL_OPTIONS_ERASE_COURSE_RECORDS` | Erase records       | `ovl_i14`   |
| 15  | `OVL_OPTIONS_SAVE_AND_LOAD`   | Save & load              | `ovl_i15`   |
| 16  | `OVL_16`                      | Unknown                  | —           |
| 17  | `OVL_CEREMONY`                | Award ceremony           | —           |
| 18  | `OVL_DEMO` / `OVL_TIME_TRIAL` | Demo playback / Time trial (shared) | — |

> **Note:** `OVL_DEMO` and `OVL_TIME_TRIAL` share the same ID (18), as noted in [`wr64ovl.h:23`](include/wr64ovl.h:23).

**Overlay entry structure** (from `gOverlayTable[]`):

```c
typedef struct {
    u32 romStart;   // ROM offset start
    u32 romEnd;     // ROM offset end
    u32 textStart;  // VRAM .text start  (always 0x802C5800)
    u32 textEnd;    // VRAM .text end
    u32 dataStart;  // VRAM .data start
    u32 dataEnd;    // VRAM .data end
    u32 bssStart;   // VRAM .bss start
    u32 bssEnd;     // VRAM .bss end
} Overlay;
```

### 3.4 Game Initialization Flow

```
Boot
 │
 ├─> osInitialize()
 ├─> Create Idle Thread
 │    └─> Create Main Thread
 │         └─> SysMain_Thread()          [sys_main.c]
 │              ├─> Hardware init (VI, PI, AI)
 │              ├─> AudioThread_Init()    [audio_thread.c]
 │              ├─> Load overlay (OVL_BOOT_UP)
 │              │    └─> 0x802C5800 region
 │              ├─> Game State Machine
 │              │    ├─> GAME_STATE_BOOT_UP (0x05)
 │              │    ├─> GAME_STATE_TITLE_SCREEN (0x02)
 │              │    ├─> GAME_STATE_MAIN_MENU (0x03)
 │              │    ├─> GAME_STATE_RIDER_SELECT (0x0A)
 │              │    ├─> GAME_STATE_COURSE_SELECT (0x14)
 │              │    ├─> GAME_STATE_TIME_TRIAL (0x28)
 │              │    └─> ... (see GameState enum in game.h)
 │              └─> Per-frame loop:
 │                   ├─> SysUtils_UpdateControllers()
 │                   ├─> Game logic tick
 │                   ├─> GFX task build
 │                   └─> Audio task scheduling
 │
 └─> Audio Thread (started by AudioThread_Init)
      └─> AudioThread_CreateTask()
           ├─> AudioSynth_Update()
           ├─> AudioSeq_ProcessSequences()
           └─> AudioThread_ProcessCmds()
```

The [`GameState`](include/game.h:28) enum drives overlay loading and the main state machine, with transitions managed through `gGameState` and `gPrevGameState`.

---

## 4. Decompilation Progress by Subsystem

### 4.1 Audio Engine

**Estimated completion: ~97%** — 5 remaining `GLOBAL_ASM` stubs

The audio engine is derived from **Super Mario 64's** audio system, with function naming conventions preserved (e.g., `Nas_*`, `AudioHeap_*`, `AudioSeq_*`, `AudioThread_*`).

**Source files** (all in [`src/audio/`](src/audio)):

| File | Status | ASM Stubs | Notes |
|------|--------|-----------|-------|
| [`audio_effects.c`](src/audio/audio_effects.c) | ✅ Complete | 0 | ADSR, vibrato, portamento |
| [`audio_general.c`](src/audio/audio_general.c) | ⚠️ 5 remaining | 5 | 2 "chonkers", 1 "problematic switch" |
| [`audio_heap.c`](src/audio/audio_heap.c) | ✅ Complete | 0 | Memory pool management |
| [`audio_load.c`](src/audio/audio_load.c) | ✅ Complete | 0 | Bank/sequence DMA loading |
| [`audio_playback.c`](src/audio/audio_playback.c) | ✅ Complete | 0 | Note allocation, playback |
| [`audio_seqplayer.c`](src/audio/audio_seqplayer.c) | ✅ Complete | 0 | M64 sequence script interpreter |
| [`audio_synthesis.c`](src/audio/audio_synthesis.c) | ✅ Complete | 0 | RSP audio command generation |
| [`audio_thread.c`](src/audio/audio_thread.c) | ✅ Complete | 0 | Audio thread management |

**Remaining stubs in [`audio_general.c`](src/audio/audio_general.c):**

| Function | Line | Notes |
|----------|------|-------|
| [`func_800C010C()`](src/audio/audio_general.c:408) | 408 | "chonker" — large function |
| [`func_800C11CC()`](src/audio/audio_general.c:417) | 417 | decomp.me: `cOjnQ` |
| [`func_800C1BD8()`](src/audio/audio_general.c:668) | 668 | decomp.me: `io8AQ` |
| [`func_800C37F4()`](src/audio/audio_general.c:1429) | 1429 | "chonker" — large function |
| [`func_800C3EF4()`](src/audio/audio_general.c:1466) | 1466 | "problematic switch" statement |

### 4.2 System Layer

**Estimated completion: ~92%** — 6 remaining `GLOBAL_ASM` stubs

| File | Status | ASM Stubs | Notes |
|------|--------|-----------|-------|
| [`main.c`](src/game/main.c) | ✅ Fully matched | 0 | Entry point, main loop setup |
| [`sys_audio.c`](src/sys/sys_audio.c) | ✅ Fully matched | 0 | Audio system interface |
| [`sys_main.c`](src/sys/sys_main.c) | ⚠️ 1 remaining | 1 | [`SysMain_Thread()`](src/sys/sys_main.c:106) |
| [`sys_utils.c`](src/sys/sys_utils.c) | ⚠️ 5 remaining | 5 | Controller handling, math utils |

**Remaining stubs in [`sys_utils.c`](src/sys/sys_utils.c):**

| Function | Line |
|----------|------|
| [`func_800498A4()`](src/sys/sys_utils.c:655) | 655 |
| [`func_80049A94()`](src/sys/sys_utils.c:657) | 657 |
| [`func_80049C9C()`](src/sys/sys_utils.c:659) | 659 |
| [`func_8004A3C0()`](src/sys/sys_utils.c:766) | 766 |
| [`func_8004A8B0()`](src/sys/sys_utils.c:768) | 768 |

### 4.3 Game Logic

**Estimated completion: ~50–60%** — ~148 remaining `GLOBAL_ASM` stubs

This is the largest subsystem, covering jet-ski physics, race logic, AI, water simulation, HUD rendering, save/load, and general game state management. Source files reside under [`src/game/`](src/game).

| File | ASM Stubs | Description |
|------|-----------|-------------|
| [`code_52CD0.c`](src/game/code_52CD0.c) | **34** | Race logic, AI, course management |
| [`code_C6C0.c`](src/game/code_C6C0.c) | **30** | Core gameplay, jet-ski physics |
| [`water_69D0.c`](src/game/water_69D0.c) | **19** | Water simulation — nearly untouched |
| [`wr64_save.c`](src/game/core/wr64_save.c) | **14** | Save/load system (~70% done) |
| [`code_43DA0.c`](src/game/code_43DA0.c) | **11** | Gameplay subsystem |
| [`code_24B00.c`](src/game/code_24B00.c) | **7** | HUD / display lists |
| [`code_4F850.c`](src/game/code_4F850.c) | **6** | Game loading, course setup |
| [`code_2FB10.c`](src/game/code_2FB10.c) | **6** | Physics helpers |
| [`code_68A10.c`](src/game/code_68A10.c) | **5** | 3D object management |
| [`code_2C670.c`](src/game/code_2C670.c) | **5** | Collision / geometry |
| [`code_5480.c`](src/game/code_5480.c) | **3** | Low-level game utilities |
| [`code_383C0.c`](src/game/code_383C0.c) | **2** | Game subsystem |
| [`code_3A580.c`](src/game/code_3A580.c) | **2** | Game subsystem |
| [`code_43A60.c`](src/game/code_43A60.c) | **2** | NON_MATCHING: [`func_80088EA8()`](src/game/code_43A60.c:44) |
| [`code_3AC00.c`](src/game/code_3AC00.c) | **1** | NON_MATCHING: [`func_80085510()`](src/game/code_3AC00.c:1291) |
| [`wr64_game_load.c`](src/game/wr64_game_load.c) | **0** | ✅ Fully matched |
| [`wr64_unused_print.c`](src/game/wr64_unused_print.c) | **0** | ✅ Fully matched (debug prints) |
| [`code_4C750.c`](src/game/code_4C750.c) | **0** | ✅ Fully matched |
| [`code_6EF50.c`](src/game/code_6EF50.c) | **0** | ✅ Fully matched |
| [`code_39F00.c`](src/game/code_39F00.c) | **0** | ✅ Fully matched |
| [`code_24270.c`](src/game/code_24270.c) | **0** | ✅ Fully matched |
| [`code_42670.c`](src/game/code_42670.c) | **0** | ✅ Fully matched |
| [`code_BD30.c`](src/game/code_BD30.c) | **0** | ✅ Fully matched |
| [`code_BF40.c`](src/game/code_BF40.c) | **0** | ✅ Fully matched |

### 4.4 Codeseg

**Estimated completion: ~15%** — ~176 remaining `GLOBAL_ASM` stubs

The codeseg contains rendering, camera, 3D math, and display list generation code. These are the hardest files to decompile due to heavy floating-point math and complex data dependencies. Source files in [`src/codeseg/`](src/codeseg).

| File | ASM Stubs | Notes |
|------|-----------|-------|
| [`A95D0.c`](src/codeseg/A95D0.c) | **70** | Camera/3D math — worst file in project |
| [`AF8C0.c`](src/codeseg/AF8C0.c) | **37** | Includes ~32 data symbol stubs (`D_8022XXXX`) |
| [`B97B0.c`](src/codeseg/B97B0.c) | **35** | Rendering pipeline |
| [`B66E0.c`](src/codeseg/B66E0.c) | **18** | Display list generation |
| [`B3AA0.c`](src/codeseg/B3AA0.c) | **6** | Geometry processing |
| [`C94E0.c`](src/codeseg/C94E0.c) | **3** | Matrix/transform utilities |
| [`B17D0.c`](src/codeseg/B17D0.c) | **2** | Rendering helpers |
| [`B3290.c`](src/codeseg/B3290.c) | **2** | Geometry helpers |
| [`B2A70.c`](src/codeseg/B2A70.c) | **1** | "Needs more context" |
| [`B2510.c`](src/codeseg/B2510.c) | **1** | Rendering utility |
| [`wr64_fade.c`](src/codeseg/wr64_fade.c) | **1** | 97% done; [`func_801E7908()`](src/codeseg/wr64_fade.c:299) at 86.02% on decomp.me |

### 4.5 Overlays

**Estimated completion: ~25%** — ~131 remaining `GLOBAL_ASM` stubs across 17 source files in 16 overlay directories.

Source files under [`src/overlays/`](src/overlays):

| Overlay Dir | Source File(s) | ASM Stubs | Purpose |
|-------------|----------------|-----------|---------|
| `ovl_i0`  | [`ovl_1B3EC0.c`](src/overlays/ovl_i0/ovl_1B3EC0.c)   | **4** | Boot up screen |
| `ovl_i1`  | [`ovl_1B55A0.c`](src/overlays/ovl_i1/ovl_1B55A0.c)   | **13** | Main menu (includes NON_MATCHING) |
| `ovl_i2`  | [`ovl_1B9440.c`](src/overlays/ovl_i2/ovl_1B9440.c)   | **13** | Rider select (includes data stub `D_i2_802C8BE0`) |
| `ovl_i3`  | [`ovl_1BC890.c`](src/overlays/ovl_i3/ovl_1BC890.c)    | **4** | Course overview |
| `ovl_i4`  | [`ovl_1BE0B0.c`](src/overlays/ovl_i4/ovl_1BE0B0.c)    | **7** | Course select |
| `ovl_i5`  | [`ovl_1BFF50.c`](src/overlays/ovl_i5/ovl_1BFF50.c), [`ovl_1C17E0.c`](src/overlays/ovl_i5/ovl_1C17E0.c) | **6 + 10** | Race results (2 files; `1C17E0` has 5 data stubs) |
| `ovl_i6`  | [`ovl_1C2250.c`](src/overlays/ovl_i6/ovl_1C2250.c)    | **4** | Unknown purpose |
| `ovl_i7`  | [`ovl_1C43F0.c`](src/overlays/ovl_i7/ovl_1C43F0.c)    | **1** | Time trials results — **best overlay** |
| `ovl_i8`  | [`ovl_1C49A0.c`](src/overlays/ovl_i8/ovl_1C49A0.c)    | **8** | Stunt mode results |
| `ovl_i9`  | [`ovl_1C66D0.c`](src/overlays/ovl_i9/ovl_1C66D0.c)    | **7** | Options menu |
| `ovl_i10` | [`ovl_1C9150.c`](src/overlays/ovl_i10/ovl_1C9150.c)   | **4** | Options: change names |
| `ovl_i11` | [`ovl_1CA480.c`](src/overlays/ovl_i11/ovl_1CA480.c)   | **2** | Options: view records |
| `ovl_i12` | [`ovl_1CAE40.c`](src/overlays/ovl_i12/ovl_1CAE40.c)   | **2** | Options: change conditions |
| `ovl_i13` | [`ovl_1CBAF0.c`](src/overlays/ovl_i13/ovl_1CBAF0.c)   | **20** | Options: audio — **worst overlay, entirely ASM** |
| `ovl_i14` | [`ovl_1CF180.c`](src/overlays/ovl_i14/ovl_1CF180.c)   | **2** | Options: erase course records |
| `ovl_i15` | [`ovl_1CFB60.c`](src/overlays/ovl_i15/ovl_1CFB60.c)   | **4** | Options: save & load |

Additional non-overlay segments with ASM stubs:

| File | ASM Stubs | Notes |
|------|-----------|-------|
| [`unused_code_1B1FB0.c`](src/unused_code_1B1FB0.c) | **10** | Unused/debug code segment |
| [`seg_1C3D00.c`](src/seg_1C3D00.c) | **1** | Miscellaneous segment |

### 4.6 libultra

**Status: Linked from pre-compiled ASM — not decompiled**

The project includes partial libultra C source under [`src/libultra/`](src/libultra) for compilation, but the bulk of libultra functions are linked from pre-assembled object files extracted by splat. These are matched against the original ROM at the binary level.

Key libultra source areas included:

| Subdirectory | Files | Notes |
|-------------|-------|-------|
| `src/libultra/io/` | 25+ files | PI, SI, AI, VI, controller, PFS |
| `src/libultra/libc/` | 5 files | `sprintf`, `xprintf`, `ll.c` |
| `src/libultra/audio/` | 1 file | `bnkf.c` (bank file handling) |
| `src/libultra/` | 4 files | `ldiv.c`, `strchr.c`, `vimodes.c`, `libultra_bss.c` |

Notable special handling in [`Makefile`](Makefile:389):
- Several libultra files bypass `asm-processor` and compile directly with IDO (e.g., `lookatref.o`, `ortho.o`, `cosf.o`, `ll.o`).
- [`ll.c`](src/libultra/libc/ll.c) requires post-compilation patching via [`set_o32abi_bit.py`](Makefile:530).

### 4.7 Miscellaneous Segments

| File | ASM Stubs | Description |
|------|-----------|-------------|
| [`ovl_table.c`](src/ovl_table.c) | 0 | Overlay table (hardcoded addresses, `#if 0` symbolic version) |
| [`seg_1C3780.c`](src/seg_1C3780.c) | 0 | Small miscellaneous segment |
| [`seg_1C3D00.c`](src/seg_1C3D00.c) | 1 | Small miscellaneous segment |
| [`unused_code_1B1FB0.c`](src/unused_code_1B1FB0.c) | 10 | Unused/debug code |

---

## 5. Data Structures

### 5.1 Core Game Structs

Defined in [`include/structs.h`](include/structs.h) — **815 lines, 40+ struct definitions**. The majority remain unnamed with auto-generated names of the form `UnkStruct_XXXXXXXX` (address-derived), reflecting the early stage of reverse-engineering.

**Key identified structs:**

| Struct | Size | File:Line | Description |
|--------|------|-----------|-------------|
| [`Vec3f`](include/structs.h:113) | `0x0C` | structs.h:113 | 3D float vector (x, y, z) |
| [`Vec3s`](include/structs.h:119) | `0x08` | structs.h:119 | 3D short vector (x, y, z) |
| [`GfxPool`](include/structs.h:356) | `0x18FE8` | structs.h:356 | Display list double-buffer pool |
| [`Controller`](include/structs.h:316) | `0x1C` | structs.h:316 | Controller input state |
| [`ControllerBase`](include/structs.h:537) | `0x0A` | structs.h:537 | Raw controller buttons/stick |
| [`Game_801CE608`](include/structs.h:787) | `0x18` | structs.h:787 | Game session state (mode, player, rider, wave level) |
| [`RGB`](include/structs.h:373) | `0x06` | structs.h:373 | 16-bit RGB color |
| [`MF`](include/structs.h:11) | `0x40` | structs.h:11 | Matrix union (integer + component access) |
| [`f_struct`](include/structs.h:18) | `0x18` | structs.h:18 | Generic 6-float struct |
| [`chr_struct`](include/structs.h:33) | `0x14` | structs.h:33 | Character/rider attributes |
| [`UnkStruct_801C3C50`](include/structs.h:242) | `0x16F8` | structs.h:242 | Large player/rider state (speed multiplier at +0x1578) |
| [`UnkStruct_80192690`](include/structs.h:672) | `0x1718` | structs.h:672 | Large game/course state struct |
| [`UnkStruct_801AE948`](include/structs.h:773) | large | structs.h:773 | Rendering context (lights, matrices, hilite) |
| [`UnkStruct_801C2C24`](include/structs.h:295) | `0x378` | structs.h:295 | Race state (lap count, position) |
| [`StructVarS0`](include/structs.h:413) | `0xBC` | structs.h:413 | Physics state struct |
| [`UnkStruct_801CF060`](include/structs.h:462) | `0xBC` | structs.h:462 | Player movement/position state |
| [`UnkStruct_801BC940`](include/structs.h:593) | `0xC4` | structs.h:593 | Rendering state (includes MtxF, Gfx pointers) |
| [`UnkStruct_8004B0F8`](include/structs.h:566) | `0x6C` | structs.h:566 | Spline/path interpolation data |

### 5.2 Audio Structs

Defined in [`include/wr64audio.h`](include/wr64audio.h) — **964 lines, 50+ type definitions**. Directly derived from SM64's audio engine with EU-variant extensions.

**Core audio types:**

| Type | Size | Description |
|------|------|-------------|
| [`SequencePlayer`](include/wr64audio.h:392) | large | M64 sequence player (channels, DMA, fade) |
| [`SequenceChannel`](include/wr64audio.h:481) | large | 16 channels per player, script state |
| [`SequenceChannelLayer`](include/wr64audio.h:533) | large | Up to 4 layers per channel |
| [`Note`](include/wr64audio.h:619) | large | Active note with synthesis + playback state |
| [`NoteSubEu`](include/wr64audio.h:594) | ~0x14 | EU-style note sub-struct (bitfields) |
| [`SynthesisReverb`](include/wr64audio.h:170) | `0xCC`–`0x100` | Reverb ring buffer configuration |
| [`Instrument`](include/wr64audio.h:362) | `0x20` | Bank instrument (3 key ranges) |
| [`Drum`](include/wr64audio.h:373) | variable | Drum (pan, sound, envelope) |
| [`AudioBankSample`](include/wr64audio.h:348) | variable | ADPCM sample descriptor |
| [`AdsrState`](include/wr64audio.h:439) | `0x24` | ADSR envelope state machine |
| [`M64ScriptState`](include/wr64audio.h:386) | variable | Script PC, stack, loop counters |
| [`SoundAllocPool`](include/wr64audio.h:252) | `0x10` | Heap allocator for audio memory |
| [`SoundMultiPool`](include/wr64audio.h:288) | `0x1D0` | Persistent + temporary pool pair |
| [`SharedDma`](include/wr64audio.h:226) | variable | Sample DMA cache entry (TTL-based) |
| [`AudioSessionSettingsEU`](include/wr64audio.h:653) | variable | Session preset (frequency, reverb, memory) |
| [`SPTask`](include/wr64audio.h:141) | `0x4C` | RSP task wrapper with message queue |

**Audio constants:**

```c
#define MAX_CHANNELS           16
#define MAX_SEQ_PLAYERS         4
#define SEQUENCE_PLAYERS        4
#define SEQUENCE_CHANNELS      48
#define SEQUENCE_LAYERS        64
#define NUMAIBUFFERS            3
#define AIBUFFER_LEN         0xA00   // (0xa0 * 16)
#define TATUMS_PER_BEAT        48
```

### 5.3 Game Enums

| Enum | File | Values | Description |
|------|------|--------|-------------|
| [`Course`](include/course.h:4) | course.h | 10 entries | `DOLPHIN_PARK` through `RIDER_SELECTION` |
| [`Difficulty`](include/game.h:6) | game.h | 3 entries | `NORMAL`, `HARD`, `EXPERT` |
| [`GameMode`](include/game.h:19) | game.h | 4 entries | `TIME_TRIALS`, `2P_VS`, `CHAMPIONSHIP`, `STUNT` |
| [`GameModeState`](include/game.h:12) | game.h | 4 entries | `STATE_0` through `STATE_STARTED` |
| [`GameState`](include/game.h:28) | game.h | 35+ entries | Full game state machine (menus, races, results, options) |
| [`Players`](include/player.h:4) | player.h | 3 entries | `NO_PLAYERS`, `ONE_PLAYER`, `TWO_PLAYERS` |
| [`OverlayId`](include/wr64ovl.h:4) | wr64ovl.h | 19 entries | Overlay identifiers |
| [`RiderId`](include/rider.h:4) | rider.h | 4 entries | `RHAYAMI`, `DMARINER`, `ASTEWART`, `MJETER` |
| [`RIDER_GAME_MODES`](include/rider.h:11) | rider.h | 4 entries | `PAUSE_MODE`, `UNACTIVE_MODE`, `AI_MODE`, `RACE_MODE` |

### 5.4 Key Type Definitions

From [`include/macros.h`](include/macros.h):

```c
#define ARRAY_COUNT(arr)  (s32)(sizeof(arr) / sizeof(arr[0]))
#define SCREEN_WIDTH      320
#define SCREEN_HEIGHT     240
#define SQ(x)             (x * x)
#define FABS(x)           ((x) >= 0.0f ? (x) : -(x))
#define ABS(x)            ((x) >= 0 ? (x) : -(x))
#define SIGNUM(x)         ((x) == 0 ? 0 : ((x) > 0 ? 1 : -1))
#define VIRTUAL_TO_PHYSICAL(addr)   ((uintptr_t)(addr) & 0x1FFFFFFF)
#define PHYSICAL_TO_VIRTUAL(addr)   ((uintptr_t)(addr) | 0x80000000)
```

---

## 6. Symbol Coverage

### 6.1 Named vs Unnamed Functions

The project uses a mix of recovered names and auto-generated `func_XXXXXXXX` placeholders:

| Category | Pattern | Examples |
|----------|---------|----------|
| **Named (recovered)** | Descriptive names | `SysMain_Thread()`, `AudioThread_Init()`, `AudioSeq_ProcessSequences()`, `SysUtils_UpdateControllers()`, `SysUtils_Round()`, `FadeTransition_SetProps()`, `Audio_AdsrUpdate()`, `Save_CodeToChar()`, `Mio0_Decompress()`, `Math_TwoLineSegmentsIntersect()`, `Draw_WaterEffects()` |
| **Partially named** | Prefix + generic | `AudioHeap_AllocCached()`, `AudioLoad_InitSampleDmaBuffers()`, `AudioSeq_ScriptReadU8()` |
| **Auto-generated** | `func_XXXXXXXX` | `func_800C010C()`, `func_801DB8F0()`, `func_8009A1CC()` |
| **Overlay-prefixed** | `func_iN_XXXXXXXX` | `func_i1_802C59E8()`, `func_i8_802C6E00()`, `func_i13_802C5800()` |
| **Data symbols** | `D_XXXXXXXX` | `D_801CE608`, `D_80192690`, `D_800DAB2C` |

### 6.2 Symbol Files

Located in [`linker_scripts/us/rev1/`](linker_scripts/us/rev1/):

| File | Purpose |
|------|---------|
| [`symbol_addrs.txt`](linker_scripts/us/rev1/symbol_addrs.txt) | Master symbol address definitions |
| [`audio_symbols.txt`](linker_scripts/us/rev1/audio_symbols.txt) | Audio subsystem symbols |
| [`libultra_symbols.txt`](linker_scripts/us/rev1/libultra_symbols.txt) | libultra function/data symbols |
| [`libultra_undefined_syms.txt`](linker_scripts/us/rev1/libultra_undefined_syms.txt) | Unresolved libultra references |
| [`ovl_symbols.txt`](linker_scripts/us/rev1/ovl_symbols.txt) | Overlay-specific symbols |
| [`resolve.txt`](linker_scripts/us/rev1/resolve.txt) | Cross-segment symbol resolution |

Additional symbol source: [`files/symbol_addrs.txt`](files/symbol_addrs.txt)

### 6.3 Best-Covered Areas

1. **Audio** — ~97% decompiled, SM64-derived naming covers virtually all functions. Only 5 stubs remain.
2. **System** — ~92% decompiled. Core boot and audio interface fully matched. `main.c` and `sys_audio.c` are 100%.
3. **Game utilities** — Several small game files fully matched: `wr64_game_load.c`, `wr64_unused_print.c`, `code_4C750.c`, `code_6EF50.c`, `code_39F00.c`, `code_24270.c`, `code_42670.c`, `code_BD30.c`, `code_BF40.c`.

### 6.4 Worst-Covered Areas

1. **Codeseg [`A95D0.c`](src/codeseg/A95D0.c)** — 70 ASM stubs, camera/3D math, the single worst file in the entire project.
2. **Codeseg [`AF8C0.c`](src/codeseg/AF8C0.c)** — 37 stubs (includes ~32 data symbol stubs `D_8022XXXX`).
3. **Codeseg [`B97B0.c`](src/codeseg/B97B0.c)** — 35 ASM stubs, rendering pipeline.
4. **Game [`code_52CD0.c`](src/game/code_52CD0.c)** — 34 ASM stubs, race logic and AI.
5. **Game [`code_C6C0.c`](src/game/code_C6C0.c)** — 30 ASM stubs, core jet-ski physics.
6. **Overlay [`ovl_i13`](src/overlays/ovl_i13/ovl_1CBAF0.c)** — 20 ASM stubs, entirely undecompiled (audio options overlay).
7. **Game [`water_69D0.c`](src/game/water_69D0.c)** — 19 ASM stubs, water simulation nearly untouched.
8. **Codeseg [`B66E0.c`](src/codeseg/B66E0.c)** — 18 ASM stubs, display list generation.

---

## 7. Asset Pipeline

### 7.1 MIO0 Compression

Wave Race 64 uses **MIO0 compression** for asset data within the ROM. The decompression algorithm is implemented in the project as [`Mio0_Decompress()`](include/functions.h:139).

**MIO0 tools in the project:**

| Tool | Path | Purpose |
|------|------|---------|
| `libmio0.c` / `libmio0.h` | [`tools/libmio0.c`](tools/libmio0.c) | C library for MIO0 compress/decompress |
| `mio0_extract.py` | [`tools/mio0_extract.py`](tools/mio0_extract.py) | Extract MIO0-compressed blocks from ROM |
| `mio0_decompress.py` | [`tools/mio0_decompress.py`](tools/mio0_decompress.py) | Standalone decompression script |
| `mio0_check.py` | [`tools/mio0_check.py`](tools/mio0_check.py) | Verify MIO0 block integrity |

The extraction step (`make extract`) calls:
```bash
python3 tools/mio0_extract.py baserom.us.rev1.z64
```

### 7.2 Torch/F3D Integration

**Torch** is a custom asset extraction and import tool built for this project, located at [`tools/Torch/`](tools/Torch). It handles:

- **Code extraction**: `torch code <baserom>` — Extracts code-related assets
- **Header extraction**: `torch header <baserom>` — Extracts header data
- **Modding export**: `torch modding export <baserom>` — Exports moddable assets
- **Modding import**: `torch modding import code <baserom>` — Imports modified assets

Configuration is defined in [`config.yml`](config.yml):

```yaml
config:
  gbi: F3D          # Fast3D microcode (original N64 F3D, not F3DEX)
  sort: OFFSET       # Sort assets by ROM offset
  logging: ERROR     # Logging level
  output:
    binary: wr64.otr
    code: src/assets
    headers: include/assets
    modding: src/assets
```

Currently at **stub level** — the asset extraction pipeline exists but comprehensive asset definitions are not yet created.

### 7.3 Current Asset Definitions

Only **1 GFX asset YAML** currently exists:

| File | Path |
|------|------|
| [`ast_1E5860.yaml`](assets/yaml/rev1/ast_1E5860.yaml) | `assets/yaml/rev1/` |

This indicates the asset extraction pipeline is in its very early stages. Most assets are still handled as raw binary blobs extracted by splat.

### 7.4 Asset Tools

| Tool | Purpose |
|------|---------|
| [`tools/comptool.py`](Makefile:228) | Compression tool (threads: `--threads N`) |
| [`tools/decompress_files.c`](tools/decompress_files.c) | Decompress binary files |
| [`tools/align_files.c`](tools/align_files.c) | Align binary files to boundaries |
| [`tools/n64cksum.c`](tools/n64cksum.c) | N64 checksum calculator |
| [`tools/n64crc.c`](tools/n64crc.c) | N64 CRC fixer for ROM headers |

---

## 8. Known Issues & Technical Debt

### 8.1 NON_MATCHING Guards

Three functions use `#ifdef NON_MATCHING` / `#ifndef NON_MATCHING` guards, meaning they have C implementations that are functionally correct but do not produce byte-identical assembly:

| Function | File | Line | Issue |
|----------|------|------|-------|
| [`func_80085510()`](src/game/code_3AC00.c:1291) | `code_3AC00.c` | 1291 | Register allocation mismatch ("Regalloc :(") |
| [`func_80088EA8()`](src/game/code_43A60.c:44) | `code_43A60.c` | 44 | Non-matching C implementation |
| [`func_i1_802C8E70()`](src/overlays/ovl_i1/ovl_1B55A0.c:164) | `ovl_1B55A0.c` | 164 | decomp.me: `7qW05` |

When building with `NON_MATCHING=1` (or `COMPILER=gcc`), the `AVOID_UB` flag is also set, which activates alternative code paths that avoid undefined behavior present in the original compiled binary.

Additionally, 4 functions in [`code_52CD0.c`](src/game/code_52CD0.c) use `#if 1` / `#else` ASM fallback patterns (lines 1317, 2053, 2195), suggesting near-matches that still fall back to ASM stubs.

### 8.2 decomp.yaml Vestigial Name

The [`decomp.yaml`](decomp.yaml) file contains a vestigial project name from an earlier codebase:

```yaml
name: Wonder Project J2
repo: https://github.com/LLONSIT-glitch/wonder
```

This is a leftover from when the project scaffold was forked from the "Wonder Project J2" decompilation. The actual ROM paths and build configuration correctly reference Wave Race 64:

```yaml
map: "build/waverace64.us.rev1.map"
compiled_target: "build/waverace64.us.rev1.z64"
```

**Impact:** The `decomp.yaml` is consumed by external tools (`decomp.me`, `mapfile_parser`, `permuter`). The incorrect name may cause confusion in progress reporting dashboards but does not affect the build.

### 8.3 Dual Overlay Table

[`src/ovl_table.c`](src/ovl_table.c) contains **two versions** of `gOverlayTable[]`:

1. **Symbolic version** (lines 6–26) — Uses `OVERLAY_ENTRY()` macros referencing linker symbols. Currently disabled via `#if 0` because it depends on BSS sizes that require all overlays to be correctly disassembled first.

2. **Hardcoded version** (lines 28–238) — Contains manually-entered ROM offsets and VRAM addresses. This is the active version.

```c
// This won't work until we disassemble all overlays correctly
// as the linker is unable to know the size of BSS.
#if 0
Overlay gOverlayTable[] = {
    OVERLAY_ENTRY(segment_1B1FB0),
    OVERLAY_ENTRY(ovl_i0),
    ...
};
#else
Overlay gOverlayTable[] = {
    { 0x001B1FB0, 0x001B3EC0, 0x802C5800, ... },
    ...
};
#endif
```

**Resolution needed:** Once all overlay BSS sections are correctly defined in the linker scripts, the symbolic version should replace the hardcoded one.

### 8.4 Data Migration Issues

Several overlay functions note "Matched but needs data migration" — meaning the function's C code compiles correctly, but associated `.rodata` or `.data` sections haven't been moved from assembly into C source files yet.

Example from [`ovl_1B3EC0.c`](src/overlays/ovl_i0/ovl_1B3EC0.c:143):
```c
// Matched but needs data migration
#pragma GLOBAL_ASM("asm/us/rev1/nonmatchings/overlays/ovl_i0/ovl_1B3EC0/func_i0_802C6AE4.s")
```

Additionally, [`AF8C0.c`](src/codeseg/AF8C0.c) contains **~32 data symbol stubs** (`D_80226210` through `D_80226280`) that are still referenced as assembly includes rather than C data definitions.

### 8.5 decomp.me Scratch Links

Several functions include `decomp.me` scratch URLs as comments, providing shareable decompilation progress:

| Function | File | URL | Match % |
|----------|------|-----|---------|
| [`func_800C11CC()`](src/audio/audio_general.c:416) | audio_general.c | `decomp.me/scratch/cOjnQ` | Unknown |
| [`func_800C1BD8()`](src/audio/audio_general.c:667) | audio_general.c | `decomp.me/scratch/io8AQ` | Unknown |
| [`func_801E7908()`](src/codeseg/wr64_fade.c:298) | wr64_fade.c | `decomp.me/scratch/udOjy` | **86.02%** |
| [`func_i1_802C8E70()`](src/overlays/ovl_i1/ovl_1B55A0.c:163) | ovl_1B55A0.c | `decomp.me/scratch/7qW05` | Unknown |

The decomp.me integration is configured in [`decomp.yaml`](decomp.yaml:19) with preset ID `51` and supports IDO 5.3 / IDO 7.1 compilers via the permuter tool.

---

## 9. Tools & Utilities

### 9.1 asm-differ

**Path:** [`tools/asm-differ/`](tools/asm-differ)

A diff tool for comparing assembly output between the target ROM and the current build. Used during decompilation to verify function matching.

**Usage workflow:**
1. Build the project: `make`
2. Create expected baseline: `make expected`
3. Run diff on a specific function: `./diff <function_name>`

Configuration via [`diff_settings.py`](diff_settings.py):
```python
config['baseimg'] = 'baserom.us.rev1.z64'
config['myimg']   = 'build/waverace64.us.rev1.z64'
config['mapfile']  = 'build/waverace64.us.rev1.map'
config['source_directories'] = ['src', 'include']
```

### 9.2 asm-processor

**Path:** [`tools/asm-processor/`](tools/asm-processor)

A Python-based wrapper around the IDO compiler that enables `GLOBAL_ASM()` pragmas to work. It extracts inline assembly blocks, compiles the C code, and then patches the assembly back into the object file.

**Invocation** (from [`Makefile:210`](Makefile:210)):
```bash
python3 tools/asm-processor/build.py \
  --input-enc=utf-8 \
  --output-enc=euc-jp \
  --convert-statics=global-with-filename \
  tools/ido-static-recomp/build/5.3/out/cc -- \
  mips-linux-gnu-as -march=vr4300 -32 -G0 --
```

Key flags:
- `--input-enc=utf-8` / `--output-enc=euc-jp`: Handle Japanese text encoding in the original binary
- `--convert-statics=global-with-filename`: Convert static variables to globals (scoped by filename) to match IDO's code generation

### 9.3 splat

**Path:** [`tools/splat/`](tools/splat)

A ROM splitting tool (N64 decomp community standard) that:
- Splits the ROM into individual segments based on a YAML configuration
- Generates assembly files for undecompiled code
- Generates linker scripts matching the original ROM layout
- Extracts binary data blobs

**Invocation:**
```bash
python3 tools/splat/split.py waverace64.us.rev1.yaml
```

The YAML config file (`waverace64.us.rev1.yaml`) defines segment boundaries, types, and names for the entire ROM.

### 9.4 MIO0 Tools

| Tool | Path | Language | Purpose |
|------|------|----------|---------|
| `libmio0.c` | [`tools/libmio0.c`](tools/libmio0.c) | C | MIO0 compress/decompress library |
| `libmio0.h` | [`tools/libmio0.h`](tools/libmio0.h) | C | MIO0 library header |
| `mio0_extract.py` | [`tools/mio0_extract.py`](tools/mio0_extract.py) | Python | Extract all MIO0 blocks from ROM |
| `mio0_decompress.py` | [`tools/mio0_decompress.py`](tools/mio0_decompress.py) | Python | Standalone block decompression |
| `mio0_check.py` | [`tools/mio0_check.py`](tools/mio0_check.py) | Python | Validate MIO0 block integrity |
| `comptool.py` | [`tools/comptool.py`](tools/comptool.py) | Python | Multi-threaded compression tool |

MIO0 is Nintendo's lightweight LZ-based compression format used on N64 for asset data. During `make extract`, MIO0-compressed blocks are extracted and decompressed into the `bin/` directory.

### 9.5 Code Formatting

**clang-format configuration** ([`.clang-format`](.clang-format)):

```yaml
IndentWidth: 4
Language: Cpp
UseTab: Never
ColumnLimit: 120
PointerAlignment: Left
BreakBeforeBraces: Attach
SpaceAfterCStyleCast: true
IndentCaseLabels: true
SortIncludes: false          # Preserve original include order
AllowShortFunctionsOnASingleLine: false
```

**Formatting tools:**

| Tool | Command | Purpose |
|------|---------|---------|
| [`tools/format.py`](tools/format.py) | `make format` | Auto-format all C source files |
| [`tools/check_format.sh`](tools/check_format.sh) | `make checkformat` | CI-style format verification |

Both support parallel execution via `-j <threads>`.

**Additional code quality:**

| Tool | Path | Purpose |
|------|------|---------|
| [`.clang-tidy`](.clang-tidy) | Root | Static analysis configuration |
| `m2ctx` | [`tools/m2ctx`](tools/m2ctx) | Generate `ctx.c` context files for decomp.me |
| `progress.py` | [`tools/progress.py`](tools/progress.py) | Calculate decompilation progress statistics |
| `upload_progress.py` | [`tools/upload_progress.py`](tools/upload_progress.py) | Upload progress to tracking dashboard |
| `set_o32abi_bit.py` | [`tools/set_o32abi_bit.py`](tools/set_o32abi_bit.py) | Patch O32 ABI flag in `ll.o` |
| `convLibSyms.py` | [`tools/convLibSyms.py`](tools/convLibSyms.py) | Convert library symbol formats |

---

*Document generated for the Wave Race 64 decompilation project. All file paths, function names, and statistics are derived from source analysis of the repository.*