<!-- SPDX-FileCopyrightText: 2026 hexbus -->
<!-- SPDX-License-Identifier: CC-BY-4.0 -->

# Tutor-to-TI-99/4A joystick translation plan

Joystick support belongs in the compatibility BIOS, alongside keyboard
`KSCAN`, rather than in individual cartridge patches. A cartridge must see the
same TI-99/4A player modes, axis bytes, fire key codes, and key-status behavior
regardless of how the Tutor controller is wired.

## Current state

The shim's GPL `SCAN` path implements the modified Tutor keyboard map,
including MOD/FCTN and MON/CTRL. Scan modes 1 and 2 now select the matching
Tutor controller, translate its axes through the original TI GROM0 table, and
continue with the corresponding half-keyboard scan when fire is not pressed.

The implemented physical input path uses the Tutor/Pyuuta controller
multiplexer and the same select mechanism already exercised by the native
DoorDoor port:

| Tutor operation | Controller 1 | Controller 2 |
|---|---:|---:|
| Write three-bit column selector at CRU base `>0024` | `6` | `7` |
| Fire input | CRU `>0006` | CRU `>0006` |
| Left input | CRU `>0008` | CRU `>0008` |
| Right input | CRU `>000A` | CRU `>000A` |
| Down input | CRU `>000C` | CRU `>000C` |
| Up input | CRU `>000E` | CRU `>000E` |

The selector determines which physical controller supplies the five shared
input bits. At the Tutor CRU/software boundary these contacts are
**asserted-high**: a released control reads `0` and a pressed control reads
`1`. This is proven independently by the original Tutor BIOS scanner and
Tanam's working DoorDoor scanner. The TI direction-table index is active-low;
that inversion belongs after the Tutor read, not at the physical boundary.

## TI-99/4A contract to reproduce

The original console ROM routine at CPU `>02B2` is the primary oracle:

- `KEYBRD`/player mode `1` selects joystick 1 and keyboard unit 1.
- Mode `2` selects joystick 2 and keyboard unit 2.
- The five hardware inputs are fire plus four directions.
- Fire first uses TI internal matrix indices `>29` for player 1 and `>25` for
  player 2. The original GROM0 unit table maps both to final `KEYCOD >12`.
- Directions update GPL workspace bytes `JOYY` at `>8376` and `JOYX` at
  `>8377`.
- Axes are not invented by the shim. The console indexes the original GROM0
  table at `>16E0` and obtains one signed byte for each axis.
- When fire is not pressed, KSCAN can continue into the appropriate
  half-keyboard scan. Its previous-key comparison, debounce, `KEYCOD`, and
  `STATUS` semantics still apply.

The exact 16-entry GROM0 axis table, shown as `(JOYY, JOYX)`, is:

```text
raw 0: (  0,  0)   raw 4: (  0,  0)   raw 8: (  0,  0)   raw C: (  0,  0)
raw 1: (  0,  0)   raw 5: (  4,  4)   raw 9: ( -4,  4)   raw D: (  0,  4)
raw 2: (  0,  0)   raw 6: (  4, -4)   raw A: ( -4, -4)   raw E: (  0, -4)
raw 3: (  0,  0)   raw 7: (  4,  0)   raw B: ( -4,  0)   raw F: (  0,  0)
```

The raw index retains the TI hardware's ordering and polarity. The shim reads
the Tutor's asserted-high direction contacts, discards fire, inverts the four
direction bits, and then performs the table lookup. This preserves the
console's behavior for diagonals and contradictory direction combinations.

## Compatibility boundary

Use a small two-stage interface:

```text
tutor_joy_read(port) -> {fire, left, right, up, down}
ti_kscan_joy(player, canonical_state) -> KEYCOD, STATUS, JOYY, JOYX
```

This separation is important. The first stage owns Tutor column selection,
input polarity, and future keyboard/gamepad profiles. The second stage owns
TI-visible behavior and can be tested byte-for-byte against the console ROM.

No cartridge should read Tutor CRU addresses directly. In particular, Hunt
the Wumpus' keyboard controls remain keyboard input; adding joystick support
must not disturb its arrows, Q/E actions, or FCTN combinations.

## Implementation and verification sequence

1. Add a compact native `tutor_joy_read` helper for exactly one requested port;
   do not OR both controllers together.
2. Reproduce the TI raw direction index and use the retained GROM0 `>16E0`
   table for the two axis bytes.
3. Integrate the helper only into KSCAN modes 1 and 2. Preserve modes 0 and
   3–5 and the existing Tutor keyboard profiles.
4. Feed fire through the normal KSCAN prior-key/debounce path and return final
   `KEYCOD >12`; do not turn directions into ordinary character keys.
5. Extend the deterministic CRU harness with a latched controller selector and
   independent port states.
6. Run exhaustive tests for every direction combination, fire alone,
   fire-plus-direction, held/released fire, both player modes, and simultaneous
   keyboard input.
7. Physically verify both Tutor controller ports before promoting the binary.

Steps 1–6 are complete. The JS99er-backed TMS9900 harness executes the actual
assembled `KEYGET` routine and passes 117 checks across both controller ports,
including neutral-controller ordinary keys and TI-99/4A REDO/BACK/QUIT.
Step 7 remains the hardware promotion gate.

## Acceptance tests

- Neutral input always returns `JOYY=0`, `JOYX=0`.
- All cardinal and diagonal directions match TI GROM0 byte-for-byte.
- Opposite directions match the original table rather than an assumed clamp.
- Player 1 never observes player 2 and vice versa.
- Fire generates exactly one new-press event when held, then another only
  after release and re-press.
- Fire returns final `KEYCOD >12` for both players, matching the original
  GROM0 unit table.
- Keyboard unit scanning still works in player modes 1 and 2.
- MOD/FCTN, MON/CTRL, and FCTN-= remain unchanged.
- Demonstration, Wumpus, Blasto, and the non-joystick batch remain regression
  clean.

## Evidence

- TI console ROM source: `ROM-4A_A.asm`, KSCAN at `>02B2`.
- Retained TI GROM0 image: axis table at GROM address `>16E0`.
- Tutor BIOS vector `>0044` reaches its controller normalizer at `>0DEC` and
  stores the two normalized controller bytes at `>F0EA`–`>F0EB`.
- The independent DoorDoor Tutor port reads prove selector columns 6/7 and the
  five CRU input positions above.
