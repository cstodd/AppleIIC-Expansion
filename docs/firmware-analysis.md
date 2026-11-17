# Card firmware: what it needs from the hardware

## Which ROM

The analysis started with the **Apple II Memory Expansion Card ROM 341-0344A**
(`emulators/rom_images/Apple II Memory Expansion ROM_341-0344A.bin` on the Asimov mirror), the IIe/II version of the card,
because no //c ROM `$03`/`$04` was on the mirror. It was then **confirmed against //c ROM `$03` and ROM `$04` dumps**
supplied by the project owner (32 KB each, not committed; Apple copyright). See "The //c ROMs" below.

ROM layout (4 KB): `$000` copyright text ("This wonderful, wonderful rom is brought to you by Richard Williams …"),
`$100`–`$7FF` one `$Cn00` page per slot, `$800`–`$FFF` the shared `$C800`–`$CFFF` code. The firmware reaches the
registers as `$BFF8,X`–`$BFFB,X` with X = `$80 + slot × 16` (`$C8` for slot 4, which gives `$C0C0`–`$C0C3`).
Disassembled with py65. Addresses below are `$C800`-space addresses in that ROM.

## The //c ROMs (`$03` and `$04`)

Both are 32 KB: the main bank at offset `$0000` and the auxiliary bank at `$4000`, each mapped to `$C000`–`$FFFF`.
Identification: main `$FBB3 = $06`, `$FBBF = $03` / `$04`; `$C401 = $20`, `$C403 = $00`, `$C405 = $03`, `$C407 = $00`
(the card ID bytes in the card manual). `$C4FF = $4E`.

- **Same driver code as the IIe card ROM.** The card driver is in the auxiliary bank `$D800`–`$DFFF`. It sets
  Y = `$C4` and X = `$C8` (`$D8EF`–`$D8F4` in ROM `$03`) and uses the same `$BFF8,X`–`$BFFB,X` accesses. The size test
  (ROM `$04` `$D99F`–`$D9DC`) and the register diagnostic (ROM `$04` `$DC4E`–`$DC71`, including the `ORA #$F0` check of the
  high register and `DEC $BFF8,X`) are byte-for-byte the code at `$CAB1` and `$CD3E` in 341-0344A. So the requirements
  below apply to the //c unchanged.
- **No absolute `$C0C0`–`$C0C3` accesses anywhere**, and nothing touches `$C0C4`–`$C0CF`.
- **ROM `$03` → `$04` change to the card driver:** only the SmartPort STATUS reply at `$D865`. ROM `$03` always returns
  status `$01` (device online). ROM `$04` returns `$01` only if the size test found memory (`$03B8+slot` ≠ 0), which matches
  "Firmware returns correct information when the Memory Expansion Card is not present" (IIc Tech Note #7). The
  remaining differences in `$D865`–`$DB62` are the same code moved 10 bytes (12 operand bytes adjusted by `$0A`).
  The other ROM `$04` changes (`$C8EF`, `$CFFF`, `$FD02`–`$FD0B`, `$FD4C`, `$FE74` main; `$C1FE`, `$C67E`–`$C6BD`, `$D1F2` aux)
  are outside the card driver (keyboard buffering fix etc.).
- **What this means for our card:** a //c with ROM `$03` reports a RAM disk even with no memory (it's a ROM bug, not
  the card's fault). Under both ROMs, the card is detected purely by the size test, so our card only has to pass that.

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
