# TI console ROM service map for the Pyuuta compatibility layer

## Verified authority

The TI console ROM used by this project is the 8 KiB image
`994arom.bin`, SHA-256
`599DA51E9E1968A806871D681F17B5ACBB617ACCF07191891265AEE44EBEC2B6`.
It is byte-for-byte identical to
`TMS9900Family/Consoles/TI-99/99-4A/ROM/rom0-originaldump.bin`.

The annotated source in that directory assembles as three contiguous pieces
which concatenate to the verified image without a byte difference:

| Source | CPU range | Size | SHA-256 | Contents |
| --- | --- | ---: | --- | --- |
| `ROM-4A_A.asm` | `>0000->0D19` | 3,354 | `0ABC0C4B18058A1AE106FD4B214BAD71FCEC6E47DBF2D562C3D6C07B14FE8752` | Reset/interrupt vectors, GPL interpreter, keyboard, VDP/GROM primitives, sound arming, XML dispatch, VBlank/sound/sprite service, ROM/GROM scans |
| `ROM-4A_B.asm` | `>0D1A->15D3` | 2,234 | `5CFD7FCD368C83805F5D64EB23D83FA69057F3195E29A8410B9AA293B8A59D4E` | Floating-point package, internal XML tables, cassette read/write/verify |
| `ROM-4A_C.asm` | `>15D4->1FFF` | 2,604 | `3B627599A1AC9171965C8404BF8EA35E1D7BC0E32D3994B42700012A23421BE5` | Console BASIC interpreter support and internal XML targets |

This source/listing pair is the behavioral authority. A top-down heuristic
disassembly is not needed for the internal ROM.

## Service surfaces software can reach

### GPL interpreter

The interpreter begins at `ENTRY >0024`; `NEXT >0070` fetches and dispatches
the next GPL opcode. The dispatch tables at `>0C36`, `>0C3E`, `>0C7E`, and
`>0CAC` cover the complete GPL instruction set, including:

- control flow, calls, returns, short and long branches;
- byte and word arithmetic, comparison, shifts, logic and stack operations;
- CPU, GROM, VDP and VDP-register addressing modes;
- `MOVE`, `FORMAT`, `SCAN`, `BACK`, `ALL`, `RAND`, `I/O`, and `XML`.

For general cartridge compatibility, the custom interpreter must agree with
these tables and operand decoders, not merely execute the opcodes observed in
PHM3001.

### XML native calls

`XML >0608` converts the operand into one of sixteen 16-entry native vector
tables. `XMLTAB >0CFA` supplies these bases:

| XML high nibble | Vector-table base | Ownership |
| ---: | ---: | --- |
| `0` | `FLTTAB >0D1A` | Console floating-point functions |
| `1` | `XTAB >12A0` | Console conversion, BASIC and scan/link helpers |
| `2` | `>2000` | Expansion-memory/native extension vectors |
| `3` | `>3FC0` | Expansion vectors |
| `4` | `>3FE0` | Expansion vectors |
| `5` | `>4010` | Peripheral ROM vectors |
| `6` | `>4030` | Peripheral ROM vectors |
| `7` | `>6010` | Cartridge ROM vectors |
| `8` | `>6030` | Cartridge ROM vectors |
| `9` | `>7000` | Cartridge ROM vectors |
| `A` | `>8000` | Expansion/native vectors |
| `B` | `>A000` | Expansion/native vectors |
| `C` | `>B000` | Expansion/native vectors |
| `D` | `>C000` | Expansion/native vectors |
| `E` | `>D000` | Expansion/native vectors |
| `F` | `>8300` | Scratchpad-resident vectors |

The low nibble selects a word in the chosen table. Internal XML groups `0`
and `1` must be implemented for console BASIC and Extended BASIC. External
groups must dispatch through the compatibility CPU-memory map so cartridge
ROM and DSR-provided routines can participate.

GROM0 also uses the scratchpad vector group dynamically. At GROM `>125E`, it
copies 64 bytes beginning at GROM `>1267` into TI scratchpad `>8300`; `XML >F0`
then enters the vector at `>8300`. The copied TMS9900 helper reverses bits in a
VDP range. The shim preserves the GROM0 source artifact unchanged and emits a
separate execution copy with six audited relocations for mapped GPL RAM,
native workspace, and Tutor VDP ports. PHM3005 Video Graphs now passes its
bounded smoke run through this original service path.

`XML >19` (`SROM`) and `XML >1A` (`SGROM`) are scan/link services. The
byte-verified console no-match path executes `RESET`, clearing the GPL
condition bit and continuing. The compact shim now preserves that result.
CS1 will later become a named GROM0 DSR match; CS2 will remain unavailable.

