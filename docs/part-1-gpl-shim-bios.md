# Part 1: Self-testing GPL shim BIOS

## Decision

The first executable compatibility target will not be Extended BASIC. It will be a replacement Pyuuta BIOS that can test its own native hardware layer and then run one ordinary TI cartridge GROM through a small GPL/GROM compatibility layer.

The recommended first TI payload is PHM 3001 Demonstration. PHM 3000 Diagnostic follows after the interpreter and console-call shims are stable. Extended BASIC is the later integration target.

## Important address terminology

`>9800` is the TI-99/4A CPU address of the GROM data-read port. It is not a GROM number.

The related TI ports are normally:

| CPU address | Function |
|---|---|
| `>9800` | GROM data read |
| `>9802` | GROM address read |
| `>9C00` | GROM data write |
| `>9C02` | GROM address write |

The logical GROM address is separate. Console GROMs 0 through 2 occupy the lower logical GROM space; a conventional cartridge GROM commonly begins at logical `>6000` (GROM 3). GROM 4 would begin at logical `>8000`. The actual PHM 3000/3001 dump and header must determine the payload's logical origin; it must not be inferred from `>9800` or from a filename.

On the Pyuuta, `>9800` does not need to become a new physical memory-mapped port. GPL bytecode is serviced by the shim's GPL interpreter, which implements the TI GROM-port behavior in software while reading bytes from EPROM. Native TMS9900 code that directly accesses TI address `>9800` would require patching or a compatibility call.

## Hardware allocation

The initial build assumes:

- ESE Game Adapter on an original Pyuuta, with `/KILL SROM` asserted to replace the internal system ROM;
- ESE W27C512 for reset vectors, native Pyuuta HAL, POST, GPL interpreter, and compatibility services;
- Door Door/RAM-and-ROM cartridge for the TI payload and working RAM;
- a conservative first cartridge layout of 32 KiB EPROM plus 8 KiB SRAM;
- Pyuuta MkII/Tutor implementations may omit the rear adapter where their cartridge connector already supplies the required signals.

The W27C512 is physically 64 KiB, but usable capacity is not the same as simultaneously visible CPU address space. The GAL equations and selected address windows determine what is visible. Every bank and window therefore must be tested before relying on the nominal chip sizes.

## Two-layer design

### Layer 1: native Pyuuta BIOS and POST

This layer must boot and diagnose the machine without any TI GROM payload. It owns:

- TMS9995 reset vectors and workspaces;
- RAM initialization;
- TMS9918 initialization and VBlank interrupt handling;
- SN76489A initialization;
- Pyuuta keyboard/controller multiplexing and CRU input;
- cartridge ROM banking and SRAM tests;
- cassette primitives where practical;
- a visible diagnostic UI and stable error codes.

### Layer 2: TI GPL compatibility

This layer supplies:

- a GPL interpreter;
- a virtual GROM address counter and read-ahead behavior;
- virtual GROM data/address read services;
- one cartridge GROM image at its verified logical origin;
- a small TI-console compatibility ABI used by the interpreter and payload;
- explicit `not available` results for DSR and file operations not yet implemented.

The first milestone supported one virtual cartridge GROM and one GROM base.
The current implementation has advanced beyond that milestone: it scans the
standard cartridge bases `>6000`, `>8000`, `>A000`, and `>C000`, validates
headers and bounded linked program records, and constructs its launcher table
from their names and entry addresses. TI DSR scanning, disks, and Extended
BASIC remain outside Part 1.

## Boot sequence

1. Enter through replacement reset vectors in the ESE ROM.
2. Establish a known TMS9995 workspace and stack convention.
3. Initialize native RAM, VDP, PSG, interrupts, keyboard/controller input, and bank hardware.
4. Run native POST and display its result without depending on GPL.
5. Test the virtual GROM address/read implementation against known byte patterns.
6. Run a tiny synthetic GPL program that draws text, reads a key, changes the display, and writes sound.
7. If all required tests pass, locate and validate the PHM payload header.
8. Enter PHM 3001 through the GPL interpreter.
9. After PHM 3001 is stable, run PHM 3000 and classify its results.

Holding a designated key at reset should remain in the native diagnostic menu rather than launching the TI payload.

## Fixed shim ABI

The BIOS should expose a versioned vector table at a fixed replacement-ROM address. The exact addresses will be chosen after the ESE GAL decode is confirmed.

Minimum services:

| Service | Purpose |
|---|---|
| `COLD_INIT` | Initialize all native hardware and RAM |
| `WARM_INIT` | Re-enter without destructive RAM tests |
| `VDP_READ` / `VDP_WRITE` | Transfer VRAM bytes/blocks |
| `VDP_REG` | Write a VDP register |
| `WAIT_VBL` | Wait for the next VBlank tick |
| `SOUND_WRITE` | Send a byte or sequence to the PSG |
| `KSCAN` | Return normalized key/controller state |
| `BANK_SELECT` | Select a tested ROM/SRAM bank |
| `GROM_SET_ADDR` | Set the virtual GROM logical address |
| `GROM_READ_BYTE` | Read with TI-compatible counter behavior |
| `GPL_ENTER` | Start the interpreter at a logical GPL address |
| `DSR_UNAVAILABLE` | Return a deterministic unsupported-device error |
| `CMT_READ` / `CMT_WRITE` | Cassette primitives when implemented |

