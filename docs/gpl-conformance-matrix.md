# GPL conformance matrix

This matrix compares the compact Pyuuta interpreter with the exact dispatch
tables in the byte-verified TI console ROM. `Implemented` means the family is
present and covered by the PHM3001 suite. `Partial` means useful forms exist
but the full ROM-defined family, operand spaces, or side effects are not yet
proven. `Missing` means the compact dispatcher currently reaches its visible
unsupported fault.

## Miscellaneous and control opcodes

| Opcodes | TI function | Current state | Next conformance requirement |
| --- | --- | --- | --- |
| `00` | `RTN` | Implemented | Add nested/deep-stack vectors |
| `01` | `RTNC` | Implemented | Add nested/deep-stack and both-condition vectors |
| `02` | `RAND` | Implemented | Exhaustive modulus/seed vectors |
| `03` | `SCAN` | Partial | Modified-Tutor PC-style matrix, Shift, MOD/FCTN, MON/CTRL, new-press debounce, both Tutor controller ports, exact GROM0 axis translation, and fire-as-`KEYCOD >12` are covered. Remaining work is broader stock-Tutor keyboard-profile and unusual scan-mode conformance. |
| `04` | `BACK` | Implemented | VDP-register side-effect vector |
| `05` | long `B` | Implemented | Cross-GROM/device vectors |
| `06` | `CALL` | Implemented | Cross-GROM and stack-overflow vectors |
| `07` | `ALL` | Implemented | VDP wrap/address vectors |
| `08` | `FMT` | Partial | All nested block, direction, skip, repeat and special forms |
| `0A` | `GT` status test | Implemented | Add independent true/false status vectors |
| `09,0C,0D` | remaining status-bit tests | Missing | Exact `STATUS` bit selection/reset rules |
| `0B` | interpreter `ENTRY` | Missing | Restart/vector behavior |
| `0E` | console `PARSE` | Missing | Original console BASIC entry |
| `0F` | `XML` | Partial | TI radix-100 `ROUNU`, `STST`, `FADD`, `FSUB`, `FMULT`, `FDIV`, `FCOMP`, `SADD`, `SSUB`, `SMULT`, `SDIV`, `SCOMP`, `CSN`, and `CFI` are implemented from the byte-verified ROM ABI. `XML >19/>1A` now preserve the console SROM/SGROM no-match result, and the GROM0 `XML >F0` scratchpad helper is byte-audited and relocated for Tutor RAM/VDP addresses. Complete the remaining internal/external native-vector dispatcher. |
| `10,11,12` | console `CONT`, `EXEC`, `RTNB` | Missing | Original console BASIC entries |
| `13` | `RGBA` | Missing | ROM-defined behavior |

## MOVE and branch families

| Opcodes | TI function | Current state | Gap |
| --- | --- | --- | --- |
| `20–3F` | `MOVE` | Partial | CPU and VDP destinations plus PHM3001's VDP-register destination are implemented. The Tutor BIOS packed control-pair cadence plus VBlank R1 reassertion passed physically: full 16x16 rooks, bus, football and players now render. PHM3116/PHM3119 prove that valid indexed moves can also require data from console GROM 2/1 respectively; those are missing data mappings, not missing MOVE opcodes. Complete GROM destinations, remaining source/destination pairings, and writable-GROM policy remain. |
| `40–5F` | branch if condition reset | Implemented | Exhaustive slot/address vectors |
| `60–7F` | branch if condition set | Implemented | Exhaustive slot/address vectors |

## Single-operand families

| Opcodes | TI function | Current state |
| --- | --- | --- |
| `80,81` | `ABS/DABS` | Implemented |
| `82,83` | `NEG/DNEG` | Implemented |
| `84,85` | `INV/DINV` | Implemented |
| `86,87` | `CLR/DCLR` | Implemented through the common CPU/VDP unary operand path; screen-pass Y-motion clears remain regression-tested |
| `88,89` | `FETCH/DFETCH` | Implemented |
| `8A,8B` | `CASE/DCASE` | Implemented |
| `8C,8D` | `PUSH/DPUSH` | Missing |
| `8E,8F` | `CZ/DCZ` | Implemented |
| `90–97` | `INC/DINC/DEC/DDEC/INCT/DINCT/DECT/DDECT` | Implemented |
| `98–9F` | reserved slow/no-op slots | Not required until a real image depends on their exact timing |

## Two-operand families

The low two opcode bits select byte/word and variable/immediate forms.

