# Pokémon Colosseum on the Nintendo 3DS: inventory and roadmap

First written 2026-11-06 against `master` @ `619237924`.

The counts in section 1 come from regex scans of `src/`. They are orders of magnitude, not
exact call graphs. The disc numbers come from read-only FST/FSYS scans of the GC6E01 and
GC6P01 dumps. The 3DS hardware figures in section 2 are general knowledge and were not
measured here.

## 0. Summary

- The code exists. The matching build has 1,574 objects in `configure.py`: game 1,186,
  rel 46, dolphin 188, hsd 38, musyx 8, crt 38, trk 60. All of them match and link
  (reproduced locally, see phase 0).
- The code is not portable. It was written to match MWCC's PowerPC output byte for byte,
  not to be readable or retargetable.
- In our favour, the 3DS is 32-bit like the GameCube. Pointers have the same size, so
  struct layouts and the in-place pointer relocation of HSD `.dat` files still work. Plain
  `char` is unsigned on both.
- Against us, the 3DS is little-endian. Every file loader has to swap bytes, and MWCC
  bitfields are laid out in the opposite order.
- Most of the work is in three replacement layers: graphics (GX/HSD to the PICA200 through
  citro3d), audio (the MusyX DSP backend to NDSP), and the OS layer (threads, DVD, memory
  card, RELs).
- The target is the New 3DS. The Old 3DS is probably too slow.
- The plan is a separate port repository with this decomp as a submodule, the same model as
  `pkmn-platinum-3ds` and the earlier `pkmn-colosseum-recomp`. A 32-bit PC host build comes
  first, for debugging. The 3DS comes after that.

## 1. Inventory

### 1.1 Size by directory (`src/`)

| Directory | Files | Lines | Built | Role in the port |
|---|---:|---:|---:|---|
| `game/` | 2,032 | 609k | 1,186 | Keep. Game logic, rendering through HSD/GX, menus, battles |
| `rel/` | 45 | 25k (+3.4 MB data) | 46 | Keep. 25 REL modules, linked statically |
| `hsd/` | 38 | 21k | 38 | Keep and adapt. The sysdolphin engine, which talks to GX directly |
| `musyx/` | 26 | 14k | 8 | Keep in part. Keep the synth and sequencer, replace the hardware layer |
| `dolphin/` | 232 | 38k | 188 | Replace with our platform layer (use it as the API reference) |
| `crt/`, `crt_*` | 46 | 9k | 39 | Replace with devkitARM newlib/libc |
| `trk/` | 69 | 5k | 60 | Drop. This is the MetroTRK debugger |

`src/` holds about 2,480 `.c` files, but `configure.py` builds 1,574. The rest are
alternative candidates (`*_candidate_*`, `*_r48_*` and so on) or are pulled in through
`#include`. The port has to use exactly the object list from `configure.py`. Generate that
list; copying it by hand will go wrong.

### 1.2 Hardware API use

Call counts, with the number of files in parentheses, in the code we keep:

| API | `game/` | `hsd/` | `musyx/` | What it is | Replacement |
|---|---:|---:|---:|---|---|
| `HSD_*` | 1,694 (94) | 2,456 | – | 3D engine: JObj/DObj/MObj/TObj/TExp/PObj/anim | Port HSD itself |
| `GX*` direct | 1,620 (51) | 371 (12) | – | GPU commands, TEV, vertex formats | GX emulation layer on citro3d |
| `MTX/PSMTX/PSVEC` | 857 (58) | 319 | – | Matrix math (paired singles) | Plain C versions (the SDK already has C_MTX) |
| `OS*` | 669 (106) | 96 | 20 | Threads, interrupts, alloc, ticks | libctru threads, mutexes, svcGetSystemTick |
| `CARD*` | 146 (4) | – | – | Memory card | Save file on SD |
| `SI/EXI/JOY/GBA` | 144 (28) | – | – | GBA link and serial | Stub out (GBA features are dropped) |
| `DVD*` | 72 (9) | 2 | – | Disc files | stdio on SD/romfs |
| `snd*/sal*` | 63 (8) | – | 770 | MusyX API | Keep MusyX, move `hw*` to NDSP |
| `AI/AR/ARQ/DSP` | 46 (30) | – | 40 | Audio hardware and ARAM DMA | NDSP and plain memcpy (ARAM becomes RAM) |
| `VI*` | 37 (22) | 32 | – | Video timing, framebuffer | gspWaitForVBlank / C3D_FrameBegin |
| `THP*` | 32 (2) | – | – | FMV player (MJPEG) | Own decoder or offline transcode (1.6) |
| `PAD*` | 18 (4) | – | – | Controller | hidScanInput with a mapping |

