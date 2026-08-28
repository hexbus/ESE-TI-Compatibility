<!-- SPDX-FileCopyrightText: 2026 hexbus -->
<!-- SPDX-License-Identifier: CC-BY-4.0 -->

# TBR: Relocatable compiled-GPL and unified runtime architecture

## Status

**The first relocatable binding slice is implemented experimentally; the
general compiler/optimizer remains back-burner research.**

This records a future direction suggested by the PHM3001 compatibility shim
and the PHM3052 Tombstone City source port. The immediate work remains
interpreter correctness and cartridge qualification. RC6 includes a matched
PHM3052 hardware candidate proving the smallest useful part: stock GPL
transfers through `XML >70` to relocated native code, and that code resolves
a common keyboard/frame service through a fixed core vector. The general
compiler and optimizer described here do not yet exist as a production tool.

## Implemented proof, 2026-08-28

The experiment establishes these concrete contracts:

- The stock Tombstone GROM remains byte-identical and reaches `XML >70` at
  logical `>6065`.
- `TTBI 1.0` lives in the fixed core footer at `>3FFC`; vector 1 at `>3FFE`
  provides frame processing plus common TI-99/4A KSCAN/joystick translation.
- `TTEX 1` describes a native extension in the Tanam high-ROM window and a
  biased entry pointer at `>BFFE` safely distinguishes an erased pure-GROM
  cartridge from a provider.
- The recovered native source is transformed in generated copies only:
  scratchpad `>83xx` to SRAM `>60xx`, TI VDP ports to Tutor ports, one console
  KSCAN call to TTBI vector 1, and six interrupt enables to the polled-frame
  contract.
- The generated code loads at `>8000`, enters at `>805C`, and is packed beside
  the original GROM into the proven 24 KiB logical/parallel Tanam layout.
- Structural validation proves every address, hash, checksum, and parallel
  window. JS99er proves the level menu, physical Tutor key `1`, novice state,
  and real gameplay after more than one million native instructions, with no
  unknown bus writes or shim fault.

This is relocation and service binding—not yet GPL-to-native compilation.
It nevertheless answers the architectural question: a module can name and
call stable core services without embedding the core's moving labels, while
its native body can occupy otherwise unused module ROM.

## Objective

Create one semantic compatibility runtime for TI-99 and Tutor/Pyuuta targets:

```text
TI cartridge/GROM source
        |
        v
decoded GPL + symbolic service calls + module metadata
        |
        +--> unchanged GPL fallback
        |
        +--> verified native TMS9900-family translations of hot blocks
        |
        v
platform link profile --> TI runtime or Tutor/TMS9995 runtime
```

The important property is that translated modules name services by contract,
not by the original console ROM address or by a Tutor BIOS address. A link or
load step resolves those names to the implementation available on the target.
The same translated module representation can therefore be placed differently
in memory without another cartridge-specific rewrite.

This is more than replacing one absolute address with another. GPL contains
code, data, indirect pointers, status-dependent behavior, and calls into both
console GROM and CPU ROM. The remapper must understand instructions and call
contracts; blind byte-pattern patching would eventually corrupt data or miss
dynamic calls.

## Proposed layers

### 1. Semantic service registry

Assign stable identifiers and explicit contracts to console facilities such
as:

- GROM reads and address management;
- VDP byte, block, fill, FORMAT, sprite, and motion operations;
- sound-list scheduling and PSG shutdown;
- KSCAN, joystick, timer, and interrupt behavior;
- arithmetic, FAC/XML, string, and display helpers;
- CS1, DSR, and optional extension services.

Each entry records inputs, outputs, scratchpad use, GPL status changes, VDP or
PSG side effects, timing requirements, reentrancy, and failure behavior. The
original console GROMs and ROM remain the behavioral oracle, not the runtime
layout contract.

### 2. Discoverable runtime ABI

A small fixed descriptor identifies the ABI version and a table of service
vectors. The implementations may live in fixed ESE ROM, banked ROM, cartridge
extension ROM, or native TI ROM. Callers locate the descriptor by its defined
signature/profile and dispatch through service identifiers rather than
embedding implementation addresses.

