<!-- SPDX-FileCopyrightText: 2026 hexbus -->
<!-- SPDX-License-Identifier: CC-BY-4.0 -->

# ESE compatibility runtime architecture

## Purpose and authority

This is the concise description of how the current Tomy Tutor/Pyuuta TI-99
compatibility system is laid out and how it runs. It consolidates the active
architecture; it does not replace the byte-level, electrical, or chronological
evidence in the linked documents.

The authoritative detailed address map is the
[compatibility memory-map roadmap](compatibility-memory-map-roadmap.md). The
physical EPROM layout is documented separately in
[Tanam parallel ROM windows](tanam-parallel-rom-windows.md).

## System at a glance

```text
                 TMS9995 CPU
                      |
          +-----------+-----------+
          |                       |
          v                       v
 ESE Game Adapter          Tanam ROM/RAM cartridge
 fixed compatibility BIOS  module storage + GPL RAM
 CPU >0000->3FFF           ROM >4000->5FFF, >8000->BFFF
                           SRAM >6000->7FFF
          |                       |
          +-----------+-----------+
                      |
          Tutor VDP, PSG, keyboard,
          joystick, cassette hardware
```

The ESE replaces the Tutor/Pyuuta system BIOS. The Tanam cartridge holds the
TI module and working RAM. The compatibility BIOS interprets GPL and translates
TI console services into operations on the Tutor's real TMS9918-family VDP,
SN76489-family PSG, keyboard, and controllers. This is a compatibility runtime,
not whole-machine TI-99 emulation.

## Current hardware profile

The working combination uses ESE setting `2/16` and the proved Tanam ROM/RAM
decode profile.

| CPU address | Provider | Current use |
| --- | --- | --- |
| `>0000->3FFF` | ESE | Fixed 16 KiB boot BIOS, GPL interpreter, console services, and hardware translation |
| `>4000->5FFF` | Tanam EPROM | 8 KiB low parallel-ROM window |
| `>6000->7FFF` | Tanam SRAM | GPL scratchpad/module working RAM; `>6000->60FF` implements the TI GPL scratchpad ABI |
| `>8000->BFFF` | Tanam EPROM | 16 KiB high parallel-ROM window |
| `>C000->DFFF` | Tanam/system decode | Additional SRAM window/alias in the current profile; its exact electrical relationship to the single SRAM remains a schematic/alias-test item |
| `>E000->E3FF` | Tutor hardware | VDP and PSG I/O used by the shim |
| `>F000->F0FB` | TMS9995 | On-chip RAM used for interpreter/native state and CPU workspaces |
| `>FFFC->FFFF` | TMS9995 | Reserved reset/vector end of the on-chip RAM map |

The ESE contains a 64 KiB W27C512, but `2/16` exposes a fixed 16 KiB CPU
window. Physical chip capacity and simultaneously visible CPU address space
are different things. The other selected data is not automatically usable by
the running core without different decode or banking logic.

ESE `2/32` is not an easy way to add another 16 KiB to this design: it claims
address space needed by the Tanam cartridge profile. Any future 32 KiB ESE
mode must be designed with a coordinated GAL map or bank-switch contract.

## Tanam EPROM layout

One selected 32 KiB half of the Tanam W27C512 is presented as parallel ROM:

| EPROM offset within the selected half | CPU view |
| --- | --- |
| `0x0000-0x3FFF` | `>8000->BFFF` |
| `0x4000-0x5FFF` | `>4000->5FFF` |
| `0x6000-0x7FFF` | Not ROM in this profile; the corresponding cartridge region is SRAM-selected |

The 64 KiB programmer image repeats the selected 32 KiB layout in both halves
unless a board-specific selector image is intentionally built. That repetition
is why strings can appear more than once in a hex editor.

## Logical GROM versus physical ROM

A TI physical GROM contributes 6 KiB of data inside an 8 KiB logical slot.
The current converter removes each unused 2 KiB tail and packs up to four
physical GROM payloads into the Tanam board's 24 KiB of visible ROM.

```text
TI logical address requested by GPL
                 |
                 v
       virtual-GROM address logic
                 |
       slot and 6 KiB offset
                 |
                 v
 Tanam parallel-ROM CPU window read
```

The module continues to see conventional logical GROM bases such as `>6000`,
`>8000`, `>A000`, and `>C000`. The packed bytes do not live at those CPU
addresses. The BIOS performs the translation on every virtual-GROM fetch.

Four packed GROMs exactly fill the present 24 KiB ROM budget. Five GROMs can
contain 30 KiB of useful data and require future banking or a different
hardware profile.

## Boot and execution flow

1. The TMS9995 resets through vectors supplied by the ESE.
2. The fixed BIOS establishes its workspace and initializes RAM, VDP, PSG,
   timing, keyboard, and controller state.
3. It scans the packed module at the standard logical cartridge GROM bases.
4. It validates `>AA` headers and bounded linked program records.
5. It displays the TOMY title screen and builds the program menu from the
   discovered module names and entry addresses.
6. On selection, it initializes the TI GPL workspace contract and enters the
   module through the GPL interpreter.
