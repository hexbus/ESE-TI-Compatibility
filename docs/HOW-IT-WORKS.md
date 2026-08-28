<!-- SPDX-FileCopyrightText: 2026 hexbus -->
<!-- SPDX-License-Identifier: CC-BY-4.0 -->

# How the compatibility system works

## The short version

The ESE replaces the Tomy system BIOS with a compatibility BIOS. A second
cartridge supplies converted TI module data and RAM. The BIOS interprets the
module's GPL program and translates its TI console requests to the Tutor's
real hardware.

This works well because the machines are related without being identical:

| Function | TI-99/4A | Tomy Tutor/Pyuuta | Compatibility action |
| --- | --- | --- | --- |
| CPU | TMS9900 | TMS9995 | Interpret GPL; relocate suitable native TMS9900-family code |
| Video | TMS9918A family | TMS9918 family | Reuse VDP data and translate CPU ports/timing |
| Sound | TI/SN PSG family | SN76489 family | Reuse sound streams and translate scheduling/ports |
| Firmware | Console ROM plus GROM 0–2 | Tomy system ROM | Supply the observable TI service contracts in the ESE BIOS |
| Input | TI keyboard/joystick CRU scan | Tutor keyboard/controller multiplexer | Translate to TI-99/4A KSCAN codes and joystick axes |
| Module | Serial GROM and optional CPU ROM | Parallel cartridge ROM | Pack GROM into ROM and translate logical reads in software |

## Power-up sequence

1. The TMS9995 resets into the ESE replacement BIOS.
2. The BIOS initializes the Tutor VDP, PSG, keyboard, controllers, frame
   timing, and compatibility RAM.
3. It scans the virtual cartridge GROM bases for valid TI `>AA` headers and
   linked program records.
4. It builds the menu from the module names and entry addresses it found.
5. Selecting a program initializes the TI GPL workspace and starts the GPL
   interpreter.
6. GPL bytecode is fetched through virtual GROM and executed by the native
   ESE interpreter.
7. Console calls, graphics, sound, keyboard, joystick, timing, and QUIT are
   translated to the Tutor environment.

The BIOS does not contain a hard-coded PHM3001 launcher. Cartridge discovery
is based on the module's standard header and program records.

## Virtual GROM

TI GROM is serial memory. One physical GROM supplies 6 KiB of useful bytes in
an 8 KiB logical slot. The Tanam board supplies ordinary parallel EPROM, so a
converter removes the unused 2 KiB tails and packs up to four GROM payloads
into 24 KiB.

```text
GPL asks for logical GROM byte >A123
                 |
                 v
       determine slot and 6 KiB offset
                 |
                 v
   read the corresponding Tanam EPROM byte
```

The GPL program continues to use logical bases `>6000`, `>8000`, `>A000`, and
`>C000`. Only the BIOS knows where those packed bytes physically reside.

## Console compatibility

The ESE does not emulate every chip and instruction of a complete TI-99. It
implements the behavior visible to cartridge software, including:

- GPL instructions and interpreter state;
- GROM address/read behavior;
- required console GROM and ROM services;
- VDP access, recovery timing, sprites, patterns, and screen operations;
- PSG sound lists and frame service;
- TI-99/4A keyboard, function-key, and joystick results;
- cartridge discovery, menu selection, and `FCTN-=`/QUIT.

New services are implemented from authoritative binaries, source listings,
manuals, and side-by-side traces rather than by patching individual screen
symptoms.

## Native cartridge code

Some modules contain TMS9900 CPU ROM as well as GPL. The Tutor's TMS9995 can
execute much of that instruction set directly, but the code must be relocated
and must not access TI-only hardware addresses.

The experimental mixed-module path keeps the GPL front end and places
relocated native code in unused Tanam ROM. `TTBI` vectors expose stable ESE
services, while a `TTEX` descriptor identifies the optional module extension.
Tombstone City proves the approach. Parsec is the next stress test and is not
yet a hardware release.

Important rules learned from these ports are:

- preserve R11 when a nested helper must not destroy an outer `BL` return;
- reconstruct data words placed inline after service calls;
- relocate pointers stored in data, not only visible instruction operands;
- preserve code layout where short branches and copied loops depend on it;
- keep private native workspaces out of the emulated TI scratchpad;
- trace PC, WP, R11, parameters, and bus addresses at service boundaries.
