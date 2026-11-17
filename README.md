# AppleIIC-Expansion

Apple //c variant of the [Laser 128EX memory + clock card](https://github.com/cstodd/Laser128EX-Expansion)
(SRAM in place of DRAM, a CPLD, DS3234 RTC).

## Status: research only

No hardware design yet. Building waits until the Laser card has been produced and validated in a real
machine. Until then this repo collects research: connector, timing, firmware and mechanical facts about the
memory-expandable //c (ROM `$03`/`$04`) and the //c+.

## Contents

| File | What |
|---|---|
| [docs/j13-memory-expansion-connector.md](docs/j13-memory-expansion-connector.md) | //c J13 (2×17) pinout from the IIc Technical Reference Manual and schematic 050-0180, compared with the Laser's J13, and what it means for the card |
| [docs/sources.md](docs/sources.md) | Documents found (and where), key facts from each, and what's still missing or blocked |
| [docs/open-questions.md](docs/open-questions.md) | Things still to find out before a design starts |
| [docs/ref/](docs/ref/) | Crops of the source pages |

## Key finding so far

The Laser's J13 is a **DRAM port**: the motherboard gate arrays hold the address registers and generate
RAS/CAS/refresh. The //c J13 is a **CPU-bus port** (D0–7, A0/A1/A4–A15, R/W, PH0, Q3, 7M, /RESET): the card
must decode `$C0C0–$C0C3`, hold the auto-incrementing address register and drive the data bus itself. The
SRAM/RTC approach carries over; the CPLD logic has to be redesigned, not ported.
