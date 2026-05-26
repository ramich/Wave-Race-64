# Wave Race 64 — PC Recompilation Plan

> **Project Goal:** Create a native PC port of Wave Race 64 (USA Rev 1) using N64Recomp static recompilation, enabling the game to run natively on Windows, Linux, and macOS with modern enhancements.

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Technology Stack](#2-technology-stack)
3. [Prerequisites & Dependencies](#3-prerequisites--dependencies)
4. [Phase 1: Environment Setup & Toolchain (Weeks 1–2)](#4-phase-1-environment-setup--toolchain)
5. [Phase 2: Static Recompilation (Weeks 2–4)](#5-phase-2-static-recompilation)
6. [Phase 3: Runtime Integration (Weeks 4–8)](#6-phase-3-runtime-integration)
7. [Phase 4: Graphics & Rendering (Weeks 8–16)](#7-phase-4-graphics--rendering)
8. [Phase 5: Audio & Input (Weeks 8–12)](#8-phase-5-audio--input)
9. [Phase 6: Game-Specific Fixes (Weeks 12–20)](#9-phase-6-game-specific-fixes)
10. [Phase 7: Enhancements (Weeks 20–32)](#10-phase-7-enhancements)
11. [Phase 8: Release Preparation (Weeks 32–36)](#11-phase-8-release-preparation)
12. [Risk Assessment](#12-risk-assessment)
13. [Repository Structure](#13-repository-structure)
14. [Reference Projects](#14-reference-projects)

---

## 1. Executive Summary

Wave Race 64 is a 1996 Nintendo 64 jet-ski racing game known for its groundbreaking water physics and wave simulation. This plan outlines the creation of a native PC port using **N64Recomp** — a static recompilation tool that translates N64 MIPS binary code directly into compilable C code.

### Why N64Recomp Instead of Completing the Decomp?

| Factor | Complete Decomp First | N64Recomp (This Plan) |
|--------|----------------------|----------------------|
| Time to playable PC build | 35–67 weeks (decomp) + 16–24 weeks (port) = **51–91 weeks** | **8–16 weeks** |
| Water physics code | Must decompile 19 ASM functions (hardest in project) | Recompiles from binary automatically |
| Camera/3D math | 70+ functions, 8–16 weeks to decomp | Recompiles automatically |
| Risk of introducing bugs | High (manual C rewrite) | Low (mechanical translation) |
| ROM required | No (produces standalone source) | Yes (user must provide their own ROM) |
| Mod potential | Full source access | Function-level hooking + patching |

### Current Decomp Assets That Benefit Recomp

The existing 65.71% decompilation (889/1,365 functions) provides:
- ✅ **Complete function boundary map** — All 1,365 functions identified
- ✅ **Symbol names** for ~889 functions — enables debugging, hooking, modding
- ✅ **19 overlay definitions** — fully mapped with ROM/VRAM addresses
- ✅ **Struct definitions** — Controller, Player, Audio types documented
- ✅ **Audio engine** — 97% decompiled, SM64-derived (known-good with N64Recomp)
- ✅ **libultra function identification** — all OS calls mapped
- ✅ **Memory map** — complete segment layout from linker scripts

### Target Platforms

| Platform | Graphics API | Status |
|----------|-------------|--------|
| Windows 10/11 | Direct3D 12 | Primary target |
| Linux (desktop) | Vulkan | Secondary target |
| Steam Deck | Vulkan | Secondary target |
| macOS | MoltenVK (Vulkan) | Stretch goal |

---

## 2. Technology Stack

### Core Components

| Component | Project | Repository | Role |
|-----------|---------|------------|------|
| **Static Recompiler** | N64Recomp | https://github.com/N64Recomp/N64Recomp | MIPS → C translation |
| **Rendering Engine** | RT64 | https://github.com/rt64/rt64 | F3D → D3D12/Vulkan GPU rendering |
| **Runtime Library** | N64ModernRuntime | https://github.com/N64Recomp/N64ModernRuntime | OS stubs, memory sim, threading |
| **Audio** | HLE Audio (built into runtime) | (integrated) | SM64-style audio engine emulation |
| **Input** | SDL2 | https://github.com/libsdl-org/SDL | Controller/keyboard input |
| **Windowing** | SDL2 | (same) | Window management |
| **Build System** | CMake | — | Cross-platform builds |

### Decomp Data Sources (from this repository)

| Data | Source File | Purpose in Recomp |
|------|-----------|-------------------|
| Function boundaries | `linker_scripts/us/rev1/symbol_addrs.txt` | Function splitting in recompiler |
| Audio symbols | `linker_scripts/us/rev1/audio_symbols.txt` | Audio function identification |
| libultra symbols | `linker_scripts/us/rev1/libultra_symbols.txt` | OS stub mapping |
| Overlay table | `src/ovl_table.c` | Overlay recompilation config |
| Overlay symbols | `linker_scripts/us/rev1/ovl_symbols.txt` | Overlay function names |
| Struct definitions | `include/structs.h`, `include/wr64audio.h` | Memory layout understanding |
| GBI config | `include/PR/gbi.h` (F3D_OLD defines) | Graphics microcode identification |

---

## 3. Prerequisites & Dependencies

### Required Software

| Tool | Version | Purpose |
|------|---------|---------|
| CMake | ≥ 3.20 | Build system |
| C++ Compiler | MSVC 2022 / GCC 12+ / Clang 15+ | Native compilation |
| Python | ≥ 3.10 | Build scripts, symbol processing |
| Git | ≥ 2.30 | Source control |
| Ninja | ≥ 1.10 | Fast builds (optional) |

### Required User-Provided Assets

| Asset | SHA-1 | Purpose |
|-------|-------|---------|
| `baserom.us.rev1.z64` | `508dfc2d4caa42b6f6de5263d0aed5e44ac7966a` | Source ROM for recompilation |

### Platform-Specific Requirements

**Windows:**
- Windows 10 SDK (for D3D12)
- Visual Studio 2022 with C++ workload

**Linux:**
- Vulkan SDK ≥ 1.3
- SDL2 development libraries
- X11 or Wayland development headers

---

## 4. Phase 1: Environment Setup & Toolchain (Weeks 1–2)

### 4.1 Repository Setup
- [ ] Create `WACOMalt/WaveRace64-Recomp` repository on GitHub
- [ ] Set up project structure (see Section 13)
- [ ] Add N64Recomp, RT64, N64ModernRuntime as git submodules
- [ ] Create CMakeLists.txt for the project
- [ ] Set up CI/CD (GitHub Actions) for Windows and Linux builds

### 4.2 Build N64Recomp Toolchain
- [ ] Clone and build N64Recomp from source
- [ ] Verify the recompiler runs on test ROMs
- [ ] Document build process in project README

### 4.3 Build RT64
- [ ] Clone RT64 (with submodules)
- [ ] Build RT64 for target platform (D3D12 on Windows, Vulkan on Linux)
- [ ] Run RT64 test suite to verify GPU compatibility

### 4.4 Build N64ModernRuntime
- [ ] Clone N64ModernRuntime
- [ ] Build runtime library
- [ ] Understand API for game-specific integration

### 4.5 Symbol Preparation
- [ ] Export all symbols from the decomp project into N64Recomp-compatible format
- [ ] Merge `symbol_addrs.txt`, `audio_symbols.txt`, `libultra_symbols.txt`, `ovl_symbols.txt`
- [ ] Create a master symbol list with addresses and sizes
- [ ] Identify any gaps in function boundary coverage

### Deliverable: Toolchain builds, symbols prepared, project skeleton ready

---

## 5. Phase 2: Static Recompilation (Weeks 2–4)

### 5.1 TOML Configuration
- [ ] Create `waverace64.toml` configuration file for N64Recomp
- [ ] Define ROM entry point (`0x80000400`)
- [ ] Define memory segments from decomp linker scripts
- [ ] Configure all 19 overlays with ROM/VRAM addresses:
  ```
  Overlay mappings from ovl_table.c:
  ovl_i0:  ROM 0x1B3EC0 → VRAM 0x802C5800
  ovl_i1:  ROM 0x1B55A0 → VRAM 0x802C5800
  ovl_i2:  ROM 0x1B9440 → VRAM 0x802C5800
  ovl_i3:  ROM 0x1BC890 → VRAM 0x802C5800
  ovl_i4:  ROM 0x1BE0B0 → VRAM 0x802C5800
  ovl_i5:  ROM 0x1BFF50 → VRAM 0x802C5800
  ovl_i6:  ROM 0x1C2250 → VRAM 0x802C5800
  ovl_i7:  ROM 0x1C43F0 → VRAM 0x802C5800
  ovl_i8:  ROM 0x1C49A0 → VRAM 0x802C5800
  ovl_i9:  ROM 0x1C66D0 → VRAM 0x802C5800
  ovl_i10: ROM 0x1C9150 → VRAM 0x802C5800
  ovl_i11: ROM 0x1CA480 → VRAM 0x802C5800
  ovl_i12: ROM 0x1CAE40 → VRAM 0x802C5800
  ovl_i13: ROM 0x1CBAF0 → VRAM 0x802C5800
  ovl_i14: ROM 0x1CF180 → VRAM 0x802C5800
  ovl_i15: ROM 0x1CFB60 → VRAM 0x802C5800
  ```
- [ ] Define libultra function stubs (OS functions to replace with host implementations)
- [ ] Define ignored/patched functions

### 5.2 Initial Recompilation Run
- [ ] Run N64Recomp on `baserom.us.rev1.z64` with the TOML config
- [ ] Fix any recompiler errors (usually unrecognized instructions or function boundary issues)
- [ ] Verify generated C code compiles with the native compiler
- [ ] Count generated functions vs expected (should be ~1,365)

### 5.3 Overlay Recompilation
- [ ] Verify all 19 overlays are correctly identified and recompiled
- [ ] Test overlay function table generation
- [ ] Verify address space sharing works correctly (all overlays at 0x802C5800)

### 5.4 Function Table Generation
- [ ] Generate the function lookup table for indirect calls (jalr targets)
- [ ] Verify jump table handling for switch statements
- [ ] Test indirect function call resolution

### Deliverable: All game code recompiled to C, compiles to native binary

---

## 6. Phase 3: Runtime Integration (Weeks 4–8)

### 6.1 RDRAM Setup
- [ ] Initialize 4MB RDRAM buffer (WR64 uses 4MB, not expansion pak)
- [ ] Load ROM data into simulated PI (Peripheral Interface)
- [ ] Set up initial memory state (boot vector, etc.)

### 6.2 libultra OS Stubs
- [ ] Implement/connect OS function replacements:
  - `osCreateThread` / `osStartThread` / `osDestroyThread`
  - `osCreateMesgQueue` / `osSendMesg` / `osRecvMesg`
  - `osViSwapBuffer` / `osViSetMode` / `osViSetSpecialFeatures` / `osViBlack`
  - `osAiSetFrequency` / `osAiSetNextBuffer` / `osAiGetLength`
  - `osPiStartDma` (ROM → RDRAM transfers)
  - `osEepromRead` / `osEepromWrite` / `osEepromLongWrite` (→ file save)
  - `osContInit` / `osContStartReadData` / `osContGetReadData`
  - `osPfsInit` / other Controller Pak functions (if needed)
  - `osGetTime` / `osGetCount`
  - `osInvalDCache` / `osInvalICache` (no-ops on PC)
  - `osSpTaskLoad` / `osSpTaskStartGo` / `osSpTaskYield` (→ RT64/audio dispatch)

### 6.3 Thread Simulation
- [ ] Map WR64's thread model to host threading:
  - Idle Thread (OS_PRIORITY_IDLE) → main host thread
  - Main Thread (SysMain_Thread) → game logic thread
  - Audio Thread (AudioThread_Update) → audio processing thread
  - Graphics/render thread → synchronizes with RT64
- [ ] Implement message queue simulation for inter-thread communication
- [ ] Handle VI retrace synchronization (frame pacing)

### 6.4 First Boot Test
- [ ] Attempt to boot the recompiled game
- [ ] Debug crashes — likely candidates:
  - Indirect jump targets not in function table
  - Missing OS function stubs
  - Memory alignment issues
  - Overlay loading failures
- [ ] Target: reach the Nintendo logo / title screen

### Deliverable: Game boots and shows initial graphics

---

## 7. Phase 4: Graphics & Rendering (Weeks 8–16)

### 7.1 F3D_OLD Microcode Support
- [ ] Verify RT64 supports Wave Race 64's F3D_OLD (original Fast3D) microcode
- [ ] Test GBI command differences vs F3DEX2:
  - Vertex buffer: 16 vertices (vs 32 in F3DEX)
  - Different texture rectangle format
  - Missing gSPPerspNormalize
  - Potential WR64-specific GBI extensions (F3D_WR64 flag)
- [ ] Fix any unsupported GBI commands
- [ ] Test with all 9 courses:
  - Sunny Beach, Sunset Bay, Drake Lake
  - Marine Fortress, Port Blue, Twilight City
  - Glacier Coast, Southern Island, Stunt mode courses

### 7.2 Water Rendering Verification
- [ ] Verify dynamic water surface geometry renders correctly
- [ ] Check water transparency and alpha blending
- [ ] Test wave deformation (the game generates vertex positions each frame)
- [ ] Verify water surface normals and lighting
- [ ] Check underwater rendering effects (if applicable)
- [ ] Test water spray/splash particle effects
- [ ] Verify wake trails behind jet-skis

### 7.3 Special Effects
- [ ] Screen transitions (FadeTransition system from wr64_fade.c)
- [ ] Lens flare / sun effects
- [ ] Rain/weather effects
- [ ] Fog rendering
- [ ] Particle systems (water spray, exhaust)

### 7.4 UI/HUD Rendering
- [ ] Verify 2D HUD elements render correctly (speed, position, timer)
- [ ] Menu screens and text rendering
- [ ] Title screen and logos
- [ ] Results screens and leaderboards
- [ ] Character select screen

### 7.5 Split-Screen (2P Mode)
- [ ] Verify viewport scissoring works in RT64 for 2-player mode
- [ ] Test both viewports render independently
- [ ] Verify HUD elements render correctly per viewport

### Deliverable: All courses render correctly, menus work, 2P split-screen works

---

## 8. Phase 5: Audio & Input (Weeks 8–12)

*Can be done in parallel with Phase 4*

### 8.1 Audio Integration
- [ ] Verify SM64-derived HLE audio works with Wave Race 64's audio engine
- [ ] Test music playback (M64 sequences)
- [ ] Test sound effects (ADPCM samples)
- [ ] Verify 4 sequence players work correctly
- [ ] Test spatial audio / stereo panning
- [ ] Check volume levels and mixing (16 channels)
- [ ] Test reverb effects (ring buffer implementation)
- [ ] Verify audio doesn't glitch during gameplay transitions

### 8.2 Input Configuration
- [ ] Map N64 controls to modern gamepad (see mapping table):
  - Analog stick → Left stick (jet-ski steering, full 360°)
  - A → Accelerate, B → Brake
  - Z/L/R → Lean/crouch/sharp turn
  - C-buttons → Right stick (camera)
  - Start → Pause/menu
- [ ] Implement keyboard+mouse fallback mapping
- [ ] Add analog stick dead zone configuration
- [ ] Calibrate analog stick sensitivity for jet-ski steering feel
- [ ] Add Rumble Pak → haptic feedback mapping
- [ ] Test controller hot-plug support
- [ ] Save controller preferences to settings file

### 8.3 Save System
- [ ] Redirect EEPROM read/write to file-based storage
- [ ] Implement save file format (binary EEPROM dump or structured format)
- [ ] Save file location: user's app data directory
- [ ] Test save/load for:
  - Championship progress
  - Time trial records
  - Options/settings
  - Unlockable content

### Deliverable: Full audio, working controls, save/load functional

---

## 9. Phase 6: Game-Specific Fixes (Weeks 12–20)

### 9.1 Gameplay Testing Checklist
- [ ] **Championship Mode** — Complete all cups on Normal/Hard/Expert
- [ ] **Time Trial** — All 9 courses
- [ ] **Stunt Mode** — Points, trick detection, scoring
- [ ] **2P VS Mode** — Split-screen racing
- [ ] **Menu Flow** — Character select, course select, options, records
- [ ] **All 9 Riders** — Ryota, Dave, Miles, Ayumi, etc.
- [ ] **All 9 Courses** — Verify each renders and plays correctly
- [ ] **Difficulty Levels** — Normal, Hard, Expert AI behavior
- [ ] **Unlockables** — Verify unlock conditions work

### 9.2 Known Risk Areas
- [ ] **Water physics frame-rate dependency** — May need delta-time patches if physics are tied to VI retrace
- [ ] **MIO0 decompression** — Verify runtime decompression works correctly for all compressed assets
- [ ] **Overlay transitions** — Test all overlay load/unload sequences:
  - Title → Menu (overlay transition)
  - Menu → Race (overlay transition)
  - Race → Results (overlay transition)
  - Results → Menu (overlay transition)
- [ ] **Timer accuracy** — Race timers must be accurate despite different frame pacing

### 9.3 Crash/Bug Tracking
- [ ] Set up issue tracker for game-specific bugs
- [ ] Create automated test framework for known crash scenarios
- [ ] Test edge cases:
  - Drowning / falling off jet-ski
  - Collision with walls, buoys, obstacles
  - Race finish / tie scenarios
  - Menu back-navigation edge cases

### Deliverable: All game modes playable without crashes, accurate gameplay

---

## 10. Phase 7: Enhancements (Weeks 20–32)

*Optional but highly desirable for a quality release*

### 10.1 Resolution & Display
- [ ] Arbitrary resolution support (720p, 1080p, 1440p, 4K)
- [ ] Configurable display settings menu
- [ ] VSync / frame limiter options
- [ ] Fullscreen / borderless windowed / windowed modes
- [ ] Multi-monitor support

### 10.2 Widescreen Support
- [ ] 16:9 aspect ratio correction (game renders at 4:3 natively)
- [ ] 21:9 ultrawide support
- [ ] Adjust 3D viewport calculations for wider FOV
- [ ] Fix HUD element positioning for non-4:3 ratios
- [ ] Ensure water surface extends to fill wider viewports

### 10.3 Frame Rate Enhancement
- [ ] 60fps support (original is 30fps NTSC / 25fps PAL)
- [ ] Frame interpolation for smooth motion
- [ ] Or: physics unlocking (more complex, better results)
- [ ] Ensure water simulation updates correctly at higher frame rates
- [ ] Frame pacing / frame time management

### 10.4 Enhanced Water Rendering (RT64 Path Tracing)
- [ ] Optional ray-traced water reflections
- [ ] Real-time water refractions
- [ ] Enhanced caustics
- [ ] Subsurface scattering on water surface
- [ ] Enhanced spray/splash particle effects with volumetric rendering

### 10.5 Dual Analog Camera
- [ ] Free camera control via right analog stick
- [ ] Camera smoothing / interpolation
- [ ] Optional auto-camera (original behavior) vs manual camera

### 10.6 HD Texture Support
- [ ] Texture dump utility (export all textures at original resolution)
- [ ] HD texture pack loading from external directory
- [ ] Runtime texture replacement via RT64's texture interception
- [ ] Community texture pack format specification

### 10.7 Modern UI
- [ ] Settings menu (graphics, audio, input, display)
- [ ] Controller rebinding UI
- [ ] In-game overlay (FPS counter, frame time graph)
- [ ] Notification system for settings changes

### 10.8 Mod Support
- [ ] Function hooking framework (hook any named function)
- [ ] Event system for game state changes
- [ ] Lua or native mod API
- [ ] Mod loading from external directory
- [ ] Example mods: custom courses, rider skins, physics tweaks

### Deliverable: Enhanced PC experience with modern features

---

## 11. Phase 8: Release Preparation (Weeks 32–36)

### 11.1 Packaging
- [ ] Release build optimization (LTO, PGO)
- [ ] Windows installer / portable zip
- [ ] Linux AppImage / Flatpak
- [ ] Steam Deck verified compatibility
- [ ] Distribution includes NO copyrighted material (user provides ROM)

### 11.2 Documentation
- [ ] User manual (setup, controls, settings)
- [ ] ROM verification instructions (SHA-1 check)
- [ ] Troubleshooting guide
- [ ] Controller setup guide with diagrams
- [ ] Mod development guide

### 11.3 Legal
- [ ] Ensure no Nintendo copyrighted code or assets are distributed
- [ ] User must provide their own legally-obtained ROM
- [ ] DMCA-safe distribution model (same as Zelda64Recomp)
- [ ] License selection (GPLv3 or similar)

### 11.4 Testing
- [ ] Windows 10/11 test matrix (NVIDIA, AMD, Intel GPUs)
- [ ] Linux test matrix (multiple distros, GPU drivers)
- [ ] Steam Deck testing
- [ ] Performance benchmarks on various hardware
- [ ] Controller compatibility testing (Xbox, PS, Switch Pro, generic)

### Deliverable: Release-ready builds for all target platforms

---

## 12. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **F3D_OLD GBI incompatibility** | Medium | High | Test early in Phase 2; contribute fixes to RT64 upstream |
| **Water rendering artifacts** | Medium | High | Dedicated Phase 4.2; test all courses systematically |
| **Frame-rate dependent physics** | Medium | Medium | Instrument timing code; add delta-time patches if needed |
| **Overlay loading race conditions** | Low | High | Comprehensive overlay transition testing |
| **Indirect jump table issues** | Medium | Medium | Thorough function table coverage from decomp symbols |
| **Audio glitches** | Low | Medium | SM64 audio engine is well-tested in recomp ecosystem |
| **Controller feel (steering)** | Low | Medium | Analog stick calibration and sensitivity tuning |
| **Save compatibility** | Low | Low | EEPROM binary dump is straightforward |
| **macOS support (MoltenVK)** | High | Low | Stretch goal only; focus on Windows/Linux first |

### Risk Scoring Matrix

```
               IMPACT
            Low  Med  High
     High  │ 🟡 │ 🟠 │ 🔴 │  ← macOS
LIKE  Med  │ 🟢 │ 🟡 │ 🟠 │  ← F3D, Water, Frame-rate, Jump tables
      Low  │ 🟢 │ 🟢 │ 🟡 │  ← Audio, Save, Controller, Overlays
```

---

## 13. Repository Structure

```
WaveRace64-Recomp/
├── CMakeLists.txt              # Root build configuration
├── README.md                   # Project overview, setup instructions
├── LICENSE                     # GPLv3
├── rom_info.toml              # ROM SHA-1 verification
├── waverace64.toml            # N64Recomp configuration
│
├── lib/                       # Git submodules
│   ├── N64Recomp/             # Static recompiler
│   ├── rt64/                  # Rendering engine
│   └── N64ModernRuntime/      # Runtime library
│
├── src/                       # Game-specific source
│   ├── main.cpp               # Application entry point
│   ├── config.cpp/h           # Settings management
│   ├── input.cpp/h            # Controller mapping
│   ├── save.cpp/h             # Save file management
│   └── patches/               # Game-specific code patches
│       ├── os_stubs.cpp       # libultra function replacements
│       ├── graphics.cpp       # RT64 integration patches
│       ├── audio.cpp          # Audio system patches
│       └── widescreen.cpp     # Widescreen/resolution patches
│
├── recompiled/                # Generated by N64Recomp (not committed)
│   ├── funcs/                 # Recompiled C functions
│   └── overlays/              # Recompiled overlay functions
│
├── assets/                    # Non-copyrighted game assets
│   ├── icons/                 # Application icons
│   ├── default_config.toml    # Default settings
│   └── controller_maps/       # Default controller mappings
│
├── docs/                      # Documentation
│   ├── setup.md               # Build & setup guide
│   ├── controls.md            # Controller mapping reference
│   ├── modding.md             # Mod development guide
│   └── troubleshooting.md     # Common issues & fixes
│
├── scripts/                   # Build/utility scripts
│   ├── generate_symbols.py    # Extract symbols from decomp project
│   ├── verify_rom.py          # ROM SHA-1 verification
│   └── package.py             # Release packaging
│
└── .github/
    └── workflows/
        ├── build-windows.yml  # Windows CI
        └── build-linux.yml    # Linux CI
```

---

## 14. Reference Projects

| Project | URL | Relevance |
|---------|-----|-----------|
| **Zelda64Recomp** | https://github.com/Zelda64Recomp/Zelda64Recomp | Flagship N64Recomp project; primary reference |
| **N64Recomp** | https://github.com/N64Recomp/N64Recomp | Core recompiler tool |
| **RT64** | https://github.com/rt64/rt64 | Rendering engine |
| **N64ModernRuntime** | https://github.com/N64Recomp/N64ModernRuntime | Runtime library |
| **Wave Race 64 Decomp** | https://github.com/WACOMalt/Wave-Race-64 | Our decomp fork (symbol/struct source) |
| **SM64 Decomp** | https://github.com/n64decomp/sm64 | Audio engine reference (same engine family) |
| **MM Decomp** | https://github.com/zeldaret/mm | Zelda64Recomp's source decomp reference |

---

## Appendix A: Overlay Address Table

Complete overlay mapping from `ovl_table.c` for N64Recomp configuration:

| ID | Name | ROM Start | ROM End | VRAM Start | VRAM End |
|----|------|-----------|---------|------------|----------|
| 0 | ovl_i0 | 0x1B3EC0 | (ovl_i1 start) | 0x802C5800 | (text+data+bss end) |
| 1 | ovl_i1 | 0x1B55A0 | (ovl_i2 start) | 0x802C5800 | ... |
| 2 | ovl_i2 | 0x1B9440 | ... | 0x802C5800 | ... |
| 3 | ovl_i3 | 0x1BC890 | ... | 0x802C5800 | ... |
| 4 | ovl_i4 | 0x1BE0B0 | ... | 0x802C5800 | ... |
| 5 | ovl_i5 | 0x1BFF50 | ... | 0x802C5800 | ... |
| 6 | ovl_i6 | 0x1C2250 | ... | 0x802C5800 | ... |
| 7 | ovl_i7 | 0x1C43F0 | ... | 0x802C5800 | ... |
| 8 | ovl_i8 | 0x1C49A0 | ... | 0x802C5800 | ... |
| 9 | ovl_i9 | 0x1C66D0 | ... | 0x802C5800 | ... |
| 10 | ovl_i10 | 0x1C9150 | ... | 0x802C5800 | ... |
| 11 | ovl_i11 | 0x1CA480 | ... | 0x802C5800 | ... |
| 12 | ovl_i12 | 0x1CAE40 | ... | 0x802C5800 | ... |
| 13 | ovl_i13 | 0x1CBAF0 | ... | 0x802C5800 | ... |
| 14 | ovl_i14 | 0x1CF180 | ... | 0x802C5800 | ... |
| 15 | ovl_i15 | 0x1CFB60 | ... | 0x802C5800 | ... |
| 16 | unused | 0x1B1FB0 | ... | 0x802C5800 | ... |

*Note: Exact ROM end addresses and VRAM end addresses should be extracted from the linker scripts and splat configuration.*

---

## Appendix B: libultra Functions Requiring Stubs

Functions that must be replaced with host-side implementations:

### Thread Management
- `osCreateThread`, `osStartThread`, `osSetThreadPri`, `osGetThreadPri`
- `osYieldThread`, `osDestroyThread`

### Message Queues
- `osCreateMesgQueue`, `osSendMesg`, `osRecvMesg`, `osJamMesg`
- `osSetEventMesg`

### Video Interface
- `osViSetMode`, `osViSwapBuffer`, `osViBlack`
- `osViSetSpecialFeatures`, `osViGetCurrentFramebuffer`

### Audio Interface
- `osAiSetFrequency`, `osAiSetNextBuffer`, `osAiGetLength`, `osAiGetStatus`

### Peripheral Interface (ROM Access)
- `osPiStartDma`, `osPiRawStartDma`

### Serial Interface (Controllers)
- `osContInit`, `osContStartReadData`, `osContGetReadData`

### EEPROM (Save Data)
- `osEepromProbe`, `osEepromRead`, `osEepromWrite`, `osEepromLongWrite`

### Controller Pak (if used)
- `osPfsInit`, `osPfsAllocateFile`, `osPfsReadWriteFile`, etc.

### Cache Management (No-ops on PC)
- `osInvalDCache`, `osInvalICache`, `osWritebackDCache`, `osWritebackDCacheAll`

### Timer
- `osGetTime`, `osGetCount`, `osSetTimer`

### Math
- `guMtxIdent`, `guTranslate`, `guOrtho`, `guLookAt`, `guPerspective`

### Memory
- `bcopy`, `bzero`

---

*Last updated: 2026-05-26*
*Based on Wave Race 64 decomp analysis at 65.71% (889/1,365 functions)*
