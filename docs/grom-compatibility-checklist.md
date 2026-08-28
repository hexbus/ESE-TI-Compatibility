# Tomy Tutor TI GROM compatibility checklist

This is the shared queue for testing the custom Tutor/TI compatibility BIOS.
It is intentionally stricter than a filename inventory: a normal candidate
must contain only GROM data, begin with a standard TI `>AA` header, fit in the
Tanam board's current four-GROM/24 KiB payload, and have no same-module CPU-ROM
component. Component identity is checked by content where the collection's
filename extension is known to be wrong.

"Pure GROM" describes cartridge storage only. It does **not** mean that the
module is self-contained: a cartridge can still call console GROM 1/2, require
32 KiB expansion RAM, or depend on cassette, disk, RS232, speech, or MBX
hardware. Those requirements are tracked separately below.

An emulator smoke pass means only that the compatibility title found the
module, selected its first program, entered cartridge GPL, and avoided the
visible shim fault loop for 400 runner slices. It is not a gameplay pass and
does not replace testing on the Tutor.

Status values:

- `RC baseline`: physically exercised enough to include in the release, with
  any remaining edge cases documented separately.
- `Physical smoke`: starts on the Tutor; extended paths remain unverified.
- `Emulator smoke pass`: bounded generic boot test passed.
- `Reproducible shim fault`: bounded generic boot test found a new defect.
- `Untested`: structurally convertible but not yet run.

## Automated 1–4 GROM audit snapshot

Latest complete run: **2026-08-27**, using 400 runner slices per module against
shim core SHA-256
`1089C6723C0CAFFBC8FCCB891579E3A3F10C35B64E5D7035B77574D709745A9D`.
This rerun includes the native two-port controller implementation. It is a
bounded boot/service test, not a claim that every gameplay path works.

| Result | Count | Meaning |
|---|---:|---|
| Emulator smoke pass | 47 | Module entered GPL and avoided the shim fault loop for the bounded run |
| Reproducible shim fault | 6 | Module exposed a compatibility defect or unsupported dependency |
| Generic-launch harness error | 6 | Image was built, but no bounded standard program node was found |
| **Total** | **59** | All structurally eligible current 1–4 GROM candidates |

Reproducible shim faults:

- PHM3040 TI LOGO — requires 32 KiB expansion RAM; its bounded run advances
  through the original scan/link helper, then faults later at GPL `>6063`,
  opcode `>BF00`
- PHM3045 SMU Electrical Engineering Library — generic entered the shim fault
  loop; cassette or disk is required and a possible 32 KiB expansion-RAM
  requirement remains unconfirmed because this is a prototype
- PHM3111 TI-Writer — generic entered the shim fault loop; cartridge scan table count=6 entry=>6028 name=>6015 length=19
- PHM3116 Demolition Division — a valid indexed `MOVE >32` reads data at
  console GROM 2 address `>4A67`, which is not presently mapped
- PHM3119 Meteor Multiplication — a valid indexed `MOVE >32` reads data at
  console GROM 1 address `>3500`, which is not presently mapped
- PHM3055 Editor/Assembler — generic entered the shim fault loop; full use is
  separately pending 32 KiB RAM and disk support

The grouped arithmetic and original GROM0 scan/link work converted PHM3005,
PHM3006, PHM3009, PHM3020, PHM3031, PHM3103, PHM3115, and PHM3118 from faults
to bounded passes in this run.

Generic-launch harness errors (all six are Scott, Foresman educational layouts):

- PHM3027 Addition and Subtraction 1
- PHM3028 Addition and Subtraction 2
- PHM3029 Multiplication 1
- PHM3049 Division 1
- PHM3050 Numeration 1
- PHM3051 Numeration 2

The authoritative per-module paths, hashes, and results are in
`build/grom-candidate-batch/batch-manifest.json`.

Two accidental mixed dumps are excluded from this project list and from all
59 candidate counts: PHM3155 I'm Hiding (MBX GROM+CPU-ROM) and PHM3219 Super
Demon Attack (three GROMs plus CPU ROM). They remain detectable in the archival
byte inventory only so a misleading filename cannot add them back later.

## Batch 1: next ten plus Diagnostic

Diagnostic is an additional case rather than one of the ten. It originated on
the TI-99/4; its complete phase-by-phase result must be compared with the
user's TI-99/4A js99er reference before it can be called compatible.

