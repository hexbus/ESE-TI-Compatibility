<!-- SPDX-FileCopyrightText: 2026 hexbus -->
<!-- SPDX-License-Identifier: CC-BY-4.0 -->

# Tanam Parallel ROM Windows and W27C512 Byte Layout

## Purpose

This note explains why a TI GROM dump cannot be burned byte-for-byte at offset
zero of the Tanam ROM/RAM cartridge EPROM, why the finished W27C512 image looks
out of sequence in a hex editor, and why many strings appear twice.

The key distinction is between three different address spaces:

1. the TI module's **logical serial-GROM address space**;
2. the shim's **packed 24 KiB payload**;
3. the Tanam board's **parallel W27C512 and CPU-visible ROM windows**.

These are three views of the same bytes, but their addresses are not
interchangeable.

## Serial GROM versus parallel EPROM

A TI GROM is not ordinary CPU ROM. The TI console selects a logical GROM
address and then clocks bytes through a serial data port. One physical GROM
contains 6 KiB of driven data in an 8 KiB logical slot. The final 2 KiB of each
slot is not storage that can be recovered from the device.

The Tanam cartridge instead contains an 8-bit parallel W27C512. For every
enabled CPU byte read, the GAL selects the EPROM and the EPROM drives one byte
directly onto the bus. There is no GROM serial protocol in the cartridge.

The compatibility shim supplies the missing protocol in software:

```text
GPL asks for logical GROM byte >A123
             |
             v
shim identifies the 8 KiB GROM slot and its 6 KiB physical portion
             |
             v
shim converts that logical address to a packed-payload offset
             |
             v
shim reads the byte through CPU >4000->5FFF or >8000->BFFF
```

Thus the Tanam EPROM is storage for the virtual-GROM service. It is not mapped
at the TI GROM logical addresses.

## Proven ESE `2/16` cartridge profile

With the rear ESE set to `2/16`, the current compatibility profile is:

| CPU range | Provider | Purpose |
| --- | --- | --- |
| `>0000->3FFF` | ESE | 16 KiB native shim BIOS |
| `>4000->5FFF` | Tanam EPROM | 8 KiB low parallel-ROM window |
| `>6000->7FFF` | Tanam SRAM | GPL workspace and module RAM window |
| `>8000->BFFF` | Tanam EPROM | 16 KiB high parallel-ROM window |
| `>C000->DFFF` | System/Tanam decode profile | Additional RAM region reported by the board design; its relationship to the single 32 KiB SRAM requires schematic or alias testing |
| `>E000->EFFF` | Pyuuta | VDP, PSG, and other I/O selects |
| `>F000` workspace area | Pyuuta internal/system RAM | Native shim state |

Only two architectural facts matter to the EPROM packer: 8 KiB of cartridge
ROM is visible at `>4000`, and 16 KiB is visible at `>8000`.

## Byte-for-byte parallel-window equations

Within one selected 32 KiB EPROM half, the Tanam GAL presents these blocks:

| W27C512 offset in selected half | Size | CPU view | Meaning |
| --- | ---: | --- | --- |
| `0x0000-0x3FFF` | 16 KiB | `>8000->BFFF` | high parallel-ROM window |
| `0x4000-0x5FFF` | 8 KiB | `>4000->5FFF` | low parallel-ROM window |
| `0x6000-0x7FFF` | 8 KiB | not used as ROM in this profile | the corresponding CPU cartridge region is SRAM-selected, so the image uses erased `0xFF` fill |

The mapping is byte-for-byte inside each window. There is no byte swapping and
no odd/even interleave:

```text
CPU byte >8000 + n  = EPROM selected-half byte 0x0000 + n, 0 <= n < 0x4000
CPU byte >4000 + n  = EPROM selected-half byte 0x4000 + n, 0 <= n < 0x2000
```

The apparently strange order is therefore a **block permutation**, not a byte
permutation.

## From packed GROM payload to CPU windows

PHM3001 contains four physical 6 KiB GROM payloads. Their 2 KiB logical-slot
tails are omitted, producing one contiguous 24 KiB packed payload:

| Logical GROM slot | Driven logical bytes | Packed offsets |
| --- | --- | --- |
| `>6000->7FFF` | `>6000->77FF` | `0x0000-0x17FF` |
| `>8000->9FFF` | `>8000->97FF` | `0x1800-0x2FFF` |
| `>A000->BFFF` | `>A000->B7FF` | `0x3000-0x47FF` |
| `>C000->DFFF` | `>C000->D7FF` | `0x4800-0x5FFF` |

The shim converts a logical address to a packed offset `P`. It then uses:

```text
if P < 0x2000:
    CPU address = >4000 + P
else:
    CPU address = >8000 + (P - 0x2000)
```

The EPROM packer must invert that view. One 32 KiB Tanam selector half is:

```text
EPROM 0x0000-0x3FFF = packed payload 0x2000-0x5FFF
EPROM 0x4000-0x5FFF = packed payload 0x0000-0x1FFF
EPROM 0x6000-0x7FFF = 0xFF fill
```

This is why the first byte of the logical module is found at physical EPROM
offset `0x4000`, not at offset zero.

## Why the W27C512 contains repeated strings

The W27C512 is 64 KiB, but the proven cartridge selector chooses a 32 KiB
half. To ensure that either historical selector position sees the same module,
the build duplicates the complete 32 KiB Tanam bank:

```text
W27C512 0x0000-0x7FFF = selector bank
W27C512 0x8000-0xFFFF = identical selector bank
```

Every GPL string and data table therefore normally appears twice, separated
by `0x8000`. Within each half, the first 16 KiB is the *later* portion of the
packed GROM stream, while the next 8 KiB is the *beginning*. A hex editor thus
shows both repetition and nonsequential text even though the shim reconstructs
one orderly logical GROM stream.

Any additional repeated text within one 32 KiB half comes from the original
module itself, not from the window packer.

## Worked PHM3001 W27C512 layout

| Physical W27C512 range | Contents |
| --- | --- |
| `0x0000-0x3FFF` | PHM packed offsets `0x2000-0x5FFF` |
| `0x4000-0x5FFF` | PHM packed offsets `0x0000-0x1FFF` |
| `0x6000-0x7FFF` | `0xFF` |
| `0x8000-0xBFFF` | duplicate of `0x0000-0x3FFF` |
| `0xC000-0xDFFF` | duplicate of `0x4000-0x5FFF` |
| `0xE000-0xFFFF` | duplicate `0xFF` fill |

The resulting 64 KiB file is a programmer image for the physical parallel
EPROM. It is not a standard TI GROM dump and should not be used as one in a TI
emulator.

## Worked single-GROM layout: Hunt the Wumpus

Hunt the Wumpus is one 6 KiB physical GROM. The logical packed payload is:

```text
packed 0x0000-0x17FF = Hunt the Wumpus GROM bytes
packed 0x1800-0x5FFF = 0xFF
```

Its first selected 32 KiB Tanam bank becomes:

```text
EPROM 0x0000-0x3FFF = 0xFF
EPROM 0x4000-0x57FF = Hunt the Wumpus GROM bytes
EPROM 0x5800-0x7FFF = 0xFF
```

That bank is duplicated at `0x8000-0xFFFF`. Burning the original 6 KiB dump
at W27C512 offset zero would put it in CPU `>8000->97FF`, while the shim expects
the first logical-GROM bytes through CPU `>4000`. The cartridge scanner would
therefore look in the wrong window.

## Packing and verification rules

1. Preserve the raw 6 KiB GROM dumps as source artifacts.
2. Strip or ignore emulator-container tails only when their format is known.
3. Build the 24 KiB logical packed payload first.
4. Apply the 16 KiB-high, 8 KiB-low Tanam block permutation.
5. Fill the non-ROM 8 KiB portion with `0xFF`.
6. Duplicate the 32 KiB bank into both W27C512 halves unless a deliberately
   different selector profile is being built.
7. Verify size and SHA-256 before programming, then read the device back and
   compare it byte-for-byte with the generated file.

For a concise programming walkthrough, see
[Making a Tanam W27C512 ROM from TI GROM Dumps](grom-to-tanam-rom.md).

The general-purpose implementation is `tools/make_tanam_grom_module.py`. It
accepts one to four individual GROM dumps, an already-compacted 6/12/18/24 KiB
payload, or an 8/16/24/32 KiB image containing complete logical 8 KiB slots.
For example:

```powershell
# One physical GROM; repeat --grom up to four times in logical slot order.
python tools\make_tanam_grom_module.py `
  --grom "Hunt the wumpus.bin" `
  --output build\hunt-w27c512.bin `
  --manifest build\hunt-w27c512.json

# Four already-compacted physical GROM payloads (24 KiB total).
python tools\make_tanam_grom_module.py `
  --packed build\phm3001-payload-emulator.bin `
  --output build\phm3001-w27c512.bin

# A conventional file containing one to four complete 8 KiB logical slots.
python tools\make_tanam_grom_module.py `
  --slot-image module-32k.g `
  --output build\module-w27c512.bin `
  --payload-output build\module-logical-24k.bin
```

`tools/make_hardware_candidates.py` remains the PHM3001/ESE release packager,
and `tools/make_tanam_single_grom.py` remains a compatible focused helper for a
single 6 KiB experiment.

## What remains electrically open

The ROM-window layout above is operationally proven by the PHM3001 physical
runs and reproduced by the current build. The exact GAL equations and the
single Sony CXK58257 32 KiB SRAM's full alias/bank behavior should still be
documented from the requested schematic or measured with a dedicated SRAM
alias test. That open SRAM question does not change the EPROM byte placement
described here.