Each vector must preserve a documented register contract so later XB work can depend on the same ABI.

## Native POST matrix

| Test | Method | Required for PHM 3001 |
|---|---|---|
| Reset/workspace | Verify vector entry and workspace RAM | Yes |
| CPU RAM | Walking bits and address-pattern tests in safe regions | Yes |
| VDP registers | Initialize known mode and read status | Yes |
| VRAM | Address/data pattern tests plus visible color bars | Yes |
| VBlank interrupt | Count stable interrupt ticks | Yes |
| Sprites | Visible sprite pattern and collision/status exercise | Desirable |
| PSG | Audible channel sweep/noise test | Yes for full demonstration |
| Keyboard | Matrix/CRU test with on-screen key codes | Yes |
| Controllers | Show both controller states separately | Desirable |
| ROM banks | Signature and alias tests for every selected bank | Yes |
| SRAM | Non-destructive quick test, then destructive full test on request | Yes |
| Internal-ROM kill | Confirm replacement code is executing and internal ROM is absent | Yes on original Pyuuta |
| Cassette | Tone generation and optional loopback/decode test | No |

POST must report `PASS`, `FAIL`, or `NOT TESTED`; it must not silently continue after a required failure.

## GPL/GROM validation

Before a real cartridge image is entered, the shim must verify:

1. address writes select the intended logical byte;
2. sequential reads increment correctly;
3. address reads return the expected counter state;
4. crossing a GROM boundary selects the intended EPROM bank or produces a defined error;
5. reads outside the installed virtual GROM return a stable value rather than unrelated memory;
6. cartridge headers and entry pointers are range-checked;
7. every implemented GPL opcode has a small unit/smoke test;
8. unimplemented opcodes stop with an on-screen opcode and GPL address.

## Proof targets and success criteria

### Stage A: synthetic GPL smoke payload

Success means that GPL can:

- initialize through the shim;
- write a title into VRAM;
- wait for VBlank;
- read a keyboard choice;
- write one PSG tone;
- branch and return without an illegal-opcode trap.

### Stage B: PHM 3001 Demonstration

This is the preferred first unmodified TI payload. Success means that its header is recognized, its menu/title appears, input works, and it progresses through its demonstration without interpreter traps. Missing storage functions may return explicit errors.

### Stage C: PHM 3000 Diagnostic

PHM 3000 is a compatibility stress test, not the authority on whether a Pyuuta is healthy. TI-specific console ROM checksums, scratchpad assumptions, keyboard hardware, or absent peripherals may legitimately differ.

Success means:

- the diagnostic launches through the virtual GROM layer;
- it executes without illegal GPL opcodes or corrupting the shim;
- native-equivalent tests can pass;
- TI-only tests are reported as `N/A` or `EXPECTED DIFFERENCE` where the shim can intercept them;
- failures can be correlated with the native POST results.

### Stage D: Extended BASIC

Only after Stages A through C are repeatable should the project add multiple GROMs, the XB ROM portion, larger compatibility tables, and CS1/file services.

## Capacity estimate

A first implementation should fit comfortably in the proposed hardware:

- replacement BIOS, HAL, POST, and GPL interpreter: approximately 16–24 KiB initially;
- one cartridge GROM payload: approximately 6–8 KiB, pending the actual dump;
- diagnostic strings/tables: several KiB;
- 8 KiB SRAM: adequate for interpreter state, workspaces, scratch areas, and a modest GPL runtime allocation.

The limiting risk is address decoding and bank behavior, not total raw EPROM capacity.

## Required source files and measurements

Before building the first PHM image, obtain:

- exact PHM 3001 and PHM 3000 dumps;
- byte length and SHA-256 for each dump;
- confirmation of each image's logical GROM origin and header pointers;
- decoded ESE GAL equations or a measured bank/window truth table;
- decoded Door Door GAL equations for the selected 32 KiB ROM + 8 KiB SRAM configuration;
- confirmation of which ESE EPROM region replaces system ROM and which region remains externally selectable;
- a known-safe SRAM window and write-enable behavior;
- the desired reset key for entering POST versus launching the payload.

## First build deliverables

1. ESE replacement-ROM binary containing native initialization and POST.
2. A matching symbol/map file and fixed ABI document.
3. A Door Door cartridge image containing bank signatures, SRAM test support, and the synthetic GPL payload.
4. A host-side validator that checks ROM sizes, banks, reset vectors, GPL headers, and hashes before programming EPROMs.
5. A diagnostic result sheet recording machine model, adapter version, jumper settings, ROM hashes, and every POST result.
