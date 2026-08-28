<!-- SPDX-FileCopyrightText: 2026 hexbus -->
<!-- SPDX-License-Identifier: CC-BY-4.0 -->

# Module extension and fixed-core service ABI

## Implemented experimental version

`TTBI 1.0` and `TTEX 1` are implemented as the first relocatable mixed-module
proof. PHM3052 Tombstone City has exercised the matched ESE/module pair in
both emulator and physical Tutor testing. RC6 preserves that hardware-tested
candidate, while the extension interface remains experimental. Ordinary
pure-GROM images do not require a `TTEX` descriptor.

The fixed 16 KiB ESE core ends with this six-byte service table/footer:

```text
CPU >3FFA   word        service 2 entry address, big endian
CPU >3FFC   01 02       TTBI major version 1, two service vectors
CPU >3FFE   word        service 1 entry address, big endian
```

Service 1 advances the common frame/TIME/sound/quit state and then runs the
shared Tutor-to-TI-99/4A KSCAN/joystick translator. A native caller uses the
vector rather than a build-dependent label:

```asm
       MOV  @>3FFE,R1
       BL   *R1
```

It returns the TI `KEYCOD` value in the high byte of R0 and updates the mapped
GPL `JOYY`/`JOYX` state. The Tombstone transformer additionally copies R0 to
the game's mapped `KEY` byte. Service 1 is deliberately small; it proves the
binding mechanism without pretending that the larger semantic registry is
finished.

Service 2, vectored through `>3FFA`, advances frame/TIME/sound/quit state
without scanning the keyboard. Tombstone calls it at the five source points
where the TI version used `LIMI 2` so the VDP interrupt could advance the
timer and sound. Generated callers preserve native R11 around this BL; failing
to do so replaces the caller's return link and resumes bulk VDP helpers at the
wrong instruction. Native busy waits must not let several 60 Hz intervals pass
before making this call: Tombstone's long `KILTIM` call sites are divided into
sub-frame chunks so VDP status, `TIME`, and the PSG list are serviced between
chunks instead of coalescing multiple elapsed frames into one tick.

## Purpose

The compatibility BIOS needs a generic way for a converted TI module to carry
native Tutor support code that does not belong in the fixed 16 KiB ESE core.
Initial use is an Adventure image containing the CS1 cassette front end and
codec. Later uses may include BASIC or Extended BASIC support, DSR handlers,
additional console-ROM services, and bank-switch gateways.

This is an optional extension to the parallel Tanam image. It does not alter
the original TI GROM bytes. A conventional converted module contains erased
bytes at the extension probe and behaves exactly as it does now.

The first packer, validator, and deterministic call test now agree on the
version-1 layout below. The current runtime consumes only the biased module
entry pointer; `TTEX` supplies validation and tooling metadata. A future core
must validate the full descriptor before dispatching general capabilities.

## Why the current mapping can hold extensions

The virtual-GROM payload is compacted to 6 KiB per physical GROM. Its first
8 KiB is visible through CPU `>4000–>5FFF`; any remainder begins at CPU
`>8000`. The current four-GROM limit gives these free high-ROM regions:

| GROM payload | High ROM used by GROM data | Available native extension area |
|---:|---|---|
| 1 × 6 KiB | none | `>8000–>BEFF` (16,128 bytes) |
| 2 × 6 KiB | `>8000–>8FFF` | `>9000–>BEFF` (12,032 bytes) |
| 3 × 6 KiB | `>8000–>A7FF` | `>A800–>BEFF` (5,888 bytes) |
| 4 × 6 KiB | `>8000–>BFFF` | none |

The final 256 bytes, CPU `>BF00–>BFFF`, are reserved for the optional
descriptor. A packer may emit an extension only when both its code/data and
the descriptor lie entirely after the compacted GROM payload. A four-GROM
image cannot use this unbanked extension format.

Adventure is a particularly good first case: its one GROM occupies only
`>4000–>57FF`, leaving the whole high-ROM window for native CS1 support.

## Current XML entry rule

Tombstone's stock GROM executes `XML >70` at logical `>6065`. On a TI-99/4A,
the console XML vector selects the cartridge CPU-ROM entry. In the Tutor core,
selector `>70` reads the word at CPU `>BFFE`, increments it, and branches to
the result. The packer stores `(entry - 1)` there. Consequently an erased
pure-GROM cartridge reads `>FFFF`, increments to zero, and falls through to
the ordinary visible unsupported-service fault instead of branching through
erased ROM.

