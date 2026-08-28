# Running TI-99 GPL cartridges on a Tomy Tutor/Pyuuta

## Short explanation

This project runs selected TI-99/4A cartridge software on a Tomy
Tutor/Pyuuta by replacing the Tomy's BIOS with a purpose-built compatibility
BIOS. It is not a complete TI-99 emulator.

The compatible parts of the two machines are used directly. Both use a
TMS9900-family CPU architecture, a TMS9918-family video processor, and a
closely related TI/SN sound generator. The compatibility BIOS supplies the
parts that are different: TI GPL execution, GROM addressing, console
ROM/GROM services, keyboard scanning, joystick translation, timing, and the
different hardware port addresses.

## The hardware arrangement

Two boards work together:

1. The **ESE Game Adapter** replaces the Tomy system BIOS. With the ESE set to
   `2/16`, a 16 KiB compatibility core is visible at CPU `>0000->3FFF`.
2. The **Tanam ROM/RAM cartridge board** holds the converted TI cartridge data
   and working RAM. It exposes 24 KiB of ROM in two CPU windows and an 8 KiB
   SRAM window.

```text
ESE at >0000->3FFF
  reset + TOMY title/menu + GPL interpreter + TI console services
                         |
                         v
Tanam ROM at >4000->5FFF and >8000->BFFF
  packed TI cartridge GROM data + optional relocated native code

Tanam SRAM at >6000->7FFF
  TI GPL scratchpad and module working state
```

The ESE and Tanam devices are programmed as a matched pair. The Tutor's real
VDP, PSG, keyboard, and controllers remain in use.

## Why the TI GROM image is converted

TI GROM is serial memory with its own address counter. A physical GROM
contains 6 KiB of useful data in an 8 KiB logical slot. The Tanam cartridge
contains ordinary parallel EPROM, not GROM hardware.

The conversion tool removes each unused 2 KiB GROM tail and packs as many as
four 6 KiB payloads into the Tanam board's 24 KiB ROM space. At run time the
compatibility BIOS translates a logical GROM request into a read from the
correct packed EPROM byte:

```text
GPL requests logical GROM byte >A123
              -> identify logical slot and 6 KiB offset
              -> read the packed byte from a Tanam CPU ROM window
```

The cartridge program still uses its original logical GROM addresses. The
packing detail is invisible to GPL.

## What happens at power-up

1. The TMS9995 resets into the ESE compatibility core.
2. The core initializes the Tutor VDP, sound, keyboard, controllers, timing,
   and compatibility RAM.
3. It scans the virtual cartridge GROM bases for valid TI `>AA` headers and
   program records.
4. It displays a TOMY title screen and constructs a menu from the names and
   entry addresses found in the cartridge. Module names are not compiled into
   the BIOS.
5. The selected program runs in the GPL interpreter.
6. GPL instructions and TI console calls are translated into operations on
   the Tutor's real hardware.
7. TI-99/4A function keys, keyboard codes, and joystick states are translated
   from the Tutor inputs. `FCTN-=`/QUIT returns to the TOMY master title.

## What is interpreted and what is native

Many early TI cartridges are mostly or entirely GPL. Those modules run through
the ESE GPL interpreter while still using the Tutor hardware for display and
sound.

Some cartridges also contain TMS9900 CPU ROM. Because the Tutor's TMS9995 is
in the same instruction family, suitable native code does not need CPU
emulation, but it does need relocation and hardware adaptation. The
experimental mixed-module design does this by:

- retaining the original GPL front end;
- placing relocated native code in unused Tanam ROM space;
- replacing direct TI hardware and console calls with Tutor service calls;
- binding the module to stable ESE service vectors through `TTBI`/`TTEX`.

Tombstone City is the first working proof of this GPL/native method. Parsec is
being used as the next stress test. Its source rebuilds byte-for-byte to the
stock cartridge ROM, and the Tutor experiment removes speech while retaining
the PSG music and native game engine. That Parsec image is still under
development and is not part of RC6.

## What has been demonstrated

The current RC6 hardware candidate has physically exercised:

- **PHM3001 Demonstration** through its full loop and menu paths, with some
  timing differences still being studied;
- **PHM3023 Hunt the Wumpus** with working gameplay and joystick, although
  repeat remains sensitive;
- **PHM3032 Blasto** with working gameplay, joystick, and fire;
- **PHM3052 Tombstone City** as an experimental mixed GPL/native module.

PHM3031 The Attack remains a diagnostic case and is not qualified. Broader
automated testing has found many more pure-GROM modules that boot, but a
bounded smoke test is not being presented as full gameplay qualification.

## What the compatibility BIOS replaces

The BIOS does not simply copy the entire TI console firmware. It implements
the behavior that cartridge software can observe:

- GPL instruction semantics and state;
- virtual GROM reads and address behavior;
- the required GROM 0 and console ROM service contracts;
- VDP access and recovery timing;
- PSG sound-list scheduling and frame timing;
- TI-99/4A KSCAN, function keys, joystick axes, and debounce behavior;
- cartridge discovery, menu entry, and QUIT;
- selected arithmetic, scan/link, sprite, and module-service behavior.

The implementation is expanded when cartridge traces prove another required
service. The goal is a common, tested service layer rather than a pile of
module-specific visual patches.

## Current limits

The present unbanked Tanam layout holds at most four physical GROMs, or 24 KiB
of useful GROM data. Five-GROM modules need a future banked design. Modules
that require CPU ROM, 32 KiB expansion RAM, disk, cassette, RS232, speech,
MBX hardware, or other peripherals need corresponding services and possibly
another memory profile.

VDP RAM cannot replace the cartridge SRAM or firmware ROM. Cartridge programs
control nearly all 16 KiB of VRAM for screens, patterns, sprites, sound lists,
and general data, so the compatibility runtime must keep persistent state in
CPU-addressable RAM.

## Why this approach is useful

The project separates three concerns:

- original module logic remains in GPL wherever practical;
- platform differences are concentrated in a reusable compatibility BIOS;
- performance-critical native code can later bind to stable services rather
  than to fixed TI or Tutor addresses.

That gives a path from today's GPL interpreter to a future unified runtime
that can relocate or compile well-understood GPL blocks into native
TMS9900-family code without changing the program's higher-level behavior.

## Further technical detail

- [Documentation index](README.md)
- [ESE runtime architecture](ese-runtime-architecture.md)
- [Current memory-map roadmap](compatibility-memory-map-roadmap.md)
- [Tanam EPROM byte layout](tanam-parallel-rom-windows.md)
- [GPL conformance matrix](gpl-conformance-matrix.md)
- [Module extension ABI](module-service-extension-abi.md)
- [Compatibility checklist](grom-compatibility-checklist.md)
