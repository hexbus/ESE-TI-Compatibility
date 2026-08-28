# Making a Tanam W27C512 ROM from TI GROM Dumps

## Scope

This procedure converts one to four TI cartridge GROM dumps into a 64 KiB
W27C512 programmer image for the **Tanam ROM/RAM cartridge board**. It is the
procedure used for Hunt the Wumpus and other GROM-only modules with the Tutor
compatibility shim running from the ESE at `2/16`.

This is not a general TI cartridge-ROM converter. A module that also contains
TMS9900 CPU ROM, bank switching, RAM, speech data, or a peripheral dependency
needs additional mapping and shim services. Audit the RPK or source files
before assuming that a module is GROM-only.

Always preserve the original cartridge EPROM and program a spare W27C512.

## Why the GROM file cannot be burned directly

A physical TI GROM drives 6 KiB of serial data inside an 8 KiB logical address
slot. The Tanam board presents ordinary parallel-ROM windows to the Tutor CPU:
16 KiB at `>8000->BFFF` and 8 KiB at `>4000->5FFF`. Therefore the module bytes
must be compacted, rearranged into those two windows, and placed into both
32 KiB selector halves of the W27C512.

The finished EPROM consequently contains repeated strings and apparently
out-of-order blocks in a hex editor. That is expected. It is a physical
parallel-window image, not a sequential GROM dump.

## Accepted input files

Use exactly one input mode:

| Option | Accepted sizes | Meaning |
|---|---:|---|
| `--grom FILE` | 6,144 or 8,192 bytes | One physical GROM dump; repeat up to four times |
| `--packed FILE` | 6,144, 12,288, 18,432, or 24,576 bytes | GROM devices already compacted back-to-back |
| `--slot-image FILE` | 8,192, 16,384, 24,576, or 32,768 bytes | One to four complete logical 8 KiB slots |

For an 8 KiB `.G` file, the converter uses its first 6 KiB and discards the
2 KiB logical tail. When supplying several `--grom` files, list them in logical
GROM address order: `>6000`, `>8000`, `>A000`, then `>C000`.

The first compacted GROM must begin with the TI cartridge header byte `>AA`.
The converter rejects an input that does not.

## One-GROM example: Hunt the Wumpus

From the repository root:

```powershell
python tools\make_tanam_grom_module.py `
  --grom "inputs\PHM3023-Hunt-the-Wumpus-grom3.bin" `
  --output "build\PHM3023-Hunt-the-Wumpus-w27c512.bin" `
  --payload-output "build\PHM3023-Hunt-the-Wumpus-logical-24k.bin" `
  --manifest "build\PHM3023-Hunt-the-Wumpus-w27c512.json"
```

The files have different purposes:

- `PHM3023-Hunt-the-Wumpus-w27c512.bin`: the **64 KiB file to program** into the
  W27C512;
- `PHM3023-Hunt-the-Wumpus-logical-24k.bin`: an optional logical/test payload—do not
  burn this file;
- `PHM3023-Hunt-the-Wumpus-w27c512.json`: sizes, input and output hashes, GROM count,
  board settings, and the exact byte layout.

## Multiple-GROM example

```powershell
python tools\make_tanam_grom_module.py `
  --grom "module-grom1.bin" `
  --grom "module-grom2.bin" `
  --grom "module-grom3.bin" `
  --output "build\module-w27c512.bin" `
  --manifest "build\module-w27c512.json"
```

Do not concatenate 8 KiB emulator files manually. Their 2 KiB slot tails must
be removed before the next physical GROM's 6 KiB is appended; the converter
does this automatically.

## Output layout

The converter first constructs a 24 KiB packed logical payload, filling absent
GROM devices with `0xFF`. It then creates one selected 32 KiB Tanam bank:

| Bank offset | Size | Tutor CPU window | Contents |
|---:|---:|---|---|
| `0x0000` | 16 KiB | `>8000->BFFF` | Packed offsets `0x2000->0x5FFF` |
| `0x4000` | 8 KiB | `>4000->5FFF` | Packed offsets `0x0000->0x1FFF` |
| `0x6000` | 8 KiB | SRAM-selected | Erased `0xFF` fill |

That 32 KiB bank is duplicated at W27C512 offsets `0x0000` and `0x8000`, so
both cartridge selector positions contain the same module unless a future
specialized packager deliberately creates two different banks.

## Programming and verification

1. Set the programmer device type to **W27C512** and program the generated
   65,536-byte `*-w27c512.bin` file.
2. Read the programmed chip back into a new file.
3. Compare its SHA-256 with the generated image:

```powershell
Get-FileHash -Algorithm SHA256 "build\module-w27c512.bin"
Get-FileHash -Algorithm SHA256 "build\module-w27c512-readback.bin"
```

The hashes must match exactly. For the present compatibility profile, set the
ESE to `2/16`; the generated image duplicates the module in both Tanam selector
halves.

## Common mistakes

- Burning the original 6 KiB or 8 KiB GROM dump directly.
- Burning the optional 24 KiB logical payload instead of the 64 KiB output.
- Manually concatenating 8 KiB `.G` files and retaining their 2 KiB tails.
- Supplying GROM devices in the wrong logical order.
- Treating a GROM+CPU-ROM module as GROM-only.
- Using this W27C512 layout for the different 29F002 flash-cartridge board.
- Skipping the programmer read-back/hash comparison.

The byte-level rationale and CPU-window equations are documented in
[Tanam Parallel ROM Windows and W27C512 Byte Layout](tanam-parallel-rom-windows.md).