This first entry is non-returning: Tombstone transfers from GPL into its
native game body. General returning XML services need a later call gate with
an explicit save/restore contract.

## Future full discovery rule

After identifying a valid cartridge, the ESE reads CPU `>BF00`. It treats the
area as an extension only if all of the following pass:

1. four-byte magic and format version match;
2. header and entry counts fit within the 256-byte descriptor;
3. every referenced ROM range lies within the cartridge's proven unused
   high-ROM region;
4. no entry overlaps compacted GROM bytes, the descriptor, SRAM, ESE ROM, or
   I/O space;
5. the descriptor and declared extension bytes pass their checksums; and
6. every required ABI version is supported by the ESE.

Failure means **no extension**. The normal cartridge still boots; the ESE must
not jump through an unvalidated pointer or infer services from a module title.

## Frozen TTEX v1 descriptor

The proposed `TTEX` header records:

```text
offset  size  meaning
>00     4     magic "TTEX"
>04     1     descriptor version = 1
>05     1     header length = >20
>06     1     entry count = 1
>07     1     flags; bit 0 means native extension present
>08     2     extension start CPU address, big endian
>0A     2     extension end CPU address, exclusive
>0C     2     module entry CPU address
>0E     2     unsigned byte-sum of native extension modulo 65536
>10     1     service ID 1: module.main.native
>11     1     required TTBI major = 1
>12     1     required TTBI minor = 0
>13     1     flags; bit 0 means direct CPU address
>14     2     service entry CPU address
>16     2     reserved, zero
>18     8     first eight bytes of native SHA-256
```

The descriptor occupies CPU `>BF00–>BF1F`; `>BF20–>BFFD` remains reserved and
the biased XML-entry pointer is at `>BFFE`. The executable extension must fit
entirely below `>BF00` and must not overlap compacted GROM bytes.

Each capability record needs at least a service identifier, ABI version,
flags, entry bank, entry address, and declared CPU/VDP workspace requirement.
The first implementation should use unbanked entry bank zero. Reserving the
bank field now allows a future GAL/banked cartridge to preserve the discovery
and call-gate contract.

Candidate capability identifiers include:

- TI low-level cassette write/read/verify selectors;
- `CS1` PAB/DSR dispatcher (with `CS2` deliberately absent);
- console-ROM service bundle;
- console GROM 1/2 data or service provider;
- BASIC runtime;
- Extended BASIC runtime; and
- generic DSR device table.

The list is a registry, not a promise that all capabilities exist now.

## ESE call gate

The fixed ESE core remains responsible for detection, bounds checking, ABI
selection, and preserving the interpreter state around a native service call.
Extension code owns only the capability it advertises. It must return through
one defined gate with documented results; it must not replace the interrupt
vectors or silently retain a different workspace.

The call gate must define:

- saved native registers and workspace;
- GPL PC, GROM address, condition/status, and KSCAN state preservation;
- VDP address/prefetch side effects;
- PSG and interrupt side effects;
- allowed CPU RAM and VDP workspace ranges;
- timeout/error return for absent physical hardware; and
- bank restoration before returning to GPL.

This lets Adventure's GPL remain byte-identical while its cassette calls are
routed to cartridge-resident native code. It also prevents a growing list of
cartridge-specific branches in the 16 KiB shim.

## Packaging and validation

`make_tanam_grom_module.py` accepts `--native-extension`, `--native-address`,
and `--native-entry`. It rejects overlap, inserts the code, writes `TTEX` and
the biased pointer, performs the normal Tanam block permutation, duplicates
the selector bank, and records every range and hash in its JSON manifest.

`verify_tombstone_relocatable.py` independently checks the core footer, stock
`XML >70` bytes, unchanged GROM payload, transformed native source, TTEX
fields, checksums, pointer bias, parallel-window permutation, duplicate bank,
and manifest hashes. The JS99er profile then proves transfer to native code,
TTBI keyboard selection, and entry into actual Tombstone gameplay.

Before hardware use, deterministic tests must cover:

- no descriptor (`0xFF`) and bad magic;
- unsupported version;
- truncated header or excessive entry count;
- GROM/descriptor/code overlap;
- entry outside a visible ROM window;
- bad descriptor and code checksums;
- supported and unsupported capability versions;
- normal return, service error, and timeout;
- bank restoration; and
- unchanged boot of an ordinary module.

