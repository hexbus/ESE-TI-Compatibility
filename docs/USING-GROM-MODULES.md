<!-- SPDX-FileCopyrightText: 2026 hexbus -->
<!-- SPDX-License-Identifier: CC-BY-4.0 -->

# Converting and running GROM modules

## Candidate requirements

The current simple converter accepts modules with one to four physical GROMs.
A normal candidate must:

- start with a standard TI `>AA` cartridge header;
- contain at most 24 KiB of useful GROM payload;
- have its GROM files supplied in logical order;
- have no unhandled CPU-ROM, bank-switch, speech, expansion-RAM, or peripheral
  dependency.

“Pure GROM” describes cartridge storage, not total independence. A GROM-only
module can still call console GROM 1/2 or require cassette, disk, RS232,
speech, MBX hardware, a printer, or 32 KiB expansion RAM.

## Accepted source forms

The project converter supports:

| Input form | Accepted sizes |
| --- | --- |
| Individual physical GROM | 6,144 or 8,192 bytes each |
| Already packed payload | 6,144, 12,288, 18,432, or 24,576 bytes |
| Complete logical-slot image | 8,192, 16,384, 24,576, or 32,768 bytes |

For an 8 KiB physical-GROM dump, only the first 6 KiB contains driven data.
The converter discards the unused 2 KiB logical tail.

## Conversion command

From the full engineering repository, a one-GROM module is built with:

```powershell
python tools\make_tanam_grom_module.py `
  --grom "module-grom3.bin" `
  --output "build\PHMxxxx-Module-w27c512.bin" `
  --manifest "build\PHMxxxx-Module-w27c512.json"
```

For several GROMs, repeat `--grom` in logical order: GROM 3 (`>6000`), GROM 4
(`>8000`), GROM 5 (`>A000`), and GROM 6 (`>C000`).

The 64 KiB `*-w27c512.bin` file is the programmer image. Optional packed or
logical payload files are analysis artifacts and must not be burned in its
place.

## Programming procedure

1. Program a spare W27C512 with the matched ESE compatibility image.
2. Program another spare W27C512 with one generated Tanam module image.
3. Read both devices back and verify their hashes against the manifests.
4. Install the ESE device and select `2/16`.
5. Install the module device in the Tanam ROM/RAM cartridge board.
6. Power on and verify the TOMY title, cartridge discovery, program menu, and
   module name before testing gameplay.

## Qualification

A successful menu or bounded emulator launch is only a smoke test. Full
qualification should cover:

- title and every menu path;
- keyboard, function keys, joystick, fire, and debounce;
- sprites, patterns, colors, VDP-heavy transitions, and screen clearing;
- music, effects, simultaneous sounds, and silence at transitions;
- scoring, variables, random state, replay, death/win paths, and QUIT;
- any cassette, printer, speech, disk, or expansion-memory path.

Record both the ESE and module hashes with physical test results. That avoids
mistaking a mismatched pair or stale EEPROM for a compatibility defect.