| Done | Role | PHM | Module | GROMs | Current status | Next verification |
|---|---|---:|---|---:|---|---|
| [ ] | Extra | 3000 | Diagnostic | 1 | Emulator smoke pass | Compare every phase with TI-99/4A oracle, then Tutor |
| [ ] | B1 | 3030 | A-Maze-Ing | 1 | Emulator smoke pass | Title, controls, collision, replay, FCTN-= |
| [ ] | B1 | 3031 | The Attack | 1 | Emulator state-machine pass; common DOR coordinate corruption fixed | Physical controls, scoring, replay, FCTN-= |
| [ ] | B1 | 3033 | Blackjack and Poker | 1 | Emulator smoke pass | Both games, numeric input, replay, FCTN-= |
| [ ] | B1 | 3034 | Hustle | 1 | Emulator smoke pass | Controls, scoring, replay, FCTN-= |
| [ ] | B1 | 3025 | Mind Challengers | 1 | Emulator smoke pass | All menu paths and keyboard input |
| [ ] | B1 | 3009 | Football | 2 | Emulator smoke pass | Physical options, interception, scoring, replay |
| [ ] | B1 | 3024 | Soccer | 2 | Emulator smoke pass | Options, controls, sprites, scoring, replay |
| [ ] | B1 | 3003 | Beginning Grammar | 3 | Emulator smoke pass | Lessons, keyboard, animation, replay |
| [ ] | B1 | 3020 | Music Maker | 3 | Emulator smoke pass | Physical keyboard, instruments, playback, replay |
| [ ] | B1 | 3008 | Video Chess | 4 | Emulator smoke pass | Input, legal moves, AI turn, replay, FCTN-= |

## Existing physical baselines

| Done | PHM | Module | GROMs | Current status | Notes |
|---|---:|---|---:|---|---|
| [x] | 3001 | Demonstration | Special 4-slot dump | RC baseline | Full demo/branches exercised; kept as a special packer case |
| [x] | 3023 | Hunt the Wumpus | 1 | RC baseline | Main gameplay works; rare replay/transition edge cases remain possible |
| [ ] | 3032 | Blasto | 1 | Physical smoke | Starts and reaches gameplay; extended option/game paths and pacing pending |

## Remaining structurally eligible candidates

These have burnable layouts. Every row received the bounded automated audit;
`Untested` below now means that no corresponding physical Tutor path has been
completed, not that the file was omitted from the emulator batch.

### One GROM

| Done | PHM | Module | Status / dependency note |
|---|---:|---|---|
| [ ] | 3004 | Number Magic | Emulator smoke pass; physical paths untested |
| [ ] | 3011 | Speech Editor | Emulator smoke pass; speech-hardware behavior must be scoped |
| [ ] | 3041 | Adventure | Emulator smoke pass; cartridge boots, but a cassette adventure is required to play |
| [ ] | 3054 | Car Wars | Emulator smoke pass; physical paths untested |
| [ ] | 3111 | TI-Writer | **Pending:** requires 32 KiB expansion RAM and disk |
| [ ] | 3055 | Editor/Assembler | **Pending:** requires 32 KiB expansion RAM and disk |
| [ ] | 3005 | Video-Graphs | Emulator smoke pass after original XML `>F0` scan/link helper support; physical paths untested |

### Two GROMs

