# Tutor TI compatibility memory-map roadmap

This document separates the map that is proved on the existing ESE and Tanam
ROM/RAM cartridge from a proposed scalable map.  The proposal is not a claim
that the present GAL or PCB can bank-switch; that requires the schematic and
a write-latch connectivity proof.

## What exists now

| CPU address | Present owner | Use |
| --- | --- | --- |
| `>0000->3FFF` | ESE, setting `2/16` | Always-visible 16 KiB compatibility BIOS |
| `>4000->5FFF` | Tanam cartridge ROM | 8 KiB of packed virtual-GROM payload |
| `>6000->7FFF` | Tanam cartridge SRAM | 8 KiB SRAM window; the shim currently uses `>6000->60FF` for TI GPL scratchpad |
| `>8000->BFFF` | Tanam cartridge ROM | 16 KiB of packed virtual-GROM payload |
| `>C000->DFFF` | Tanam cartridge SRAM | Mirror/window of cartridge SRAM under the proved GAL profile |
| `>E000->E3FF` | Tutor hardware | VDP and PSG I/O regions used by the shim |
| `>F000->F0FB` | TMS9995 | 252 contiguous bytes of on-chip RAM |
| `>FFFC->FFFF` | TMS9995 | Final four bytes of the CPU's 256-byte on-chip RAM map; keep reserved for vector/reset behavior |

The two Tanam ROM windows total 24 KiB.  The current packer removes each
serial GROM's unused 2 KiB tail and places four 6 KiB payloads in those
windows.  Demonstration is therefore a natural exact fit.

The TMS9995 really does contain 256 bytes of RAM, but it cannot replace the
cartridge SRAM.  The shim already uses `>F000->F075` for interpreter/native
state and `>F0A0->F0BF` as its 16-register CPU workspace.  Only 102 bytes are
currently unallocated in the contiguous on-chip region, split into 42- and
60-byte pieces.  Even an otherwise empty TMS9995 would need 256 bytes for the
TI GPL scratchpad plus 32 bytes for a non-overlapping CPU workspace.  The
external RAM requirement therefore cannot be eliminated by repacking the
on-chip RAM.

## Logical GROM capacity

A conventional module can occupy logical GROM slots 3 through 7:

| Slot | Logical range | Useful payload |
| --- | --- | --- |
| GROM 3 | `>6000->7FFF` | 6 KiB |
| GROM 4 | `>8000->9FFF` | 6 KiB |
| GROM 5 | `>A000->BFFF` | 6 KiB |
| GROM 6 | `>C000->DFFF` | 6 KiB |
| GROM 7 | `>E000->FFFF` | 6 KiB |

That is 30 KiB of useful module data in five 8 KiB logical slots.  VDP RAM is
not part of this address space.  The interpreter may use VDP RAM when GPL
explicitly addresses VDP memory, but permanent shim state must not consume it:
modules control the name, pattern, color, sprite, motion, sound-list, and
general data areas and may use nearly all 16 KiB.

## Recommended scalable design

Keep the following resources simultaneously available:

1. The ESE's 16 KiB BIOS at `>0000->3FFF` as the always-visible boot,
   virtual-GROM interpreter, bank-switch trampoline, and hot native-service
   kernel.
2. The full 8 KiB Tanam SRAM window at `>6000->7FFF`.  Initially only the
   first 256 bytes need to implement the console GPL scratchpad ABI, but
   later CPU-ROM modules, Extended BASIC, and diagnostics benefit from the
   remaining RAM.
3. Banked ROM in the existing `>4000->5FFF` and `>8000->BFFF` windows.  A
   page register selects flash pages without requiring every logical GROM or
   service to be visible at once.

Suggested logical flash contents are:

| Flash bank class | Contents |
| --- | --- |
| System bank | Exact console GROM 0, keyboard/font/sound reference tables, and cold native services |
| Module banks | Up to 30 KiB of compact GROM payload, with logical-address metadata |
| CPU-ROM banks | Module `.C`/`.D` components and later service extensions |
| Recovery/test bank | Signatures, RAM tests, and a minimal known-good module |

`VGREAD` already converts a logical GROM address into a physical packed-ROM
offset.  Extending that conversion to select a flash page preserves the GPL
program's addresses and requires no cartridge-specific call patching.

Moving the exact 6 KiB console GROM 0 image from the ESE to a system flash
bank would recover about 6 KiB in the 16 KiB BIOS for native ROM-service
implementations and conformance checks.  GROM 0 can still appear at logical
`>0000` to GPL; only its physical storage changes.

The original 8 KiB TI console ROM is useful as the behavioral oracle, but it
is not a drop-in executable extension.  TMS9900/TMS9995 instructions are
compatible, while the ROM's absolute TI hardware addresses, interrupt model,
and scratchpad assumptions are not.  Reusable service code should therefore
be a native compatibility service ROM with stable shim entry points, built
from byte-verified behavior, rather than an unmodified console-ROM image.

## Why not use ESE `2/32` as the expansion?

In `2/32` mode the ESE occupies `>0000->7FFF`.  That collides with the Tanam
ROM at `>4000->5FFF` and SRAM at `>6000->7FFF`.  The extra ESE bytes are useful
only if new decode/banking logic prevents that simultaneous collision.  There
is no free fixed 8 KiB CPU window in the present map.

## Hardware proof required before a GAL change

1. Obtain the schematic or continuity-map every GAL input/output, flash
   address pin, `/WE`, `/OE`, and cartridge select.
2. Prove whether a data bit and write strobe reach a registered GAL macrocell
   or an external latch.  A purely combinational GAL cannot remember a
   software-selected bank.
3. Preserve an always-reachable recovery bank and verify power-on bank state.
4. Run per-window ROM signatures and destructive SRAM tests before executing
   GPL.
5. Prove bank changes cannot select ROM and SRAM onto the bus together.

Until those checks pass, the current fixed 24 KiB ROM plus 8 KiB SRAM profile
remains the hardware authority.

## Back-burner relocatable runtime

Future banked storage should not make translated cartridges depend on another
set of fixed addresses. The proposed
[relocatable compiled-GPL architecture](relocatable-compiled-gpl-architecture.md)
uses symbolic service identifiers and a versioned vector table so the same
module representation can resolve to fixed ESE services, banked cartridge
services, or native TI implementations. This remains a research constraint,
not a change to the present 16 KiB ESE or 24 KiB Tanam ROM map.