## First proof completed: Tombstone City

The first proof keeps Tombstone's stock 6 KiB GROM byte-for-byte, mechanically
relocates its recovered native source to CPU `>8000`, binds its one console
keyboard call through TTBI service 1, and replaces five TI interrupt-enable
boundaries with TTBI service 2. The original TI scratchpad state moves
from `>83xx` to Tanam SRAM `>60xx`, direct VDP accesses move to Tutor ports
`>E000/>E002`, with the Tutor recovery cadence after every direct control/data
access. Level-2 interrupts remain masked because the two TTBI services poll
the frame source. The generated entry is `>805C` and the current native body
ends at `>96CF`.

The generated port also hard-bounds ship, projectile, and monster candidates
to name-table rows 3 through 20 and columns 2 through 29. This is defense in
depth behind the original painted-wall collision test: a stale tile read can
no longer let FIRE blank a barrier or let an object walk into title/HUD/VDP
table storage. If the old monster-cell pattern is unreadable, the fallback now
uses authoritative `MONTYP`; it cannot silently convert a large Morg to the
small-monster/Tumbleweed pattern.

The deterministic run reaches the stock level menu, accepts physical Tutor
key `1`, sets novice speed exactly, and reaches the `DAY / POPULATION /
SCHOONERS` gameplay HUD after executing the cartridge-native body. The upward
boundary stress records 291 proposed moves, rejects 282 boundary attempts,
commits eight legal destinations (`111..335`), and records no illegal commit.
The firing stress executes 30 projectile candidates and 25 writes with no
out-of-bounds write while preserving all 23 mountain/barrier tiles. This is an
intermediate mixed-module result, not the separately planned all-GPL port.

## Follow-on proof: Adventure plus CS1

1. Validate unmodified Adventure through its pre-cassette paths.
2. Implement and test CS2 rejection and the CS1 in-memory codec.
3. Link that service for the free Adventure high-ROM region.
4. Generate a `TTEX` descriptor pointing to the CS1 call gate.
5. Confirm the base GROM hash is unchanged.
6. Test PAB operations and direct GPL cassette selectors in the emulator.
7. Qualify one short record on Tutor tape hardware before a complete
   adventure load.

The same mechanism can later host BASIC/XB or other DSR bundles. Large
runtimes may exceed the free unbanked region; they will require the planned
banked cartridge evolution, but the descriptor and ESE call gate can remain
the same.

## Preserving a 32 KiB ESE evolution

The service ABI must not assume that all optional code always lives on the
Tanam cartridge. A later rear ESE set to a 32 KiB decode may hold the fixed
core plus substantially more console services, GROM data, DSR front ends, or
a resident BASIC layer. The same capability registry can search built-in ESE
providers first and then a module `TTEX` descriptor.

A 32 KiB ESE is not automatically additive memory. If it directly decodes CPU
`>0000–>7FFF`, it overlaps the present Tanam low-ROM window at
`>4000–>5FFF` and GPL/SRAM window at `>6000–>7FFF`. Both boards must never
drive the same bus cycle. That profile therefore needs one of these proven
hardware arrangements:

1. disable the overlapping Tanam selects and use only its non-overlapping high
   ROM and separately decoded RAM;
2. move the required writable workspace to a verified non-overlapping SRAM
   window; or
3. change the GAL so a larger physical ESE ROM is banked behind the existing
   16 KiB CPU window at `>0000–>3FFF`.

The third option preserves the currently proven CPU map while allowing
multiple 16 KiB ESE banks. The first two may be simpler with the existing
`2/32` switch setting, but require schematics or bus-select measurements before
use.

Most importantly, 32 KiB of ESE **ROM** does not satisfy software that requires
the TI 32 KiB **RAM expansion**. TI Logo, TI-Writer, and Editor/Assembler still
need a writable compatibility mapping and correct address semantics. The
32 KiB ESE can provide the code that manages that mapping, but the storage must
come from SRAM (or a proven virtual-memory scheme where the caller permits it).

Accordingly, version 1 reserves a provider/location field and a bank number in
each capability record. An unbanked module uses provider `module` and bank
zero; a future ESE-resident or banked service can use the same service ID and
call gate without changing cartridge software.
