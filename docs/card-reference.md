# Apple IIc Memory Expansion Card: hardware facts

Source: *Apple IIc Memory Expansion Card Technical Reference* (APDA A2G0047, © 1987, packaged 7/13/88),
Chapter 2 "Hardware" pp. 5–8 and Appendix A. (The OCR text reads "Apple lie", but the registers are at `$C0C0` (slot 4)
and the connector and ROM map are the //c's, so this is the IIc manual.) The PDF was supplied by the project owner;
it's not committed here (copyright APDA/Apple). Crops: [`ref/mec-fig2-1-component-layout.png`](ref/mec-fig2-1-component-layout.png),
[`ref/mec-fig2-2-connector.png`](ref/mec-fig2-2-connector.png).

## Registers

The card has "a 20-bit address register and a 1-byte data register".

| Address | Register |
|---|---|
| `$C0C0` | Address low byte |
| `$C0C1` | Address middle byte |
| `$C0C2` | Address **high nibble (D3–D0)** |
| `$C0C3` | Data. "the address register automatically increments after each access of the data register", so avoid indexed instructions that make false reads |

- "The address pointer **must be loaded in the following order: low byte, middle byte, high byte.**" Our design should
  behave correctly for that order. Whether another order also works on Apple's card is unknown, so don't depend on it.
- The manual says nothing about what bits D7–D4 of `$C0C2` read back as. (The Laser's equivalent reads them as 1.)
- RESET disables the registers until a `$C4xx` access (Memory Expansion Card Tech Note #1, see [sources.md](sources.md)).

## Parts

| Part | What it does |
|---|---|
| 32 × 256K×1 DRAM (U1–U32), 4 rows of 8 | 256K steps up to 1 MB |
| **Gate array** (UA18), "custom CMOS IC that provides the standard Apple IIc I/O bus electrical interface" | RAM refresh, RAM addressing (the 20-bit auto-increment pointer), timing, address decoding |
| **PAL 16L8A-2CN** | "on-board control logic for the RAM array" |
| **74HCT245** bus driver (the text says "74LHCT245") | Data bus buffer to J1 |

## Refresh (DRAM only, not needed for SRAM)

"Each 256K of dynamic RAM is refreshed every 4 milliseconds. A refresh cycle is performed during **alternating phase 1
clock cycles**. During a refresh cycle, one 256K set of RAM is refreshed; the remaining sets … during subsequent refresh
cycles … does not conflict with CPU access time. A 256K RAM refresh is completed in about 2 milliseconds."

So the card does all DRAM work during PH1 (PH0 low), when the CPU isn't using the bus. That's why the connector has
PH0, Q3 and 7M.

## Connector

- Fig. 2-2 gives the same pinout as schematic 050-0180, including **pin 23 = A14**. Only the table in the IIc Technical
  Reference (Fig. 11-43) differs; it shows "A15" twice. See [j13-memory-expansion-connector.md](j13-memory-expansion-connector.md).
- The text calls it a "34-pin card-edge connector", but the drawing shows a 2×17 pin field.
- Fig. 2-1 shows the card's connector **J1** at the right-hand end of the card, drawn as **two blocks with a gap at about
  the pin 17/18 row**. That matches the break in the IIc Technical Reference's outline drawing, so it's probably a split
  or keyed connector. Still to confirm on a real card or board.

## Card layout (Fig. 2-1)

- Long, narrow board; two rows of 16 DRAMs across the left two-thirds; the gate array UA18 (large square package) and J1 at
  the right end; two smaller ICs (the PAL and the '245) near the gate array; RP1–RP3 resistor packs; a notch on the top edge
  (marked J2A).
- There are no dimensions in the manual. Measure a real card, or take the outline from the //c logic board stand-off
  positions.

## Firmware (Appendix A)

- Main ROM `$C400`: card boot code and the ProDOS/DOS entry point. `$C4FF` is the low byte of the entry address; the SmartPort
  dispatch is at `$C400 + ($C4FF) + 3`. ID bytes: `$C401=$20`, `$C403=$00`, `$C405=$03`, `$C407=$00`.
- Auxiliary ROM `$D800–$DFFF`: the driver (`xread`, `xwrite`, `prdblk`, `pwrblk`…), `testsize` and `format` at `$DA00`, and
  diagnostics `$DD00` (`addresstest`, `rollovertest`, `addbustest`…). To see exactly how the firmware sizes the card and
  what it expects from the high nibble, disassemble `testsize` and `rollovertest` from a ROM `$03`/`$04` image.
- The card holds no firmware of its own.

## What this means for our design

- One gate array + PAL + '245 ≈ one CPLD + '245. The CPLD would implement the `$C0C0–$C0C3` decode, the 20-bit pointer
  with auto-increment on data access, the RESET disable / `$C4xx` enable, and the SRAM strobes in PH0. No refresh logic.
- The firmware only ever sees the register interface, so an SRAM card is invisible to software as long as
  `testsize`/`rollovertest` pass. Check those routines before choosing how the high nibble reads back and how the pointer
  wraps at 1 MB.