Direct GX use is concentrated in a few files: `ps_range_80168C64.c` (218), `psdisp.c` (152),
`gs_gfx_layer_state.c` (96), `psdisptev.c` (93), and the `gs_gfx_*`, `gs_light` and
`gs_material` files. That is the particle system and Genius Sonority's own render helpers.
Everything else goes through HSD.

The code uses 189 distinct GX functions, 595 HSD functions and 110 OS functions. The GX
layer has to implement about those 189.

### 1.3 Assembly

- `docs/asm_evidence/registry.json` lists 133 registered asm functions: dolphin 82, trk 21,
  crt 17, game 10, musyx 2, hsd 1.
- Nearly all of it lives in code we replace anyway (SDK, runtime, TRK).
- The 10 in game are mostly in `src/game/gs_thread_exact_800F0030.c` (7, the context switch
  of the game's own task/thread system), plus a few others. They need rewriting, using
  libctru threads or coroutines.
- Most `asm void` blocks in `game/` (about 620) sit behind `#if 0` as historical fallbacks.
  They are not built and do not matter.

### 1.4 Portability of the C code

| Problem | Size | Notes |
|---|---|---|
| Unnamed functions (`fn_8…`) | ~30,100 refs in `game/` | Compiles fine but makes debugging hard. `docs/function_renames.csv` can update names |
| Untyped data (`lbl_8…`) | ~41,600 refs, 4,923 `extern u8/u32 lbl_[]` arrays | Data read as raw bytes or words at offsets. The main source of endianness bugs when the data comes from files or when words are split into fields |
| `0x80xxxxxx` in code | 2,296, about 80% in 5 data files | `snd_volume_data.c` (1,038), `data_802EF0A8.c` (517), `data_8027A500.c` (275), `gs_task_data.c` (207), `fight_ai_data.c` (128). Some are colours or flags; some are pointers stored as numbers and must become `&symbol` |
| Fixed GC memory addresses | A few | For example `*(u32*)0x800000F8` (bus clock). Replace with constants |
| Bitfields | 138 declarations | MWCC/PPC and GCC/ARM order them differently. Only a problem where they overlay file or hardware data; check each struct |
| Pointer↔int casts | ~400 | 32-bit to 32-bit, so usually harmless on the 3DS |
| Matching tricks | Everywhere | `RULE-EXCEPTION`, shaping temporaries, pragmas. Usually harmless under GCC, but `#pragma` and `__declspec` need a compat header |
| C++ | 4 files | Negligible |

### 1.5 REL modules

- The game loads RELs from FSYS resource group `0x0E`: `OSLink`, then `_prolog`
  (`fn_8017F484`). On release it calls `_epilog` (`fn_8017F6B4`).
- The modules were built with SN ProDG (GCC), not MWCC (see `docs/REL_MODULES.md`). Their
  source is in `src/rel/`.
- The decomp covers 25 of the 168 REL modules on the USA disc:
  - covered: 1,404,116 bytes, of which `common_rel` alone is 1,369,520;
  - not covered: 143 modules, 168,656 bytes in total, median 1.1 KB. These are the maps in
    D1–D4, M2–M4, S2 and T1, plus `toolbattle_menu` and `waza_viewer_rel`.

  They are small map scripts in the same ProDG pattern as the M1 modules, so they should be
  easy to decompile. Without them those maps cannot be played in the port. This is a work
  package of its own (phase 1b).
- The port links everything statically. `OSLink` becomes a lookup in a table of
  `{module id → prolog, epilog, symbols}`. Module globals must be reinitialised on
  `_prolog`, because the original loads them fresh every time.

### 1.6 Game data

From the code:

- FSYS archives with LZSS compression. A parser and decoder already exist in
  `tools/fsys_extract.py`; the game's loader is in `src/game/fsys/`.
- HSD `.dat` for models, animations and materials. Same format as Melee, with a relocation
  table for pointers.
- GX textures (CMPR, I4, I8, IA4, RGB5A3, CI…). The PICA200 only takes RGBA8, RGB565,
  RGBA4, RGBA5551, L/A/LA, ETC1 and ETC1A4, at most 1024×1024 and power-of-two sizes, so
  everything needs converting.
- MusyX sound data: `.proj/.pool/.sdir/.samp` with DSP-ADPCM samples. No DVD audio
  streaming was found in `game/`.
- THP videos (MJPEG): `movie/gs_logo.thp`, `tpc.thp`, `openingdemo.thp`,
  `autodemo01.thp` and `staffroll.thp`.

#### USA dump GC6E01 (the reference for the port)

- 1,872 files, 626 MB. 1,852 `.fsys` archives, 596 MB unpacked.
- THP 279 MB, `.samp` 9.4 MB. Music sequences are in `bgm_archive.fsys`.
- `main.dol` SHA-1 is `870e8b96…`, the same as `build.sha1`.

FSYS member types. The type is the third byte of the member ID. The meanings are guesses from
names and magic bytes and have not been confirmed in code:

| Type | Count | Unpacked | Probably | Examples |
|---|---:|---:|---|---|
| 0x02 | 178 | 151 MB | Map and floor models (HSD dat) | `D1_garage_1F`, `carde_bg` |
| 0x20 | 1,098 | 142 MB | Move effects (`wzx_*`) | `10manvolt_attack` |
| 0x1E | 527 | 120 MB | Pokémon and trainer models (`pkx_*`) | `absol`, `achamo` |
| 0x04 | 1,100 | 111 MB | Animation and camera data | `camera_shake_data` |
| 0x12 | 1,487 | 55 MB | Textures/2D (header starts with width and height, e.g. 64×64) | `menu_052` |
| 0x06 | 124 | 4.7 MB | Per-map data | |
| 0x0A | 168 | 4.1 MB | Messages/text | |
| 0x14, 0x24 | 54 | 2.0 MB | Particle data (`GPT1`) | `smallexplosion` |
| 0x0C | 162 | 1.7 MB | Fonts/glyphs | `jis_1`, `system_tool` |
| 0x1C | 169 | 1.5 MB | REL modules | `pocket_menu` |
| 0x0E, 0x00, 0x08, 0x18, … | ~460 | ~2.6 MB | Scripts, collision, camera, sequences | |

What this means for the port:

- The data fits easily on an SD card. After conversion we expect about 0.5–1 GB, depending
  on how the THP videos are transcoded.
- Textures (0x12, and those inside models) are most of the conversion work.
- Models (0x02/0x1E) and effects (0x20) are the three biggest blocks for the endian
  swappers. Walk their struct trees first.

#### PAL dump GC6P01, for comparison

2,006 files, 675 MB, 1,981 `.fsys`, 738 MB unpacked. It is the same data plus language
variants (`cb_menu_fr/ge/it`, `colbtl_e/f/g/i/s`, five copies of some RELs) and a different
`GSfsys.toc`. The decomp, its hashes and the REL source all describe GC6E01. Do not use the
PAL dump for the port or for the matching build.

### 1.7 What already exists

- Toolchain: devkitPro at `C:\devkitPro` with devkitARM, libctru and citro3d. The
  `pkmn-platinum-3ds` workflow is to build through MSYS2 bash and test in Azahar and on a
  New 3DS XL with Luma3DS and the Homebrew Launcher.
- `tools/fsys_extract.py`, so FSYS and LZSS are already documented.
- `docs/function_renames.csv` and `docs/NAMING_SCHEME.md` section 5, with the recipe for
  updating names in a port repository.
- An earlier native recomp (`pkmn-colosseum-recomp`, not on this machine) got to roughly
  call 31 of 55 in the boot sequence (`docs/CURRENT_STATE.md`). If it still exists, its
  platform layer may be worth reusing.

## 2. Target platform

| | GameCube | Old 3DS | New 3DS |
|---|---|---|---|
| CPU | PPC 750CL, 486 MHz, paired singles | ARM11 2×268 MHz | ARM11 4×804 MHz (+L2) |
| RAM for the game | 24 MB + 16 MB ARAM | ~64 MB (96 MB extended) | ~124 MB (178 MB extended) |
| GPU | Flipper: 16 TEV stages, 8 textures | PICA200: 6 combiners, 3 texture units + proctex, programmable vertex shaders | same |
| Screen | 640×480 | 400×240 + 320×240 touch | same |
| Audio | DSP, 64 voices, ADPCM | NDSP, 24 channels, ADPCM (the same DSP-ADPCM format) | same |

What this means:

- Memory is not a problem on the New 3DS.
- CPU and GPU are the bottleneck. Particles, shadows and multi-texture materials will need
  simplifying. The 400×240 screen helps: smaller textures and less fill rate.
- The goal is 30 fps on the New 3DS. What the original reaches in each scene still has to be
  measured.
- The bottom screen could show the battle menu or a PDA/menu. That is optional and belongs
  in a later phase.

## 3. Architecture of the port

```
colosseum-3ds/                     (new repository)
├─ decomp/          <- submodule: this repository, pinned to a commit
├─ platform/
│  ├─ include/dolphin/*.h         <- same API as the SDK, our own implementation
│  ├─ os/   (threads, alarms, ticks, alloc, OSLink table)
│  ├─ dvd/  (files on SD/romfs)
│  ├─ gx/   (GX -> citro3d; GX -> OpenGL for the PC host)
│  ├─ pad/  card/  vi/  ai_dsp/ (MusyX hw -> NDSP / software mixer)
│  └─ thp/
├─ compat/          <- MWCC compat header: asm, pragma, __declspec, intrinsics (__cntlzw, ...)
├─ swap/            <- endian swappers per file format and struct
├─ patches/         <- small, documented changes to decomp files
├─ tools/
│  ├─ gen_objects.py   <- object list from decomp/configure.py
│  └─ convert_assets/  <- ISO -> SD folder: PICA textures, audio, video
└─ build/  3ds/ (devkitARM)   host/ (32-bit GCC/Clang, SDL2 + OpenGL)
```

Principles:

1. Leave the decomp alone. Port changes go in `patches/` or behind `#ifdef PORT`, so the
   matching build stays green and new decomp commits merge easily.
2. Keep the SDK API. Game code still calls `GXSetTevOrder` or `DVDRead`, and the platform
   implements them. That saves thousands of edits in game code.
3. Fix endianness at load time. Data is swapped once, by type, after it is read, so structs in
   memory are native. The alternative, accessor macros across 600k lines, is too invasive.
4. Build the host 32-bit (`-m32` or i686 mingw). Pointer size and layout then match the 3DS
   and the GameCube. A 64-bit build breaks `.dat` relocation.
5. Ship no game data. The converter builds an SD folder (`sdmc:/3ds/colosseum/`) from the
   user's own ISO.

## 4. Roadmap

Size per phase: S = days, M = 1–3 weeks, L = 1–2 months, XL = several months. These are rough
part-time estimates. Expect a year rather than a month in total.

### Phase 0: baseline with a GC6E01 disc (S), done

- Put the ISO in `orig/GC6E01/disc/`, not directly in `orig/GC6E01/` (see
  `docs/REL_MODULES.md`).
- Extract `sys/main.dol` and every `.fsys` that `build.ninja` names (24, not the 5 listed in
  `docs/REL_MODULES.md`), then run `python configure.py && ninja`.
- Take an inventory of the disc data.
- Status 2026-11-06: reproduced on Windows with the USA disc.
  - `ninja` reports `26 files OK` (main.dol and 25 RELs).
  - `configure.py progress` reports 100% (8,677/8,677 functions, 2,256/2,256 units).
  - The `report.json` SHA-256 is `2779ecf8…c7a0`, the same as the README.
  - It needed `pip install --user ninja` and two Windows fixes in `tools/project.py`
    (commit `99687f610`).
  - dtk rewrites `splits.txt` and the `symbols.txt` files on every build (different order,
    comments dropped). Do not commit those changes; restore them with `git checkout`.

### Phase 1: port repository and compiling (M)

- Set up the repository, the submodule and `gen_objects.py`. Add a compat header for the MWCC
  constructs.
- Compile the game code, `rel/`, `hsd/` and the MusyX synth with host GCC in 32-bit mode
  and with devkitARM. Stub the SDK for now (`platform/` with empty functions).
- Replace the 10 game asm functions and the inline intrinsics.
- Turn addresses stored as numbers in data into symbols (the 5 data files in 1.4).
- Done when everything links into one binary, on the host and on the 3DS, even though it
  does nothing yet.

### Phase 1b: missing REL modules (M, can run in parallel)

- Add the 143 small map modules (165 KB in total) to the decomp through the existing dtk
  pipeline: `config.yml` → `modules:`, symbols, splits and ProDG source.
- This happens in the decomp repository itself, with the same matching validation. Start
  with the maps the player reaches first after M1.

### Phase 2: OS layer and boot loop (M)

- Threads, message queues, alarms and ticks. Rewrite the game's own task system from
  `gs_thread_*` first.
- Heaps (`OSAlloc`), with ARAM as a plain RAM buffer.
- Replace `OSLink` with the static module table, including `_prolog`/`_epilog` and data
  reinitialisation.
- Trace the boot to a log and the terminal.
- Done when the host build reaches the title loop without graphics, with the same call order
  as in Dolphin.

### Phase 3: files and endianness (L)

- DVD to a folder on SD (on the host, a local folder). FSYS and LZSS already work through the
  game code itself.
- Swappers per format: the FSYS header, HSD `.dat` (walk the struct tree through
  JObj/DObj/MObj/TObj/PObj/AObj and so on), message tables, Pokémon/move/trainer data, and
  floor/map data.
- Go through the `lbl_` arrays that read file data and type them where needed.
- Add a differential check: compare struct dumps from the host against Dolphin memory.
- Done when every file for the title screen and the first map loads without a crash and the
  dumps agree.

### Phase 4: graphics (XL, the main job)

- The GX layer implements about 189 GX functions against a common "PICA profile":
  - display lists and indexed vertices become single-index vertex buffers, cached per PObj;
  - matrices and lighting run in the PICA vertex shader (GX channels become our own shader);
  - TEV maps to at most 6 combiners, with fallback rules for longer chains and for indirect
    texturing, which the PICA does not have;
  - more than 3 textures per material means multipass or simplifying;
  - a texture cache that converts at load time, or better, ahead of time in the asset
    converter.
- Backend 1 is OpenGL on the host, with the same PICA limits enforced artificially, so you
  can debug with RenderDoc.
- Backend 2 is citro3d on the 3DS.
- Order: title screen (2D), then menus, then the overworld model, then battle models,
  particles (`ps*`), and last shadows and effects.
- 640×480 to 400×240 is handled in the projection. Check the 2D UI for legibility.
- Done when the host build runs from the title screen to the first battle with correct
  images.

### Phase 5: input, saves, timing (M)

- PAD to the 3DS buttons and circle pad. CARD to a save file on SD.
- VI and frame pacing.
- Disable the GBA and e-Reader features (`game/gba/`, 39 files) cleanly.
- Done when a full save/load cycle works.

### Phase 6: audio (L)

- Keep the MusyX synth and sequencer. Replace the `hw*`/`sal*` layer (DSP voices, ARAM DMA)
  with:
  - on the host, a software mixer (ADPCM decoding and mixing) through SDL;
  - on the 3DS, NDSP. It takes the same DSP-ADPCM format but has only 24 channels, so
    voice stealing by priority is needed.
- Reverb and effects are off at first; a cheap version may come later.
- Done when music and sound effects are right in the title, the overworld and battles.

### Phase 7: 3DS bring-up and performance (L–XL)

- Run in Azahar, then on a New 3DS XL. Turn on extended memory mode.
- Profile. Expected hotspots: skinning and animation, particles, matrix math (the old
  paired-single code becomes plain VFP), and the MusyX sequencer.
- Optimisations:
  - a second CPU core (New 3DS) for audio and loading;
  - smaller textures from the converter;
  - LOD and a particle budget.
- THP videos: transcode offline to something the 3DS plays cheaply, or write a simple
  low-resolution MJPEG decoder.
- Done when the game holds 30 fps in the overworld and in battle on a New 3DS.

### Phase 8: full playthrough and polish (L)

- Play the whole story, then Mt. Battle, so that all REL modules get used.
- Make the asset converter easy to use (ISO in, SD folder out), with a hash check of the ISO.
- Add a bottom-screen UI, if wanted.

## 5. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| TEV and multitexture do not fit the PICA200 (6 stages, 3 textures) | High | Visual errors | A fallback table per material pattern. Measure first how many materials fall outside the profile |
| Too slow on the New 3DS in battles and effects | Medium–high | Low frame rate | Profile early (pull part of phase 7 forward), effect budget, second core |
| Endianness bugs in untyped `lbl_` data | High | Subtle gameplay bugs | Differential struct dumps against Dolphin, type the data step by step |
| Behaviour of the game's own thread system (asm context switch) | Medium | Hangs and races | Emulate it cooperatively on one thread first, move to real threads later |
| Bitfield layout on ARM | Low–medium | Wrong flags | Check the 138 declarations and use explicit masks where needed |
| Legal | – | – | Distribute code only. Users convert their own ISO, and no game data ever goes into a repository |
