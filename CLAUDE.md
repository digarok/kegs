# KEGS 1.38 — Kent's Emulated GS

Apple IIgs emulator by Kent Dickey. Licensed under GNU GPL v3.

## Build

```bash
cd src

# macOS native (default — vars symlink points to vars_mac):
make -j 20          # produces KEGSMAC.app

# Linux:
rm vars; ln -s vars_x86linux vars
make -j 20          # produces xkegs

# macOS X11 (requires XQuartz):
rm vars; ln -s vars_mac_x vars
make -j 20          # produces xkegs
```

Windows builds use Visual Studio 2022 (`kegswin.vcxproj`).

## Project Layout

```
src/                  All source code (C + Swift)
  Makefile            Main build file
  vars                Symlink to platform vars file (vars_mac, vars_x86linux, vars_mac_x)
  ldvars              Object list and PROJROOT
  dependency          Header dependency rules
  comp_swift          Shell script wrapping swift frontend compiler (Swift 4)
  cp_kegs_libs        Perl script to bundle Swift dylibs into .app
  defc.h              Master header — includes defcomm.h, protos.h, etc.
  defcomm.h           Shared defines, struct macros, type aliases
  config.h            Compile-time config constants
  engine.h            CPU inner loop (included as code, not just declarations)
doc/                  README and internals documentation
lib/                  macOS app bundle assets (icons, NIB, etc.)
config.kegs           Runtime configuration (disk slots, speed settings)
```

## Architecture Overview

- **CPU**: 65816 emulation in `engine_c.c` / `engine.h`. The `engine.h` file contains actual function body code included multiple times with different preprocessor contexts for specialization.
- **Main loop**: `run_16ms()` in `sim65816.c`, called every ~1ms from Swift via `DispatchQueue.main.asyncAfter`. Simulates one VBL frame (~17,030 cycles at 1MHz).
- **Memory**: Page-table system with pointer arrays. Upper bits of pointers encode I/O/shadow/breakpoint flags. Accessed via `GET_PAGE_INFO_RD/WR` macros.
- **Events**: Sorted linked list of `Event` structs with `dword64` cycle timestamps (types: `EV_60HZ`, `EV_STOP`, `EV_SCAN_INT`, `EV_DOC_INT`, `EV_VBL_INT`, `EV_SCC`, `EV_VID_UPD`, `EV_MOCKINGBOARD`).
- **Video**: Device-independent rendering in `video.c` into `Kimage` 32-bit ARGB buffers. Platform drivers blit to screen.
- **Sound**: Ensoniq DOC chip in `doc.c`, Mockingboard in `mockingboard.c`, platform drivers via ring buffer.
- **Disks**: IWM (`iwm.c`) for 5.25"/3.5", SmartPort (`smartport.c`) for hard drives, WOZ format (`woz.c`), DynaProFS (`dynapro.c`) for live directory-as-ProDOS-volume.
- **Serial**: SCC (`scc.c`) with TCP/IP modem emulation and real serial port support.
- **Swift-C bridge**: `Kegs-Bridging-Header.h` includes `defc.h`. Swift calls C functions directly.

## Key Source Files

| File | Purpose |
|------|---------|
| `sim65816.c` | Main loop, events, interrupts, `main()`/`kegs_init()` |
| `engine_c.c` | 65816 CPU execution engine |
| `moremem.c` | Memory banking, soft switches, I/O at $C000-$C0FF |
| `video.c` | All video mode rendering |
| `sound.c` / `doc.c` | Sound synthesis, Ensoniq DOC |
| `mockingboard.c` | Mockingboard card emulation |
| `iwm.c` | Disk drive controller (5.25" and 3.5") |
| `smartport.c` | SmartPort hard drive emulation |
| `woz.c` | WOZ disk image format |
| `dynapro.c` | DynaProFS (Unix dir as ProDOS volume) |
| `adb.c` | Apple Desktop Bus (keyboard, mouse) |
| `config.c` | Config UI and config.kegs parsing |
| `scc.c` | Serial communications controller |
| `debugger.c` | Built-in debugger/disassembler |
| `AppDelegate.swift` | macOS app entry, window management |
| `MainView.swift` | macOS NSView: drawing, keyboard, mouse |

## Coding Conventions

- **Indentation**: Tabs (8-space tab width), K&R brace style
- **Global variables**: `g_` prefix (e.g., `g_c068_statereg`, `g_halt_sim`)
- **Functions**: `module_verb_noun` (e.g., `iwm_move_to_ftrack`, `scc_add_to_readbuf`)
- **Structs**: Defined via `STRUCT(Name)` macro → `typedef struct Name_st Name;`
- **Types**: `byte` (u8), `word16` (u16), `word32` (u32), `dword64` (u64)
- **Constants**: `ALL_CAPS_WITH_UNDERSCORES`
- **Debug logging**: Conditional printf macros gated on runtime `Verbose` bitmask (e.g., `iwm_printf`, `scc_printf`)
- **RCS IDs**: Each `.c` file uses `INCLUDE_RCSID_C` / `defc.h` pattern to embed `$KmKId:` version strings

## Dependencies

- **macOS**: Xcode (clang + swift), Cocoa/CoreGraphics/CoreAudio frameworks, Perl
- **Linux**: GCC/clang, X11 (libX11, libXext), PulseAudio, Perl
- **Windows**: Visual Studio 2022, DirectSound/WinMM/GDI32/WS2_32
- No third-party C libraries. Decompression (gzip, zip, ShrinkIt) is implemented in-tree.

## Running

Requires an Apple IIgs ROM file. Place `ROM.03` (256KB IIgs ROM) in the working directory or configure path in `config.kegs`. Ships with `NUCLEUS03.gz` (ProDOS boot) and `XMAS_DEMO.gz` demo disk images.
