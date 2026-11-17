# Card firmware: what it needs from the hardware

## Which ROM

No ROM `$03`/`$04` //c image was found on the Asimov mirror (the `apple_iic_rom.zip` image is the UniDisk 3.5 ROM,
`$FBBF = $00`; the Spanish IIc ROM is the original, `$FF`). This analysis uses the **Apple II Memory Expansion Card ROM
341-0344A** instead (`emulators/rom_images/Apple II Memory Expansion ROM_341-0344A.bin`), the IIe/II version of the card.
Apple IIc Tech Note #5 says the //c card is "functionally identical" to that card, and the //c ROM map (card reference,
Appendix A) has the same routines (`testsize`, `rollovertest`, `addresstest` …). So these requirements should carry
over, but **re-check against a ROM `$03`/`$04` dump** when one turns up.

ROM layout (4 KB): `$000` copyright text ("This wonderful, wonderful rom is brought to you by Richard Williams …"),
`$100`–`$7FF` one `$Cn00` page per slot, `$800`–`$FFF` the shared `$C800`–`$CFFF` code. The firmware reaches the
registers as `$BFF8,X`–`$BFFB,X` with X = `$80 + slot × 16` (`$C8` for slot 4, which gives `$C0C0`–`$C0C3`).
Disassembled with py65. Addresses below are `$C800`-space addresses in that ROM.

## Requirements found

1. **All three address registers must read back**, including the effect of auto-increment and the carries from low to
   mid to high. The format routine (`$CB7A`) clears memory by writing `$00` to the data register until it sees bits 7–4
   of the **mid** register become non-zero. Another loop (`$CBCC`) writes until the **low** register reaches a given value.
2. **`$C0C2` reads back with D7–D4 = 1.** The register diagnostic (`$CD40`–`$CD55`) writes each pattern
   `FF CC AA 55 33 00` to low, mid and high, then checks `low == p`, `mid == p`, and **`high == p OR $F0`**. (The Laser's
   `$C0D2` behaves the same way.) Normal driver code masks with `AND #$0F`, so only the diagnostic depends on this.
3. **The pointer wraps at 20 bits:** `$FFFFF` → `$00000`. The rollover diagnostic (`$CD61`–`$CD75`) steps across the end and
   then requires `(high AND $0F) OR mid OR low == 0`.
4. **Read-modify-write on the low register must work.** The size test uses `DEC $BFF8,X` (`$CAC5` …) to step the
   pointer back by one. A 65C02 RMW cycle reads the register twice and writes once (an NMOS 6502 writes the old value,
   then the new). Rule: **reading an address register has no side effects, and a write simply loads it.** Then every CPU
   behaves the same.
5. **The data register increments on reads and on writes** (both are used back to back in the size test and the
   diagnostics).
6. **Size detection (`$CAB1`):** for each 64K step, high nibble `$F` down to `$0`, set the pointer to `hi:0000`, save the
   byte there, write `$A5`, read it back, restore the saved byte. The first step that reads back `$A5` gives the size:
   (nibble + 1) × 64K. Consequence: **addresses above the fitted memory must not alias** onto real memory, or a
   small card is over-reported. With a full 1 MB of SRAM this can't happen. If we ever offer less, the extra addresses
   must not respond (or must read something other than what was written).
7. **Registers are written in the order low, mid, high** everywhere. Nothing in the code depends on side effects of
   writing one register (e.g. writing low doesn't clear mid/high).
8. **No accesses to `$C0C4`–`$C0CF`.** The mirrors caused by A2/A3 missing from J13 are harmless.

Also seen: the `CMP #$04` that should be `CPY #$04` at `$C45E` in the slot page, the bug the Memory Expansion Card
Tech Note #1 says was fixed in the //c ROM.

## CPLD checklist (from the above)

- `$C0C0`/`$C0C1`: 8-bit read/write; `$C0C2`: write D3–D0, read `{1111, A19..A16}`.
- `$C0C3`: SRAM read or write at the pointer, then pointer + 1 (20-bit, wraps to 0) at the end of the access.
- Reads of `$C0C0`–`$C0C2` have no side effects; writes load only the register written.
- Registers disabled after /RESET until any `$C4xx` access (Tech Note #1).
- Fit the full 1 MB, or block aliasing above the fitted size.