| Opcodes | TI family | Current state | Implemented subset |
| --- | --- | --- | --- |
| `A0–A3` | `ADD/DADD` | Implemented for PHM forms | Variable/immediate byte and word forms; complete exhaustive address/status vectors |
| `A4–A7` | `SUB/DSUB` | Implemented for PHM forms | Variable/immediate byte and word forms; complete exhaustive address/status vectors |
| `A8–AB` | `MUL/DMUL` | Implemented for PHM forms | Add zero/overflow and all address-space vectors |
| `AC–AF` | `DIV/DDIV` | Partial | Immediate-byte `AE` and immediate-word `AF`, including adjacent quotient/remainder output; variable forms `AC/AD` remain |
| `B0–B3` | `AND/DAND` | Implemented for PHM forms | Variable/immediate byte and word forms; complete exhaustive address/status vectors |
| `B4–B7` | `OR/DOR` | Implemented for PHM forms | PHM3031 proves immediate-word `DOR >0101,@YPT` without `VGREAD` register clobber; complete exhaustive CPU/VDP variable/immediate address/status vectors |
| `B8–BB` | `XOR/DXOR` | Implemented for PHM forms | Variable/immediate byte and word forms; complete exhaustive address/status vectors |
| `BC–BF` | `ST/DST` | Implemented | Complete side-effect vectors for pseudo-variables and VDP |
| `C0` | `EX` byte exchange | Implemented | Exhaustive CPU/VDP and aliasing vectors; Wumpus `>7167` permanently covers CPU scratchpad exchange and status preservation |
| `C1` | `DEX` word exchange | Missing | Implement CPU/VDP word exchange with the original no-`FILLST` status behavior |
| `C2–C3` | undefined/reserved slots | Trapped | Retain a visible fault until an authoritative image proves another interpretation |
| `C4–D7` | signed/unsigned/equality comparisons | Implemented for PHM forms | Exhaustive byte/word, variable/immediate and status vectors |
| `D8–DB` | logical `IFAND` (`CLOG/DCLOG`) | Implemented for PHM forms | Exhaustive CPU/VDP, byte/word and variable/immediate vectors |
| `DC–DF` | `SRA/DSRA` | Partial | Immediate-word `DF`; byte and variable forms missing |
| `E0–E3` | `SLL/DSLL` | Partial | Immediate byte/word `E2/E3`; variable forms missing |
| `E4–E7` | `SRL/DSRL` | Partial | Immediate-word `E7`; byte and variable forms missing |
| `E8–EB` | `SRC/DSRC` | Partial | Immediate `EA/EB`; variable `E8/E9` missing |
| `EC–EF` | `COINC` | Missing | Exact table and condition behavior |
| `F0–F3` | reserved slow/no-op slots | Deferred | Match only if required |
| `F4–F7` | `I/O` | Partial | Immediate GROM/VDP sound-list arm; CRU and cassette forms missing. Cassette selectors `4/5/6` are separately specified as write/read/verify of a four-byte length-plus-VDP-address list; see `cassette-cs1-translation-plan.md`. |
| `F8–FB` | `DGBA` library call | Missing | Cross-GROM library entry and return behavior |
| `FC–FF` | reserved slow/no-op slots | Deferred | Match only if required |

## Service-level blockers for broad cartridge support

1. `XML` is the largest semantic blocker. Internal groups 0 and 1 expose the
   floating-point, conversion, scan/link, and BASIC services; external groups
   dispatch into cartridge ROM, peripheral ROM, expansion code, or scratchpad.
2. Operand decoding must be verified independently of opcode arithmetic.
   Every CPU, VDP, GROM, indirect, indexed, immediate, byte, and word form gets
   a small vector with expected result and exact `STATUS` byte.
   PHM3001's finance path demonstrated why family-level inventory is not
   enough: `ADD` was listed as present, but its variable-byte form loaded the
   decoded source address instead of the byte stored there. Only that operand
   form, with a source address whose low byte differed from its contents,
   exposed the defect.
3. VBlank/TIME/sound/sprite service ordering is already implemented and
   PHM3001-tested, but it still needs byte-for-byte state vectors against
   `REMOTE/TIMING/TSTSND` from the original ROM.
4. Console GROM1/2 must be mapped not only for BASIC execution but for
   cartridge data reads. PHM3119 reads GROM 1 at `>3500`, and PHM3116 reads
   GROM 2 at `>4A67`, without dispatching executable GPL there.
5. Extended BASIC additionally needs its 24 KiB GROM payload, two 8 KiB CPU
   ROM pages, expansion RAM policy, and a real runtime bank mechanism.
6. Console GROM0 can construct native code dynamically. Video Graphs exposed
   this through `XML >F0`: GROM0 copied a 64-byte TMS9900 bit-reversal helper
   to TI scratchpad `>8300` and executed its vector there. Static GPL opcode
   inventory therefore cannot prove completeness by itself; copied-code and
   external-vector destinations must also be traced.

## Efficient implementation order

1. Build a tiny table-driven conformance cartridge that runs one opcode and
   addressing-mode vector at a time and records result/status/memory effects.
2. Use the exact original ROM interpreter as the oracle in the host harness.
3. Complete arithmetic/logical/shift/unary families and address decoding.
4. Implement `XML` group 0, then group 1, then external vector dispatch.
5. Add console GROM1/2 and run console BASIC smoke tests.
6. Add qualified banking and the Extended BASIC RPK profile only after the
   stock console-service vectors pass.
