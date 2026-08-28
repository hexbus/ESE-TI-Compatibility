# Pyuuta/Tutor Connectors and Model Variants

This page normalizes the conflicting connector information collected so far. A signal listed here is not automatically proof of its electrical behavior; see the evidence ledger for confidence and validation status.

## Numbering conventions

- Original Pyuuta cartridge connector: 36 contacts. The user reports that even-numbered contacts face the front of the machine.
- Rear expansion connector: 50 contacts. The user reports that odd-numbered contacts are on the upper side.
- Signal names beginning with `/` are active low.
- TI-style hexadecimal addresses are written with `>`.

## Original Pyuuta cartridge connector J-1

| Odd | Signal | Even | Signal |
|---:|---|---:|---|
| 1 | GND | 2 | GND |
| 3 | D7 | 4 | N.C. |
| 5 | D6 | 6 | N.C. |
| 7 | D5 | 8 | A15 / CRUOUT |
| 9 | D4 | 10 | A13 |
| 11 | D3 | 12 | A12 |
| 13 | D2 | 14 | A11 |
| 15 | D1 | 16 | A10 |
| 17 | D0 | 18 | A9 |
| 19 | VCC | 20 | A8 |
| 21 | `/CS CROM1` | 22 | A7 |
| 23 | A14 | 24 | A3 |
| 25 | A2 | 26 | A6 |
| 27 | GROMCLK | 28 | A5 |
| 29 | `/DBIN` | 30 | A4 |
| 31 | `/WE` / CPUCLK | 32 | N.C. |
| 33 | N.C. | 34 | N.C. |
| 35 | `/CS CROM0` | 36 | N.C. |

The original front connector does not expose a complete low-address/control set for the later RAM/large-ROM cartridges. This is why the rear Game Adapter—or a console modification—is needed.

## Rear expansion connector C-1

| Odd | Signal | Even | Signal |
|---:|---|---:|---|
| 1 | GND | 2 | GND |
| 3 | D7 | 4 | `/INT1` |
| 5 | D6 | 6 | `/HOLD` |
| 7 | D5 | 8 | A15 / CRUOUT |
| 9 | D4 | 10 | A13 |
| 11 | D3 | 12 | A12 |
| 13 | D2 | 14 | A11 |
| 15 | D1 | 16 | A10 |
| 17 | D0 | 18 | A9 |
| 19 | `/IOPORT` | 20 | A8 |
| 21 | `/MEMEN` | 22 | A7 |
| 23 | A14 | 24 | A3 |
| 25 | A2 | 26 | A6 |
| 27 | READY | 28 | A5 |
| 29 | `/DBIN` | 30 | A4 |
| 31 | `/WE` / CPUCLK | 32 | A1 |
| 33 | `/INT4` / EC | 34 | A0 |
| 35 | SELEXM | 36 | ROMCLK |
| 37 | `/RESET` | 38 | `/EXP0` |
| 39 | `/VDP` | 40 | `/EXP1` |
| 41 | `/SOUND` | 42 | `/EXP2` |
| 43 | `/EXM00` | 44 | `/EXP3` |
| 45 | `/EXM40` | 46 | `/EXM80` |
| 47 | LAQ / HOLDA | 48 | `/KILL SROM` |
| 49 | VCC | 50 | VCC |

Some supplied diagrams assign the final expansion-select positions differently. The table above follows the user-supplied rear-port material, but contacts 38–46 must be continuity-checked before hardware is built.

## Rear decode signals

