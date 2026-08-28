# Tomy Tutor/Pyuuta (ぴゅう太) ESE TI-99 Compatibility Shim v3.0

This directory is a public technical overview of a compatibility BIOS that
runs selected TI-99/4A cartridge software on a Tomy Tutor/Pyuuta using an ESE
Game Adapter and a Tanam ROM/RAM cartridge board.

It is not full TI-99 emulation. The compatibility BIOS executes TI GPL,
recreates the console services cartridge programs expect, and drives the
Tutor's real TMS9918-family video and SN76489-family sound hardware. Programs
with suitable TMS9900 native code can also be relocated to execute directly
on the Tutor's related TMS9995 CPU.

```text
ESE Game Adapter                     Tanam ROM/RAM cartridge
16 KiB replacement BIOS              converted TI module + working RAM
          |                                      |
          +------------------+-------------------+
                             |
                    Tomy Tutor/Pyuuta
             TMS9995 + TMS9918 + SN76489
```

## Documentation

- [Complete technical documentation index](docs/README.md)
- [How the compatibility system works](docs/HOW-IT-WORKS.md)
- [Hardware and memory map](docs/HARDWARE-AND-MEMORY.md)
- [Converting and running GROM modules](docs/USING-GROM-MODULES.md)
- [Current compatibility status and limits](docs/STATUS.md)

## Current hardware arrangement

- ESE Game Adapter set to `2/16`.
- ESE W27C512 containing the matched 64 KiB compatibility image.
- Tanam ROM/RAM cartridge board containing a matched 64 KiB module image.
- Original Tomy Tutor/Pyuuta video, sound, keyboard, and controllers.

The ESE image and module image must be built and tested as a matched set. Do
not combine an experimental native-module image with an older ESE core.

## Demonstrated modules

The RC6 hardware candidate has exercised PHM3001 Demonstration, PHM3023 Hunt
the Wumpus, PHM3032 Blasto, and an experimental GPL/native PHM3052 Tombstone
City port. The documents describe the remaining timing and input limitations.
PHM3031 The Attack is still a diagnostic case and is not qualified.

## Repository scope

This public directory intentionally contains documentation only. TI cartridge
ROM/GROM images are not included. Before publishing a larger source archive,
remove proprietary cartridge binaries unless their redistribution is
authorized.

The engineering project and its release archives contain the assembler,
conversion tools, manifests, emulator tests, recovered-source notes, and the
complete chronological worklog.