The first allocation is deliberately minimal: TTBI major/count bytes at
`>3FFC` and one 16-bit vector at `>3FFE`. `TTEX 1` reserves `>BF00–>BFFF` in
the unbanked module extension window. Additional services require a larger
versioned table or a banked provider; they must not silently repurpose this
frozen first vector.

### 3. GPL decoder and relocatable intermediate representation

The toolchain must parse full GPL addressing modes, operands, branches, data
regions, GROM-base transitions, XML calls, and known console entry points. It
emits:

- symbolic labels and relocations;
- logical GROM-to-physical storage mappings;
- service references instead of fixed firmware addresses where proven;
- preserved opaque data for anything not safely classified;
- a module manifest containing source hashes, ABI requirements, and target
  memory constraints.

This representation becomes the common input to both TI and Tutor builds.

### 4. Tiered execution

Correctness comes before native conversion:

1. Run unknown GPL through the compatible interpreter unchanged.
2. Redirect proven console-service calls through the runtime ABI.
3. Translate verified hot GPL basic blocks to position-independent native
   TMS9900/TMS9995 code.
4. Keep a guarded fallback whenever indirect control flow or state cannot be
   proven.

Likely native wins include GROM fetch/dispatch removal, bulk VDP moves and
fills, sprite motion, sound lists, arithmetic, and frequently executed game
loops. VDP access pacing, interrupt boundaries, keyboard state, GPL status,
workspace rules, and observable timing must still match the required hardware
contract; faster is not correct if it races the VDP or changes game state.

## Tombstone City's second role

The GPL port remains a useful standalone proof, but it is also the first data
set for this future architecture. While mapping the mixed GROM/CPU-ROM module,
retain information that a later compiler and linker will need:

- every GROM-to-native and native-to-GROM boundary;
- the native calling convention, workspace, return path, and clobbers;
- console ROM/GROM services reached directly or indirectly;
- shared scratchpad, VDP, sprite, sound, keyboard, and timer state;
- hot loops and approximate cycle costs;
- which routines are pure game logic versus hardware services;
- stock-oracle checkpoints before and after each boundary.

That avoids throwing away the knowledge gained during the all-GPL port. It
also lets Tombstone City later compare three executions of the same behavior:
the stock mixed cartridge, the portable GPL port, and compiled native blocks
using the unified ABI.

## Validation model

Every optimization needs differential tests against the stock TI oracle and
the interpreter path. At defined checkpoints compare GPL PC, workspace and
scratchpad bytes, status flags, relevant VRAM and sprite tables, PSG command
streams, input state, score/lives, and major state transitions. Native blocks
are enabled only for an exact recognized source/ABI version and fall back on
any failed guard.

Success means that one translated module package can target the TI and Tutor
through different runtime profiles, with no hand-patched service addresses,
while producing equivalent externally visible behavior.

## Risks to resolve before implementation

- Mixed code/data and computed GROM addresses can defeat static discovery.
- XML may enter arbitrary native cartridge code, including code whose state
  contracts are not documented.
- Console services often communicate through GPL status and shared scratchpad
  rather than ordinary function arguments.
- Interrupt, sound, keyboard, random-number, and VDP timing can be observable
  game state rather than incidental delay.
- A fixed ESE has limited code space; useful implementations may require a
  banked service ROM or a cartridge extension.
- Original TI firmware redistribution must remain separate from the tooling;
  user-supplied dumps, hashes, manifests, and transformation patches are the
  safer release model.

## Updated implementation sequence

1. **Done:** map Tombstone's first GPL/native boundary and prove a relocatable
   native module entry plus one stable core service.
2. Continue GPL interpreter and service-conformance work across qualified
   cartridges; the mixed proof must not hide missing GPL behavior.
3. Complete the separately scoped faithful all-GPL Tombstone proof.
4. Derive a broader versioned semantic service registry from multiple cartridges,
   GROM0, console ROM, and trace evidence.
5. Build the relocatable GPL decoder/IR and link profiles without changing
   execution.
6. Redirect a returning leaf service and prove complete state equivalence;
   Tombstone's current main entry is intentionally non-returning.
7. Translate one measured hot GPL block to native code and quantify the gain.
8. Consider translated console GROM1/GROM2, BASIC/XB, and broader cartridge
   remapping only after the contracts are stable.

This sequence preserves the current working BIOS while accumulating exactly
the evidence needed to decide whether a generalized optimizer is worthwhile.