| Done | PHM | Module | Status / dependency note |
|---|---:|---|---|
| [ ] | 3002 | Early Learning Fun | Emulator smoke pass; physical paths untested |
| [ ] | 3006 | Home Financial Decisions | Emulator smoke pass; standalone physical finance paths untested |
| [ ] | 3007 | Household Budget Management | Emulator smoke pass; physical paths untested |
| [ ] | 3010 | Physical Fitness | Emulator smoke pass; physical paths untested |
| [ ] | 3017 | Terminal Emulator I | Emulator smoke pass; full use pending RS232 support |
| [ ] | 3018 | Video Games 1 | Emulator smoke pass; physical paths untested |
| [ ] | 3019 | Disk Manager 1.0 | Emulator smoke pass; full use pending disk support |
| [ ] | 3045 | SMU Electrical Engineering Library | Reproducible shim fault; full use needs cassette or disk; possible 32 KiB expansion requirement is unconfirmed |
| [ ] | 3088 | Computer Math Games VI | Emulator smoke pass; physical paths untested |
| [ ] | 3089 | Disk Manager 2.0 | Emulator smoke pass; full use pending disk support |
| [ ] | 3091 | Milliken Subtraction | Emulator smoke pass; physical paths untested |
| [ ] | 3093 | Milliken Division | Emulator smoke pass; physical paths untested |
| [ ] | 3094 | Milliken Integers | Emulator smoke pass; physical paths untested |
| [ ] | 3098 | Milliken Number Readiness | Emulator smoke pass; physical paths untested |
| [ ] | 3100 | Milliken Equations | Emulator smoke pass; physical paths untested |
| [ ] | 3115 | Alien Addition | Emulator smoke pass after grouped arithmetic coverage; physical paths untested |
| [ ] | 3116 | Demolition Division | Valid indexed MOVE needs console GROM 2 data at `>4A67`; not a missing MOVE opcode |
| [ ] | 3118 | Minus Mission | Emulator smoke pass after grouped arithmetic coverage; physical paths untested |
| [ ] | 3119 | Meteor Multiplication | Valid indexed MOVE needs console GROM 1 data at `>3500`; not a missing MOVE opcode |

### Three GROMs

| Done | PHM | Module | Status / dependency note |
|---|---:|---|---|
| [ ] | 3027 | Addition and Subtraction 1 | Scott Foresman autoboot; generic launcher does not yet model it |
| [ ] | 3028 | Addition and Subtraction 2 | Scott Foresman autoboot; generic launcher does not yet model it |
| [ ] | 3029 | Multiplication 1 | Scott Foresman autoboot; generic launcher does not yet model it |
| [ ] | 3051 | Numeration 2 | Scott Foresman autoboot; generic launcher does not yet model it |
| [ ] | 3064 | Touch Typing Tutor | Emulator smoke pass; broad physical keyboard exercise pending |
| [ ] | 3083 | Computer Math Games II | Emulator smoke pass; physical paths untested |
| [ ] | 3090 | Milliken Addition | Emulator smoke pass; physical paths untested |
| [ ] | 3092 | Milliken Multiplication | Emulator smoke pass; physical paths untested |
| [ ] | 3096 | Milliken Decimals | Emulator smoke pass; physical paths untested |
| [ ] | 3099 | Milliken Laws of Arithmetic | Emulator smoke pass; physical paths untested |
| [ ] | 3103 | Milliken Number Readiness & Addition | Emulator smoke pass; valid combined image using the shared Milliken base plus Number Readiness and Addition GROMs |

### Four GROMs

| Done | PHM | Module | Status / dependency note |
|---|---:|---|---|
| [ ] | 3012 | Securities Analysis | Emulator smoke pass; console GROM 1/2 dependency audit pending |
| [ ] | 3013 | Personal Record Keeping | Emulator smoke pass; likely BASIC/console GROM 1/2 dependency audit pending |
| [ ] | 3022 | Personal Real Estate | Emulator smoke pass; console GROM 1/2 dependency audit pending |
| [ ] | 3040 | TI LOGO | **Pending:** requires 32 KiB expansion RAM; later bounded fault at GPL `>6063` after scan/link succeeds |
| [ ] | 3049 | Division 1 | Scott Foresman autoboot; generic launcher does not yet model it |
| [ ] | 3050 | Numeration 1 | Scott Foresman autoboot; generic launcher does not yet model it |
| [ ] | 3085 | Computer Math Games III | Emulator smoke pass; physical paths untested |
| [ ] | 3086 | Computer Math Games IV | Emulator smoke pass; physical paths untested |
| [ ] | 3095 | Milliken Fractional Numbers | Emulator smoke pass; physical paths untested |

## Possible candidates requiring layout review

These fit the byte capacity but are not automatically burnable yet.

| PHM | Module | Why held |
|---:|---|---|
| 3038 | Connect Four | Two-slot file has its standard `>AA` header in the second slot; device order must be established |
| 3044 | Personal Report Generator | Two-slot file does not begin with a standard header; layout must be established |

PHM3001 Demonstration is not in this hold queue: it is already checked as an
RC baseline above. Its 33,024-byte collection artifact continues to use the
already-proven special packing path.

