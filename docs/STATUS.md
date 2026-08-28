# Compatibility status and limits

## Release baseline

RC6 is the current preserved hardware candidate. It is a useful engineering
baseline, not a production-qualified release.

| Module | Current physical status |
| --- | --- |
| PHM3001 Demonstration | Full demonstration loop and menu paths work; timing differences from a TI-99/4A remain under study |
| PHM3023 Hunt the Wumpus | Gameplay and joystick work; joystick/key repeat can advance twice on a short input |
| PHM3032 Blasto | Gameplay, joystick, and fire work well in current testing |
| PHM3052 Tombstone City | Experimental matched GPL/native port launches and gameplay works; not an all-GPL TI image |
| PHM3031 The Attack | Still lacks correct Player 1 joystick behavior on physical hardware; not qualified |

Automated smoke testing has entered many additional one-to-four-GROM modules,
but it proves only bounded interpreter/service progress. It does not prove all
gameplay paths or peripheral behavior.

## Present capabilities

- TI-style cartridge header and program-record discovery.
- One-to-four-GROM packed module images.
- GPL interpreter with the instruction and service subset exercised by the
  current module suite.
- Native VDP and PSG output with Tutor recovery timing.
- TI-99/4A keyboard/function-key and Tutor joystick translation.
- TI QUIT back to the TOMY master title.
- Experimental optional native extensions using fixed ESE service vectors.

## Known scope limits

- Five-GROM/30 KiB modules require banking.
- CPU-ROM modules require relocation and service adaptation.
- Modules requiring 32 KiB expansion RAM are pending a different RAM profile.
- Disk, RS232, speech, MBX, and most DSR devices are not supplied.
- CS1 cassette translation is designed but not yet a qualified runtime
  service; CS2 should report an error.
- Thermal-printer activity must be detected and safely rejected or stubbed
  until a device service exists.
- Console GROM 1/2 data or service dependencies must be identified per module.

## Development direction

The immediate work is correctness and trace-based module qualification. The
longer-term architecture keeps open:

- banked module storage for five GROMs and CPU ROM;
- a coordinated 32 KiB ESE profile;
- CS1 cassette and additional discoverable services;
- a semantic service registry shared by GPL and relocated native code;
- verified GPL-to-native optimization for frequently executed blocks.

The guiding rule is to fix shared interpreter or console-service contracts
once, then rerun the entire cartridge regression suite. Module-specific
patches are reserved for real hardware dependencies or intentional port
changes, not as substitutes for missing GPL semantics.