| Signal | Reported selection |
|---|---|
| `/KILL SROM = 0` | Disable the internal system ROM |
| `SELEXM = 1` | Select expansion memory |
| `SELEXM = 0` | Select internal, cartridge, or I/O space |
| `/EXM00` | `>0000–>3FFF` expansion memory |
| `/EXM40` | `>4000–>7FFF` expansion memory |
| `/EXM80` | `>8000–>BFFF` expansion memory |
| `/EXMC0` | `>C000–>FFFF` expansion memory; this label appears in the decode schematic although it is absent from one connector table |
| `/EXP0` | `>E600–>E7FF`; possible ROM/signature service |
| `/EXP1` | `>E800–>E9FF`; printer I/O |
| `/EXP2` | `>EA00–>EBFF`; function not yet identified |
| `/EXP3` | `>EC00–>EDFF`; possibly numeric-keypad I/O |
| `/CROM0` | `>8000–>BFFF` cartridge memory |
| `/CROM1` | `>E400–>E5FF` cartridge I/O or auxiliary cartridge ROM |
| `/IOPORT = 1` | `>0000–>BFFF` or external-memory access |
| `/IOPORT = 0` | `>C000–>FFFF` I/O-side access |

`/EXP0` containing `55` at `>E600` is a useful lead, not yet a proven boot or device signature.

## Cartridge variants

### Original Pyuuta

The original Pyuuta needs a rear Game Adapter to supply the missing low address and control signals to enhanced front cartridges.

### Pyuuta with Game Adapter, Pyuuta MkII, and Jr

Tanam's RAM/ROM cartridge convention reports:

| Cartridge contact | Signal |
|---:|---|
| 6 | SELEXM |
| 27 | A1 |
| 31 | A0 |
| 32 | `/WE` |

The Tutor and Pyuuta MkII are reported to have the required signals at their cartridge connector already. That needs per-model continuity validation before one universal PCB pinout is published.

### Modified original Pyuuta, informally “Mk3”

Tanam's console modification routes rear-expansion signals to unused front contacts:

| Cartridge contact | Signal |
|---:|---|
| 4 | A1 |
| 6 | SELEXM |
| 31 | `/WE` |
| 33 | A0 |

This modified pinout is not cartridge-compatible with every unmodified model and must be labeled clearly.

## Known enhanced boards

### ESE rear Game Adapter

- Plugs into the rear expansion connector and presents an enhanced cartridge connector.
- Contains a Winbond W27C512-class EPROM and GAL22V10-class logic in the photographed example.
- Can override the original Pyuuta system ROM with either the US Tutor system
  ROM or Japanese Pyuuta system ROM. Tutor BASIC itself is a separate 16 KiB
  option ROM and is not supplied by the ESE alone.
- The physical EPROM is 64 KiB; the exact decoded/banked payload visible to the CPU is determined by its GAL equations.

### Green RAM and ROM cartridge

- Contains a 27C512-class 64 KiB EPROM.
- Contains a Sony CXK58257-class 32 KiB SRAM.
- Uses GAL22V10-class decode/banking logic.
- The current user-tested arrangement is described as 32 KiB ROM plus 8 KiB SRAM, with other modes selected by jumpers or GAL behavior.
- Do not infer the actual CPU-visible SRAM range from chip capacity alone.

### Green flash cartridge

- Uses a flash device plus GAL22V10 decode.
- A15, A16, and A17 jumpers select flash image/bank bits.
- The photographed board is marked `PYUTA FLASH MEMORY CARTRIDGE V1.3
  (FRONT)`. Its operational label says `0 Pitfall, 1 BASIC` and
  `GA sett 2,16k`.
- The independently read EN29F002 has data only in its first 64 KiB. With A16
  and A17 selecting page zero, A15=0 selects physical `>00000->07FFF`
  (Pitfall) and A15=1 selects `>08000->0FFFF` (BASIC/support firmware).
- It is suitable for repeatable diagnostic images once its decode is captured.

## Hardware validation checklist

Before publishing a construction pinout:

1. Choose one physical viewing orientation and photograph the connector with contacts 1 and 2 marked.
2. Continuity-test every ground, VCC, data, address, and control contact.
3. Compare original Pyuuta, Tutor, MkII/Jr, Game Adapter output, and modified “Mk3” separately.
4. Confirm `/WE` is not confused with CPUCLK on variants that multiplex or relabel contact 31.
5. Record voltage, idle state, and active polarity for every control line.
6. Capture GAL fuse maps or derive truth tables from hardware tests.

Until this checklist is complete, the connector tables are engineering references, not a guarantee that differently numbered historical drawings can be mixed safely.
