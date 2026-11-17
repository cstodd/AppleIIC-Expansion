# Open questions

Research to do before any //c design work. Building starts only after the Laser 128EX card is validated
on real hardware.

## Card behaviour
The //c card is "functionally identical" to the IIe card (Apple II Memory Expansion Card), so its docs and
schematic apply too. See [sources.md](sources.md).
- [x] Registers (card manual): 20-bit pointer; `$C0C2` holds the high nibble (D3–D0); load low → mid → high; the data
      register auto-increments. See [card-reference.md](card-reference.md).
- [x] `$C0C2` reads back with D7–D4 = 1; the pointer wraps `$FFFFF` → `$00000`; all registers must read back with the
      auto-increment applied. See [firmware-analysis.md](firmware-analysis.md) (IIe card ROM 341-0344A, confirmed
      identical in //c ROM `$03` and `$04`).
- [x] /RESET: turns the registers **off** until any access to `$C400`–`$C4FF` (`$Cn00`) re-enables them (Memory Expansion
      Card Tech Note #1). That's why J13 carries A8–A15 and /RESET. Still open: is the address register cleared too?
- [x] The mirrors at `$C0C4–$C0CF` (no A2/A3 on J13): the card firmware never touches them.
- [ ] Does the motherboard drive or float D0–D7 during a `$C0Cx` read? That decides whether the card can drive the
      bus directly.

## Timing (scope on a real //c and //c+)
- [ ] Read: data setup and hold around the falling edge of PH0.
- [ ] Write: when data is valid relative to PH0 and R/W.
- [ ] //c+ (4 MHz): what Apple's card needs a different PAL for, and whether `$C0Cx` accesses slow to 1 MHz.
  Lead: the IIc Plus CGGA runs **port 4 fast** by default (Appendix H). Conflicting reports: Apple Tech Note #7 says the
  original card doesn't work in the //c+, but forum sellers say Apple's card and the Chinook C-RAM work there. Apple had a
  separate //c+ card ("Tiger Tail Plus", per Jeff Mazur on Applefritter).
  Lesson from the 128EX: strobes timed off the CPU at 1 MHz can come from a different path when accelerated.

## Mechanical
- [x] Mounting: component side down on four nylon stand-offs, connector on the right, under the keyboard. Only
      logic board A2S4100 has the connector (dealer instructions).
- [ ] Connector part number and height; stand-off hole positions; card outline.
- [ ] The break at the pin 17/18 row: shown in the IIc Technical Reference's Fig. 11-43 and in the card's J1 (card manual
      Fig. 2-1). A split or keyed connector? Check a real card.
- [ ] Clearance under the keyboard.
- [ ] //c+: the memory card "bridged over the CPU" (Jeff Mazur, Applefritter). Check the //c+ connector position and the
      height over the CPU before assuming one board fits both machines.

## Firmware
- [x] Size detection: probes 64K steps from `$F0000` down with `$A5`; addresses above the fitted memory must not alias.
      See [firmware-analysis.md](firmware-analysis.md).
- [x] //c ROM `$03`/`$04`: same driver as the IIe card ROM. The only card-related change in `$04` is the STATUS reply
      (online only when memory is found). See [firmware-analysis.md](firmware-analysis.md).
- [x] RTC baseline: port the Laser rev 0.2 interface (unlock key + SPI port, 122-byte driver in ProDOS's clock slot),
      retargeted to `$C0Cx`/`$C400`. Optional extra: emulate AE's **DClock** (a DS1215 phantom clock on D0 behind the RAM
      at `$08xxxx`) so DClock-specific software works too. See [rtc-dclock.md](rtc-dclock.md).
- [ ] (Deferred until the RTC choice is made.) Whether to stay DClock-compatible. If yes, the CPLD emulates the phantom-clock
      *protocol*; no DS1215/DS1315 is fitted. Then get the exact recognizer rules from the DS1215/DS1315 data sheet (reads during
      recognition, whether the RAM write still happens).
- [ ] Battery-back the SRAM too, as the a2heaven RamExpress II+ does? (product decision)

## Sources to find
- Apple II Memory Expansion Card (670-0024) schematic (applelogic.org, blocked here)
- Schematics of Apple's card or the Sequential Systems / AE RAM Express IIc (PAL equations)
- IIc+ technical notes (accelerator and memory expansion)
