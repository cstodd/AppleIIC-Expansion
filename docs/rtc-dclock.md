# Real-time clock: the //c convention is AE's DClock

## Prior art: a2heaven RamExpress II+

a2heaven sells (when in stock) the **RamExpress II+**: 1 MB **static RAM**, a built-in RTC, and battery backup of both RAM
and clock (CR2032). It's "Compatible with Apple Memory Expansion Card, RamExpress and DClock from AE". It was designed by Plamen
Vaysilov "(VHDL design)/firmware, circuit design and the PCB". That's very close to what this project plans.
- Product page: `https://www.a2heaven.com/webshop/index.php?rt=product/product&product_id=144`
- User Guide v1.02 (Feb 2017): `https://www.a2heaven.com/webshop/resources/pdf_document/18/82/7.pdf`
- Disks: DClock Utility Disk (`…/resources/archive/18/82/5.zip`) and RamExpress II+ disk (`…/archive/18/82/6.zip`)

From the user guide:
- Works in **A2S4100** (//c ROM 3/4) **and A2S4500** (//c+). Not A2S4000 (ROM 255/0) boards, even with a ROM 3/4 chip fitted.
  The guide warns that the ROM version alone doesn't prove the connector is there.
- The card's pins go into the //c's memory expansion **socket**. Orientation: "with the front of the computer at the bottom"
  (Fig. 3). The card passes Apple's built-in test (`C40AG`, "CARD SIZE = 1024K").
- The clock is used through ProDOS via **`DCLOCK.SYSTEM`** (Applied Engineering), which patches the clock driver into
  ProDOS at boot. ProDOS doesn't recognise the clock by itself.

## The DClock protocol (from AE's own source)

The DClock Utility Disk includes `SOURCE.UTIL`: "DClock utility routines, written October 14, 1988 by John Stephen III,
Copyright (c) 1988 Applied Engineering" (ORCA/M listing of `AE.CLOCK.UTIL`). It isn't committed here (copyright). The
mechanism:

- The clock is a **Dallas phantom clock** (DS1215 / DS1216 "SmartWatch" type) sitting **behind bit D0 of the RAM byte at card
  address `$080000`** (high nibble `$8`). "After this pattern is sent the clock chip will bank itself in in place of the RAM chip
  at bit zero of bank $08 for 64 reads/writes."
- **Reset:** `LDA $CFFF`, `LDA $C400` (enable the card's registers), set the pointer to `$080000`, **read** `$C0C3`. That resets the
  pattern recognizer and returns the RAM byte, which the routine saves.
- **Enable:** write the 64-bit pattern **`C5 3A A3 5C C5 3A A3 5C`**, 8 bytes, each sent **LSB first on D0**, one bit per write
  to `$C0C3`. After every write the routine does `STZ $C0C0` to undo the auto-increment, so all 64 writes hit `$080000`.
- **Read time:** 64 reads of `$C0C3` (again `STZ $C0C0` after each one to stay on `$080000`), D0 = next bit, LSB first.
  **Write time:** 64 writes the same way.
- **Restore:** set the pointer to `$080000` and write the saved byte back. (The clock access corrupts bit 0 of that RAM byte.)
- **Data**, 8 BCD bytes in order: hundredths (`00`–`99`), seconds, minutes, hours (`00`–`23`), day of week (1 = Sun … 7 = Sat),
  date, month, year (`00`–`49` = 2000–2049, `50`–`99` = 1950–1999). Unused bits must be 0.
- **Detection:** read the time and validate the BCD ranges. If the data is invalid, write Jan 1 1950 00:00:00.00, read it again and
  validate. Clock present only if valid.

### The ProDOS driver patch (`AE.CLOCK.PTCH`, loads at `$D742`), disassembled

Same sequence (`PHP`/`SEI`, `LDA $C400`, read `$C0C3` to reset and save, 64 pattern writes on D0 with `STZ $C0C0` after each, 64 reads,
restore the byte), with one important difference: **it never writes the middle address register `$C0C1`**. It sets only `$C0C0 = $00`
and `$C0C2 = $08`, so the clock is accessed at **`$08xx00`, with xx left over from earlier RAM-disk use**. That's how a phantom clock
works: the DS1215 watches the chip-enable and D0 of the RAM chip it sits behind, not a particular address. So
**the emulation must respond to any address with high nibble `$8` (any middle and low byte)**. To behave like the real hardware, where
the clock sits behind one 256K×1 DRAM, treat the whole 256K row `$80000`–`$BFFFF` (A19:A18 = `10`) as the clock's chip: any access
there takes part in pattern recognition. The patch then converts the BCD time to ProDOS format at `$BF90`–`$BF93`.

## What this means for our card

- To work with existing //c software (DCLOCK.SYSTEM, ClockWorks for AppleWorks, the a2heaven/AE utilities), **the CPLD should
  emulate the DS1215 phantom-clock protocol on D0 at `$080000`**:
  - The clock's "chip" is the 256K row `$80000`–`$BFFFF` (at minimum: any address with high nibble `$8`, because the ProDOS patch
    leaves the middle byte unset).
  - A read there resets the recognizer.
  - 64 consecutive writes there are compared on D0 against `C53AA35CC53AA35C` (LSB first per byte). On a match, the next
    64 accesses there are clock bits on D0, not RAM.
  - Normal RAM-disk traffic in that row will often reset the recognizer. That's harmless, as with the real DClock. (Check the DS1215 data sheet for the exact rules: whether reads during
    recognition reset it, and whether the RAM write still happens during pattern writes.)
- This is emulation of the *protocol* only; no Dallas phantom-clock chip is fitted. The RTC part is a separate design choice.
- **Backing store:** e.g. the DS3234 we already use on the Laser card (or whatever RTC the new board uses). It has no hundredths, so return `00`. Day/date/month/year/time map
  directly (it's BCD too). The CPLD needs the 8 bytes ready when the clock banks in: either keep a shadow copy refreshed over
  SPI in the background, or burst-read over SPI on pattern match. (The driver loop takes ~20 µs per bit, so a burst read at
  1–4 MHz SPI is fast enough, but check the first-bit timing.) Writes: collect 64 bits, then write to the DS3234.
- Unlike the Laser card, the //c side needs **no new firmware**. The existing DClock driver does it all.
- **Possible extra:** a2heaven battery-backs the whole 1 MB SRAM. Our design could do the same (the SRAM needs a low-standby
  part and a power switchover). A product decision for later.

## Leads

- (Done) `AE.CLOCK.PTCH` disassembled; see above.
- ROM 4X / 5X enhanced //c ROMs (`github.com/mgcaret/rom4x`, linked from the a2heaven page) may also read the clock.
- DS1215 data sheet: the exact recognizer rules.
