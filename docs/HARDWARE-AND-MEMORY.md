# Hardware and memory map

## Boards

The current system uses two cooperating boards:

- The **ESE Game Adapter** overrides the Tomy system ROM and provides the
  fixed compatibility BIOS.
- The **Tanam ROM/RAM cartridge board** provides module ROM and working SRAM.

The canonical setting is ESE `2/16`. Although the ESE contains a 64 KiB
W27C512, that setting exposes one fixed 16 KiB CPU window. Chip capacity is
not the same as simultaneously visible CPU address space.

## Current CPU map

| CPU address | Provider | Use |
| --- | --- | --- |
| `>0000->3FFF` | ESE | 16 KiB reset BIOS, GPL interpreter, services, and Tutor hardware translation |
| `>4000->5FFF` | Tanam EPROM | 8 KiB low ROM window |
| `>6000->7FFF` | Tanam SRAM | GPL scratchpad and module/native working RAM |
| `>8000->BFFF` | Tanam EPROM | 16 KiB high ROM window |
| `>C000->DFFF` | Tanam/system decode | Additional SRAM window/alias; exact electrical aliasing remains a schematic/test item |
| `>E000->E1FF` | Tutor | TMS9918-family VDP access |
| `>E200->E3FF` | Tutor | SN76489-family PSG access |
| `>F000->F0FB` | TMS9995 | On-chip interpreter/native state and workspaces |
| `>FFFC->FFFF` | TMS9995 | Reserved reset/vector end of on-chip RAM |

The first 256 bytes of Tanam SRAM, `>6000->60FF`, implement the TI GPL
scratchpad contract. Native helpers must use other workspace addresses.

## Tanam W27C512 layout

Within one selected 32 KiB half of the EPROM:

| EPROM offset | Size | CPU view |
| --- | ---: | --- |
| `0x0000-0x3FFF` | 16 KiB | `>8000->BFFF` |
| `0x4000-0x5FFF` | 8 KiB | `>4000->5FFF` |
| `0x6000-0x7FFF` | 8 KiB | Not ROM in this profile; the cartridge region is SRAM-selected |

The programmer image normally repeats this 32 KiB layout in both halves of
the W27C512. Repeated strings and apparently reordered data in a hex editor
are therefore expected.

## Capacity

Four physical GROM payloads fit exactly:

| Logical GROM | Logical slot | Useful bytes |
| --- | --- | ---: |
| GROM 3 | `>6000->7FFF` | 6 KiB |
| GROM 4 | `>8000->9FFF` | 6 KiB |
| GROM 5 | `>A000->BFFF` | 6 KiB |
| GROM 6 | `>C000->DFFF` | 6 KiB |

A conventional five-GROM cartridge can contain 30 KiB, which exceeds the
present unbanked 24 KiB module ROM space. Supporting it requires coordinated
banking or a different decode profile.

ESE `2/32` cannot simply be substituted to gain space because it claims CPU
address space used by the Tanam arrangement. A future 32 KiB ESE must be
designed together with the cartridge GAL map and bank-switch mechanism.

## Why VRAM is not replacement RAM

The Tutor has 16 KiB of VDP RAM, but modules control its screen, pattern,
color, sprite, motion, sound-list, and general-data areas. A module can use
nearly all of it and can change its layout dynamically. Persistent GPL and
native runtime state must therefore remain in CPU-addressable RAM.

## Hardware caution

- Preserve original programmed devices and use spare W27C512 parts.
- Treat an ESE and native-module image as a matched build.
- Verify programmed devices by reading them back and recording hashes.
- Do not infer new banking or SRAM alias behavior solely from nominal chip
  capacities; prove the GAL decode and board wiring first.