7. GPL opcodes read module bytes through virtual GROM. Console calls are
   handled by native compatibility services, and VDP/PSG activity is sent to
   the Tutor hardware.
8. Keyboard and joystick scans are translated to TI-99/4A KSCAN semantics.
9. Console QUIT returns through the compatibility BIOS to the TOMY title
   screen.

The discovery process is data driven. PHM3001 and its entry address are not
hard-coded into the ESE core.

## Fixed core and optional native modules

Pure-GROM modules need only the fixed BIOS and their packed payload. Some TI
cartridges also contain native TMS9900 CPU ROM. For those modules, the current
experimental `TTBI`/`TTEX` design can place relocated TMS9900-family code in
otherwise unused Tanam high ROM and bind it to fixed ESE services.

- `TTBI` is the stable service-vector footer in the 16 KiB ESE core.
- `TTEX` describes and validates an optional native module extension.
- GPL can enter the native extension through the module's normal XML path.
- Native code calls core services through vectors rather than build-dependent
  addresses.

Tombstone City proves this mixed GPL/native path. Parsec is the next, more
demanding case because its native engine performs inline-parameter VDP calls,
keeps long-lived return links in R11, and originally uses speech hardware.
See the [module extension ABI](module-service-extension-abi.md). The Parsec
work remains an engineering experiment and is not part of this public
baseline.

## Responsibility boundaries

| Component | Owns |
| --- | --- |
| ESE fixed core | Reset, title/menu, GPL interpreter, virtual GROM, console services, timing, keyboard/joystick translation, VDP/PSG hardware access, QUIT |
| Tanam EPROM | Packed GROM payload and, where space permits, an optional relocated native extension |
| Tanam SRAM | GPL scratchpad ABI, module variables, and native working state |
| Module | GPL program, graphics/data, module-specific rules, and optional native engine |
| Tutor hardware | Actual CPU, VDP, VRAM, PSG, keyboard, controllers, and future cassette transport |

VDP RAM is not spare firmware storage. Modules own name, pattern, color,
sprite, motion, sound-list, and general VRAM regions and can use nearly all
16 KiB. Persistent interpreter state therefore remains in CPU-accessible RAM.

## Porting rules learned from native modules

These rules are now part of the architecture rather than one-off Parsec
debugging notes:

1. **Preserve R11 across nested native calls.** A generated `BL` inside a
   routine that still needs its caller's R11 destroys the outer return link.
   Use an explicit save/restore or a `BLWP` trampoline with a private
   workspace.
2. **Reconstruct inline service parameters.** TI routines often place data
   words immediately after `BL @SERVICE`. A conservative flow disassembly can
   mislabel those words as instructions. Relocation must understand the call
   contract and relocate pointer values inside the inline block.
3. **Relocate data pointers, not only instruction operands.** Addresses stored
   in tables and inline records are just as important as visible `MOV` or
   `BL` operands.
4. **Preserve layout when code depends on it.** Short branches, copied loops,
   inline parameters, and address tables make casual insertion unsafe. Prefer
   same-size transforms or reassemble from authoritative source with explicit
   assertions.
5. **Keep helper workspaces outside the emulated TI scratchpad.** The mapped
   `>6000->60FF` region belongs to the TI GPL scratchpad contract; private
   native workspaces must not silently overlap it.
6. **Trace service entry and exit state.** Final screen corruption can be far
   removed from the bad call. Record PC, WP, R11, inline parameters, and bus
   addresses at the compatibility boundary.

These constraints are important for Parsec and for future compiled modules
such as TI CPU-ROM games, console GROM translations, and optimized GPL blocks.

## What is current and what is future

Implemented or physically demonstrated:

- Fixed 16 KiB ESE compatibility core at `2/16`.
- Packed one-to-four-GROM Tanam images.
- GPL interpreter and the console-service subset exercised by the qualified
  modules.
- TI-99/4A keyboard/function-key and Tutor joystick translation.
- Experimental mixed GPL/native extensions through `TTBI`/`TTEX`.

Planned, not implied by the current hardware:

- Five-GROM/30 KiB module banking.
- A coordinated 32 KiB ESE profile.
- General CPU-ROM cartridge relocation.
- Complete TI console ROM/GROM service coverage.
- CS1 cassette translation, DSR devices, disk, RS232, speech, and 32 KiB
  expansion-RAM compatibility.
- Semantic GPL-to-native compilation and automatic hot-block optimization.

## Detailed references

- [Documentation index](README.md)
- [Current memory-map roadmap](compatibility-memory-map-roadmap.md)
- [Tanam parallel ROM windows](tanam-parallel-rom-windows.md)
- [Part 1 GPL shim BIOS](part-1-gpl-shim-bios.md)
- [Module extension and fixed-core ABI](module-service-extension-abi.md)
- [TI console ROM service map](ti-console-rom-service-map.md)
- [GPL conformance matrix](gpl-conformance-matrix.md)
- [Joystick translation](joystick-translation-plan.md)
- [Make a Tanam ROM from GROM dumps](grom-to-tanam-rom.md)
