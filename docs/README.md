<!-- SPDX-FileCopyrightText: 2026 hexbus -->
<!-- SPDX-License-Identifier: CC-BY-4.0 -->

# Technical documentation index

This directory contains the public technical reference for the Tomy
Tutor/Pyuuta ESE TI-99 Compatibility Shim. It deliberately excludes original
TI or Tomy ROM/GROM images, private filesystem paths, raw worklogs, and
unqualified experimental binaries.

## Start here

1. [Technical brief](technical-brief.md) — concise explanation for readers
   familiar with the TI-99/4A but new to the Tutor implementation.
2. [Runtime architecture](ese-runtime-architecture.md) — consolidated system
   view covering boot, cartridge discovery, GPL, services, memory, and native
   extensions.
3. [How the compatibility system works](HOW-IT-WORKS.md) — shorter overview.
4. [Current status and limits](STATUS.md) — hardware-tested baseline and
   unsupported facilities.

## ESE and Tanam hardware

- [Hardware and memory map](HARDWARE-AND-MEMORY.md)
- [Connector and model reference](connectors-and-models.md)
- [Tanam parallel ROM windows and W27C512 layout](tanam-parallel-rom-windows.md)
- [Present and future memory-map roadmap](compatibility-memory-map-roadmap.md)

## GPL and TI console compatibility

- [Self-testing GPL shim BIOS](part-1-gpl-shim-bios.md)
- [GPL conformance matrix](gpl-conformance-matrix.md)
- [TI console ROM service map](ti-console-rom-service-map.md)
- [Joystick translation](joystick-translation-plan.md)

## Module images and extensions

- [Converting and running GROM modules](USING-GROM-MODULES.md)
- [Detailed GROM-to-Tanam W27C512 procedure](grom-to-tanam-rom.md)
- [Module extension and fixed-core service ABI](module-service-extension-abi.md)
- [Relocatable compiled-GPL architecture](relocatable-compiled-gpl-architecture.md)

## Compatibility tracking

- [One-to-four-GROM compatibility checklist](grom-compatibility-checklist.md)

The checklist distinguishes structural conversion, bounded emulator smoke
tests, and physical gameplay qualification. A module reaching its title or
menu is not automatically considered compatible.
