# DMX Interface Dev Board v2 — Altium hardware

Production hardware for the **DMX Interface Rev 2**: a W6300-EVB-Pico2 (RP2350 +
W6300) carrier that takes Art-Net / sACN / wired DMX / USB in and drives eight
WS2812 strings, split across **two boards**. This repository holds the Altium
sources and generated manufacturing outputs. The firmware, design record, review
history and bring-up checklist live in `dmx_interface_firmware`.

| Board | Project | Size / stack | Job |
|---|---|---|---|
| **DMX Interface DB v2** (logic board) | `DMX Board/DMX Interface DB v2.PrjPcb` | 110 × 81.5 mm, 4-layer JLC04162H-7628, 1 oz outer / 0.5 oz inner, controlled impedance (USB 90 Ω, DMX 120 Ω) | Module, isolated DMX, FTDI USB-DMX port, UI, power tree, 8 × PTC-fused aux/data outputs |
| **LED Power Board** | `LED Power Board/LED Power Board.PrjPcb` | 85 × 90 mm, 4-layer, **2 oz outer** / 0.5 oz inner | Strip power only: screw-terminal input, 8 × 5 A MINI blade fuses, 470 µF per output, pluggable 5.08 mm outputs |

## Folder map

```
DMX Board/
  DMX Interface DB v2.PrjPcb      Altium project (7 sheets, harness-connected)
  Block Diagram.SchDoc            top sheet: module ↔ UI/LED/FTDI/DMX/Power, PSU options noted
  RP2350.SchDoc                   W6300-EVB-Pico2 socket (U1), RUN header, pin map
  WS2812.SchDoc                   SN74ACT245 level shift, 330 R series, 8 × PTSM 3-pin outputs with 3 A PTCs
  DMX.SchDoc                      isolated RS-485 front end (THVD1400, TLP2368 ×3, 1S7BE DC-DC), XLR-5 in/out
  FTDI.SchDoc                     FT232RNL self-powered USB-DMX port, USB-C, USBLC6, RESET# gate
  Power.SchDoc                    V_LED input (J26), F9 → L2/U6 → 5V_LOGIC → D2 → V_SYS → TLV75733, USB power paths
  UI.SchDoc                       TCA9555 (0x20), PCA9633 (0x62), M24C02 (0x56), TFT/button/PWM headers
  *.Harness                       harness definitions for the block diagram
  DMX Interface.PcbDoc            layout
  DMX Interface_Assy.PCBDwf, _FAB.PCBDwf   assembly / fabrication drawings
  DMX Interface.BomDoc            ActiveBOM (variant "Production")
  Job1.OutJob                     output job → "Project Outputs for DMX Interface DB v2/"
  Project Outputs for DMX Interface DB v2/
    BOM/  Gerber NC Drill/  OrCadPCB2Netlist/  PCB Print/  PCBDrawing/
    Report Board Stack/  Schematic Print/  Status Report.Txt
  review.md, DMX_Interface_Rev1_Design_Review.md   the Rev 1 (Nucleo + RP2040) review — identical copies, historical

LED Power Board/
  LED Power Board.PrjPcb, Sheet1.SchDoc, PCB1.PcbDoc
  Assembly Drawing.PCBDwf, Fabrication Drawing.PCBDwf, ActiveBOM.BomDoc
  Job1.OutJob (plus Assembly*.OutJob / Fabrication.OutJob)
  Project Outputs/                same structure as above

DMX_Interface_Multiboard_Project/   Altium multi-board project tying the two boards together (Assembly1.MbaDoc)
Panel Templates/                    only ECO logs — ignored
```

`.gitignore` excludes `__Previews/`, `History/` and every `Project Logs*/`
folder. The `Project Outputs` folders **are** tracked: they are what gets sent
to the fab and what the reviews were run against.

## Electrical summary

**Sources for `V_LED` on the logic board** (three, any one boots the logic):

1. **J26** (Molex Micro-Fit+ 206832-0801, 4 × V_LED + 4 × GND) from the PSU /
   LED Power Board — full power. Strip current stays on the power board; the
   logic board carries only its logic branch and eight 3 A aux/data feeds.
2. **FTDI USB-C (J2) with a plain 5 V brick** — low-power installs. U14
   TPS2553DBVR-1 USB power switch: current limit ≈ 1.45 A (R37 18k),
   reverse-voltage cut-off, EN from TCA9555 **P04** (R38 pull-down, off until
   firmware drives it high), FAULT to **P07** (R35 pull-up), soft start, then F2
   0ZCF0200FF2C 2 A PTC. **5 V build only** — F2 is DNP on 12/24 V builds.
3. **Module USB** — bench feed only: D3 B360A → F18 0.5 A PTC → V_LED.

The brick also boots the logic directly on any build through F1 (1.1 A PTC)
→ D4 → 5V_LOGIC. VBUS present-detect dividers (10k/20k) go to TCA9555 P05
(FTDI) and P06 (module). Trim the PSU to ≥ 5.2 V so it wins when both are
attached.

**Logic branch:** V_LED → F9 1.1 A PTC → L2 ferrite (5 V build) *or* U6
OKI-78SR-5/1.5 (12/24 V builds) → 5V_LOGIC (C17 47 µF SP-Cap) → D2 B340A →
V_SYS → module buck and U9 TLV75733 (3V3, enabled by module 3V3_OUT through
JP3/R26/R31 so the carrier rail never rises before the RP2350 IOVDD). L3 →
5V_LVL_SHIFT for U10.

**Build variants (one voltage per board, silk-marked):**

| | 5 V | 12 V | 24 V |
|---|---|---|---|
| L2 ferrite / U6 OKI | L2 fitted, U6 DNP | U6 fitted, L2 DNP | U6 fitted, L2 DNP |
| D1 TVS | SMDJ5.0A | SMDJ12A | SMDJ26A |
| F2 (brick path) | fitted | DNP | DNP |
| F10–F17 output PTCs | 0ZCF0300BF2C | 0ZCF0300BF2C | 0ZCF0150FF2C (≥ 30 V) |
| C2 | 50 V part on all builds | | |

**LED Power Board:** J3 (OSTYK, both positions V_LED) and J6 (both GND) →
eight Keystone 3544-2 MINI holders with 5 A 0297005.WXNV fuses → 470 µF per
output → Würth 691319510004 headers, mating plugs 691351500002. R1 10k bleed.
There is **no fuse, TVS or reverse protection on this board**: a 40–50 A
MIDI/ANL fuse in the PSU harness is required, and the only clamp is D1 on the
logic board through the J26 cable. Rated 8 × 5 A = 40 A through the board;
input band ≈ 15 mm of 2 oz copper, each output ≈ 8.5 mm.

**Isolation:** the DMX side (THVD1400, XLR shells excepted) sits on 3V3ISO /
GNDISO from PS1 (GAPTEC 1S7BE, 1 kV); plane split on both inner layers,
optocouplers straddle the gap; XLR shells bond to CHGND via PTH3 / R4 (0 Ω
fitted) / C1 (DNP).
