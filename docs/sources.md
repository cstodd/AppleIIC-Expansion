# Sources and findings

What has been found, where it is, and what it tells us. Asimov mirror base:
`https://mirrors.apple2.org.za/ftp.apple.asimov.net/` (the mirror's `site_files.txt` lists every file).

## Found and read

| Document | Path on the mirror | Key facts |
|---|---|---|
| *Apple IIc Technical Reference Manual*, 2nd ed. | `documentation/hardware/machines/Apple IIc Technical Reference Manual (alt).pdf` | J13 pinout (Fig. 11-43, p. 291) and schematic 050-0180 (Fig. 11-44, pp. 292–296). See [j13-memory-expansion-connector.md](j13-memory-expansion-connector.md) |
| Apple IIc Technical Note #5, *Memory Expansion on the Apple IIc* (Oct 1986, rev. Nov 1988) | `documentation/misc/APPLE II Technical Notes-Apple IIc.zip` | "The memory expansion card for the IIc is **functionally identical to the card for the IIe**, but the IIc card 'lives' in slot 4 and the firmware is included in the ROM on the IIc motherboard." Software detects the card by asking how many blocks it has, not by ID bytes |
| Apple IIc Technical Note #7, *Existing Versions* | same zip | ROM `$03` = first memory-expandable IIc. ROM `$04` adds bug fixes and "returns correct information when the Memory Expansion Card is not present". **IIc Plus (`$05`)** "contains a Memory Expansion Card connector" but "**not** the original IIc Memory Expansion Card" |
| Memory Expansion Card Technical Note #1, *Q&A* (1986, rev. 1988) | `documentation/misc/APPLE II Technical Notes-Memory Expansion Card_2523001 Q&A.pdf` | **RESET turns the card's registers off until an access to `$Cn00`**: "to optimize speed and the number of pins and logic on the **custom gate array**. The boot scan hits `$Cn00` anyway". Firmware screen holes: `$478+n` = number of 64K banks, `$4F8+n` = power-up byte (`$A5`), `$578+n`. |
| *Apple IIc Memory Expansion Card Technical Reference* (APDA A2G0047, 1987). PDF supplied by the project owner, not on the mirror | not committed (copyright) | 20-bit pointer, register map, load order, gate array + PAL 16L8 + 74HCT245, refresh in PH1, connector pinout, card layout, ROM map. See [card-reference.md](card-reference.md) |
| *Apple IIc Memory Expansion Card Dealer Installation Instructions* (030-1379-A) | `documentation/hardware/storage/memory/` | See "Mechanical and service" below |
| *Apple IIe Memory Expansion Card Owner's Guide* (file named "Apple IIc Memory Card Owners Guide.pdf"), *Apple II Memory Expansion Card User's Manual* | `documentation/hardware/storage/memory/` | User level only. No register or hardware detail (the II manual's Appendix B points programmers to the firmware calls) |
| IIc Plus Technical Reference, Appendix H, *Controlling the Apple IIc Plus Accelerator* | `documentation/hardware/machines/IIcPlus_Zip_HWRefAppendixH.pdf` | The cache glue gate array (CGGA) has a speed bit per port. **Port 4 defaults to fast** (with ports 3 and 7); ports 1, 2, 5 and 6, the speaker and the paddles default to 1 MHz |

### What this answers

- **Why J13 carries A8–A15 and not just `$C0Cx` decode inputs:** the card has to see the `$C400` access that re-enables its
  registers after RESET (Q&A #1). That matches /RESET being on the connector. Our CPLD needs: /RESET → registers
  disabled; any access to `$C4xx` → enabled.
- **Behaviour reference:** because the //c card is "functionally identical" to the IIe card (Apple II Memory Expansion
  Card, "Slinky"), the IIe card's documentation and schematic describe the //c card's register behaviour too.
- **//c Plus:** has the connector, but Apple's own //c card doesn't work there. The CGGA running port 4 fast by default
  is a likely reason (the card's I/O or timing at 4 MHz). That's a lead, not confirmed.

### Mechanical and service (dealer instructions)

- Logic board model **A2S4100** has the connector. **A2S4000** needs a logic board upgrade first.
- The card mounts **component side down** on **four nylon stand-offs** pressed into holes in the logic board, with the
  connector on the right-hand side; the keyboard goes back on top. Fig. 1 shows the expansion connector at the
  right-front of the logic board.
- The card has 32 × 256K DRAMs in four banks of eight (256K → 1 MB, fill banks 1 → 4) plus the controller IC.
- Built-in diagnostic: `CALL -151`, then `C40AG`. It shows the card size and loops a RAM test; errors are reported as
  `DATA ERROR XXYYYY-ZZ` / `ADDRESS ERROR …`.
- Rubber feet stuck to the bottom of the keyboard sit above the card. Memory ID stickers: 384K/640K/896K/1.15 MB.

## Applefritter forum threads (www.applefritter.com)

Forum posts are second-hand. Treat them as leads, not facts.

| Thread | Key points |
|---|---|
| [Memory For Apple IIc Plus](https://www.applefritter.com/node/17721) (2006/2013) | The `C40AG` card test also exists on the //c+. A //c+ **with no card** reported `CARD SIZE = 1 MEG` and then `ADDRESS ERROR`. Likely cause: with nothing on the bus, the `$A5` the size test just wrote is still sitting on the data lines when it reads back, so every 64K step "passes". See firmware-analysis.md |
| [Apple //c 1MB Memory Expansion, Apple //c+ …](https://www.applefritter.com/content/updatedapple-c-1mb-memory-expansion-apple-c-sold-separately-or-together) (2013) | Seller says Apple's //c card "will work … in its later look-alike (model#4100) and in the //c+ (model#4500)", and in a model 4000 only with an updated motherboard. **This contradicts Apple IIc Tech Note #7** ("not the original IIc Memory Expansion Card"). Unresolved; possibly a card revision difference |
| [FS: CT-RAM-C memory expansion for IIc/IIc+](https://www.applefritter.com/node/23539) (2008) | Chinook Technologies C-RAM: for the memory-expandable //c (ROM 3) or //c+, "just attach it to the connector", "tested to work in a IIc+ system". 256K, expandable to 1 MB |
| [WTB: AE RAM Express for Apple IIc & IIc+](https://www.applefritter.com/content/closed-wtb-ae-ram-express-apple-iic-iic) (2017) | AE RAM Express is for //c and //c+. A modern replacement is sold by a2heaven (product 144 in its web shop) |
| [The CDrive Card](https://www.applefritter.com/node/24740), posts by Jeff Mazur (2010–2014) | "When the //c+ came out, **with its memory card that plugged in and bridged over the CPU** …" (so CPU-socket add-ons needed a ribbon cable). Apple's code name for "their //c+ memory card" was **"Tiger Tail Plus"**. Mazur designed the **Proclock //c and Superclock //c** clock boards (fitted in the CPU socket, "upside-down") |
| [Fun with the Apple //c memory expansion card](https://www.applefritter.com/node/8001) (2005) | The *IIc Technical Reference*, 2nd ed., covers the //c+. The *IIc Memory Card Reference* exists (we now have it) |

## Available, not yet examined

| What | Path on the mirror | Why |
|---|---|---|
| IIe/IIc firmware source (ProDOS `.2mg` images) | `emulators/rom_images/Apple IIe IIc Firmware SourceCode.zip` | The `$C400` RAM-disk driver shows register use and size detection. Check whether this is ROM `$03`/`$04` or earlier |
| IIc ROM images | `emulators/rom_images/apple_iic_rom.zip` | Checked: it's the UniDisk 3.5 ROM (`$00`), not `$03`/`$04`. The Spanish IIc ROM is the original (`$FF`) |
| //c ROM `$03` and ROM `$04` (32 KB dumps) | supplied by the project owner, not committed (copyright) | Card driver confirmed identical to 341-0344A; ROM `$04` STATUS fix. See [firmware-analysis.md](firmware-analysis.md) |
| Apple II Memory Expansion Card ROM 341-0344A | `emulators/rom_images/Apple II Memory Expansion ROM_341-0344A.bin` | Disassembled: see [firmware-analysis.md](firmware-analysis.md) |
| Apple IIc Plus Technical Look (HyperCard) | `documentation/hardware/misc/Apple IIc+ Technical Look (Mac Hypercard).zip` | IIc+ memory connector notes |
| *Apple IIc Memory Expansion Card Owner's Guide* (scanned, no text layer) | `documentation/hardware/storage/memory/` | Probably user level, but check for a technical appendix |
| Zip Chip manuals and patent | `documentation/hardware/accelerators/` | How a Zip-style accelerator handles slot I/O (the CGGA is Zip-derived) |

## Not found yet / blocked

| What | Where it should be | Status |
|---|---|---|
| Schematic of the Apple II Memory Expansion Card (670-0024) | applelogic.org (Peripheral Cards page) | Host allowed, but its HTTPS certificate chain fails verification through the proxy |
| wiki.apple2.org *RAM Cards* (//c vs //c+ PAL notes, AE / Sequential cards) | wiki.apple2.org | Host allowed, but the gateway can't connect to it (down or no HTTPS) |
| Archived copies of the two pages above | web.archive.org | Host allowed, but the connection is dropped during the TLS handshake (outside the environment settings) |