### Synchronous hardware services

| Entry/function | ROM address | Required adapter behavior |
| --- | ---: | --- |
| GPL `SCAN` / `KSCAN` | `>02B2` | TI keyboard modes, joystick/player selection, `KEYCOD`, `STATUS`, debounce and modifier semantics translated to the Tutor matrix |
| `FORMAT` | `>04DE` | Exact Graphics-I address, wrap, repeat and nested-block behavior |
| GPL `I/O` dispatcher | `>05C8` | Sound-list source selection, CRU input/output, optional cassette functions |
| Sound-list arm | `>05D6` | Set `SNDADD >83CC`, `STFLGS >83CE`, and GROM/VDP source state |
| XML dispatcher | `>0608` | Internal and external native vector dispatch |
| Multi-space `MOVE` | `>061E` onward | Exact CPU/GROM/VDP/register source and destination semantics |
| GROM program scan | `SGROM >0B24` | Scan console and cartridge GROM headers/program lists |
| Peripheral ROM/DSR scan | `SROM >0AC0` | CRU-select ROMs and follow power-up/DSR/subprogram link chains |

### Asynchronous console services

The VDP interrupt path starts at `REMOTE >0900`. Its compatibility-critical
portion is `TIMING >094A` through `SNDEXT >0ABA`:

1. acknowledge VDP interrupt status;
2. advance automatic sprite motion using `MOTION >837A` as the exact active
   sprite count (zero disables motion), VDP `>0780` motion records, and VDP
   `>0300` sprite attributes;
3. decrement `STFLGS` and service GROM- or VDP-sourced sound records from
   `SNDADD` (`TSTSND >09E8`);
4. preserve the sound-list branch rules: count `>00` changes address, count
   `>FF` changes address and source, and an ordinary record with zero duration
   stops scheduling;
5. update `VDPST >837B` and increment `TIME >8379` (`TIMEXT >0A84`);
6. invoke the optional user interrupt hook.

This complete ordering is required. Servicing TIME, sound, SCAN, and VDP
status as independent approximations can lose or duplicate a frame.

## What can initially be deferred

- Cassette `WRITE >1346`, `VERIFY >1426`, and `READ >142E` remain a separate
  planned service bank. CS1 is now an explicit compatibility target; CS2 must
  return bad-device-name. The TI record layer uses duplicated 64-byte blocks
  with leader, sync, count, and checksums; it will sit above the byte-verified
  Tutor `>ED00`/`>EE00–>EE60` transport instead of substituting CRU addresses.
  See `cassette-cs1-translation-plan.md`.
- Debug-board and unused console entries.
- Speech hardware until a selected cartridge requires it.
- Physical expansion-card implementations. Their XML/DSR dispatch paths must
  fail cleanly or route to a future adapter rather than corrupt state.

Standard cartridge ROM/GROM software, console BASIC, and Extended BASIC are
not deferrable targets. Special cartridges with their own processors, RAM,
GROM emulators, speech or unusual banking remain separate hardware profiles.

## ESE placement consequences

The W27C512 stores 64 KiB, but the verified selectors expose one 32 KiB half,
and `16` mode exposes the replacement BIOS in CPU `>0000->3FFF` so a front
cartridge can provide the complementary ROM/RAM windows. Therefore two useful
build profiles are distinct:

1. **32 KiB reference profile:** patched 8 KiB console ROM, 6 KiB GROM0 and
   the Pyuuta adapter/interpreter fit together. This is the quickest full-ROM
   behavioral oracle and may support self-contained payloads.
2. **16 KiB cartridge profile:** the compact adapter/interpreter plus authentic
   6 KiB GROM0 remains in the rear ESE window; cartridge ROM/GROM data and RAM
   live on the front board. Exact ROM algorithms are ported into the compact
   implementation.

Until runtime selection of the other W27C512 regions is electrically proven,
the 64 KiB capacity must not be treated as four simultaneously callable
16 KiB banks.

## Implementation strategy

1. Keep the pre-GROM0 16 KiB image and reports as the control.
2. Retain exactly 6 KiB of authentic GROM0 (`>0000->17FF`).
3. Use the exact ROM source to build conformance tests for every GPL opcode,
   addressing mode, XML group, and interrupt service.
4. Build a 32 KiB source-patched ROM reference profile.
5. Port only hardware-dependent entry points into Pyuuta adapters; preserve
   ROM algorithms and scratchpad side effects.
6. Qualify PHM3001 first, then console BASIC/Extended BASIC, then a cartridge
   matrix covering ROM-only, GROM-only, mixed ROM/GROM, CRU and DSR behavior.