## Outside the current four-GROM payload

Pure-GROM modules requiring five or more devices remain valid future targets,
but need a larger/banked cartridge mapping before physical testing. Modules
with `.C`, `.D`, or raw `.bin` companions are tracked separately as mixed
GROM+CPU-ROM work and are not silently classified as GROM-only.

### Five-GROM images (currently out of scope)

These occupy five logical 8 KiB GROM slots. Although each physical GROM drives
6 KiB, the current Tanam conversion reserves only the first four slots, so none
of these can be represented completely by the present 24 KiB payload.

| PHM | Module |
|---:|---|
| 3014 | Statistics |
| 3015 | Early Reading |
| 3036 | Zero Zap |
| 3037 | Hangman |
| 3039 | Yahtzee |
| 3043 | Reading Fun |
| 3046 | Reading On |
| 3047 | Reading Roundup |
| 3048 | Reading Rally |
| 3059 | Scholastic Spelling — Level 3 |
| 3060 | Scholastic Spelling — Level 4 |
| 3061 | Scholastic Spelling — Level 5 |
| 3062 | Scholastic Spelling — Level 6 |
| 3082 | Reading Flight |
| 3084 | Computer Math Games I |
| 3097 | Milliken Math Sequences — Percents |
| 3113 | Microsoft Multiplan v1.04 |
| 3114 | Alligator Mix |
| — | Tunnels of Doom |

The archive also contains a five-slot **Milton Bradley Gamevision prototype
aggregate**. It is not one normal five-GROM module: Connect Four, Hangman, and
the other constituent games use their own logical GROM bases and must be split
and mapped individually before testing. Word Invasion (PHM3169) occupies seven
GROM slots and is likewise outside the present mapping.

## Known external hardware and mixed-module holds

| PHM | Module | Why full compatibility is pending |
|---:|---|---|
| 3041 | Adventure | Cartridge boots; cassette media supplies the selected adventure |
| 3040 | TI LOGO | Requires 32 KiB expansion RAM |
| 3055 | Editor/Assembler | Requires 32 KiB expansion RAM and disk for normal use |
| 3111 | TI-Writer | Requires 32 KiB expansion RAM and disk |
| 3017 | Terminal Emulator I | Requires RS232 for terminal communications |
| 3035 | Terminal Emulator II | Mixed GROM+ROM module; requires RS232, with optional speech support |
| 3019 | Disk Manager 1 | Requires a disk system |
| 3089 | Disk Manager 2 | Requires a disk system |
| 3045 | SMU Electrical Engineering Library | Uses cassette or disk storage; prototype may also require 32 KiB expansion RAM, not yet confirmed |

## Console GROM 1/2 dependency audit

The emulator harness records both GPL instructions dispatched from console
GROM and data bytes read from console GROM. The batch manifest reports
`observed_console_grom_pcs`, `observed_console_grom_read_ranges`, and their
union in `observed_console_groms`. Tracking only the program counter was not
enough: GPL can execute in a cartridge while an indexed `MOVE` fetches fonts,
tables, or other data from a console GROM.

The 2026-08-27 bounded run found exactly two data-only dependencies:

- PHM3119 Meteor Multiplication reads console GROM 1 at `>3500`.
- PHM3116 Demolition Division reads console GROM 2 at `>4A67`.

Neither module entered console GROM 1/2 as executable GPL; their apparent
`MOVE >32` faults were the missing source data being interpreted after the
read failed. No other candidate accessed console GROM 1/2 in the bounded boot
path, including the three business candidates. That does not prove their
deeper application paths are self-contained, so Securities Analysis, Personal
Record Keeping, and Personal Real Estate remain flagged for workflow tests.

## Repeatable commands

Refresh the byte-level inventory:

```powershell
python tools/inventory_grom_candidates.py "TI cart files" `
  --json build/grom-candidate-inventory.json --candidates
```

Build and smoke-test Batch 1 plus Diagnostic:

```powershell
python tools/build_grom_candidate_batch.py --set top --slices 400
```

Build every structurally eligible image without running it:

```powershell
python tools/build_grom_candidate_batch.py --set all --no-smoke
```

The generated files live under `build/grom-candidate-batch`. A PASS there is
an emulator boot result only; check the corresponding box only after the
documented paths have been exercised on real hardware.
