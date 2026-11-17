# Open questions

Research to do before any //c design work. Building starts only after the Laser 128EX card is validated
on real hardware.

## Card behaviour (needs the *Apple IIc Memory Expansion Card Reference*, not found yet)
- [ ] Register details at `$C0C0–$C0C3`: how the unused high address bits read back on a 1 MB card.
- [ ] Does /RESET clear the address register?
- [ ] Do the mirrors at `$C0C4–$C0CF` (no A2/A3 on J13) matter? Does any firmware touch them?
- [ ] Does the motherboard drive or float D0–D7 during a `$C0Cx` read? That decides whether the card can drive the
      bus directly.

## Timing (scope on a real //c and //c+)
- [ ] Read: data setup and hold around the falling edge of PH0.
- [ ] Write: when data is valid relative to PH0 and R/W.
- [ ] //c+ (4 MHz): what Apple's card needs a different PAL for, and whether `$C0Cx` accesses slow to 1 MHz.
  Lesson from the 128EX: strobes timed off the CPU at 1 MHz can come from a different path when accelerated.

## Mechanical
- [ ] Connector type, height and location on the 050-0180 board; card outline and mounting.
- [ ] The break at the pin 17/18 row in Fig. 11-43: a key or polarisation feature?
- [ ] Clearance under the keyboard.

## Firmware
- [ ] ROM `$03`/`$04` RAM-disk driver at `$C400`: how it detects card size.
- [ ] RTC: how ProDOS and other software would find the DS3234 on a //c, which has no slots. (The Laser card's approach
      may not carry over.)

## Sources to find
- *Apple IIc Memory Expansion Card Reference* / technical notes
- Schematics of Apple's card or the Sequential Systems / AE RAM Express IIc (PAL equations)
- IIc+ technical notes (accelerator and memory expansion)
