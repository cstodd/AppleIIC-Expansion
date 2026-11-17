# Open questions

Research to do before any //c design work. Building starts only after the Laser 128EX card is validated
on real hardware.

## Card behaviour
The //c card is "functionally identical" to the IIe card (Apple II Memory Expansion Card), so its docs and
schematic apply too. See [sources.md](sources.md).
- [x] Registers (card manual): 20-bit pointer; `$C0C2` holds the high nibble (D3–D0); load low → mid → high; the data
      register auto-increments. See [card-reference.md](card-reference.md).
- [ ] What D7–D4 of `$C0C2` read back as, and how the pointer wraps at 1 MB. Answer from the ROM's `testsize` /
      `rollovertest` (aux ROM `$DA00`/`$DD00`).
- [x] /RESET: turns the registers **off** until any access to `$C400`–`$C4FF` (`$Cn00`) re-enables them (Memory Expansion
      Card Tech Note #1). That's why J13 carries A8–A15 and /RESET. Still open: is the address register cleared too?
- [ ] Do the mirrors at `$C0C4–$C0CF` (no A2/A3 on J13) matter? Does any firmware touch them?
- [ ] Does the motherboard drive or float D0–D7 during a `$C0Cx` read? That decides whether the card can drive the
      bus directly.

## Timing (scope on a real //c and //c+)
- [ ] Read: data setup and hold around the falling edge of PH0.
- [ ] Write: when data is valid relative to PH0 and R/W.
- [ ] //c+ (4 MHz): what Apple's card needs a different PAL for, and whether `$C0Cx` accesses slow to 1 MHz.
  Lead: the IIc Plus CGGA runs **port 4 fast** by default (Appendix H).
  Lesson from the 128EX: strobes timed off the CPU at 1 MHz can come from a different path when accelerated.

## Mechanical
- [x] Mounting: component side down on four nylon stand-offs, connector on the right, under the keyboard. Only
      logic board A2S4100 has the connector (dealer instructions).
- [ ] Connector part number and height; stand-off hole positions; card outline.
- [ ] The break at the pin 17/18 row: shown in the IIc Technical Reference's Fig. 11-43 and in the card's J1 (card manual
      Fig. 2-1). A split or keyed connector? Check a real card.
- [ ] Clearance under the keyboard.

## Firmware
- [ ] ROM `$03`/`$04` RAM-disk driver at `$C400`: how it detects card size. ROM `$03` always reports a card present;
      ROM `$04` fixes that (IIc Tech Note #7). Source/ROM images are on the mirror (see sources.md).
- [ ] RTC: how ProDOS and other software would find the DS3234 on a //c, which has no slots. (The Laser card's approach
      may not carry over.)

## Sources to find
- Apple II Memory Expansion Card (670-0024) schematic (applelogic.org, blocked here)
- Schematics of Apple's card or the Sequential Systems / AE RAM Express IIc (PAL equations)
- IIc+ technical notes (accelerator and memory expansion)
