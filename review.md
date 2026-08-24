# DMX Interface Rev 1 — Full Design Review

**Scope:** All 22 schematic sheets plus embedded netlist and BOM from `DMX_Interface_DB.pdf` (carrier board with NUCLEO-H563ZI + RP2040 Pico, isolated DMX front end, drive-select matrix, and SmartLED / Low-Power / High-Power output modules). Connectivity claims below were traced through the netlist embedded in the PDF, not just the drawings. Where the PDF extraction was ambiguous, the item is explicitly marked **VERIFY**.

**Verdict:** The architecture is sound and most of the board checks out — connector pin maps between carrier and modules are consistent, the isolation barrier is crossed correctly, I²C pull-up topology is clean, and the keying scheme prevents most mis-mating. However, this review found **several new hard errors on the RP2040 pin assignment and the drive-select matrix** that were not in your existing notes, plus two **absolute-maximum-rating hazards** (TLC59116 at 24 V, and 5 V into RP2040 GPIOs via the SmartLED level shifter). Details and a prioritized Rev 2 change list follow.

---

## 1. Critical — will not work as drawn

### 1.1 RP2040 DMX_I2C pins cannot map to a hardware I²C controller ⚠️ NEW

Netlist (sheet 10): `DMX_I2C.ALERT#` = GP4 (pin 6), `DMX_I2C.SDA` = **GP5** (pin 7), `DMX_I2C.SCL` = **GP6** (pin 9), annotated "I2C1".

On the RP2040, SDA is only available on even-numbered GPIOs and SCL on odd-numbered ones:

| GPIO | I²C capability |
|---|---|
| GP4 | I2C0 **SDA** |
| GP5 | I2C0 **SCL** |
| GP6 | I2C1 **SDA** |
| GP7 | I2C1 **SCL** |

As wired, SDA sits on an SCL-only pin and SCL on an SDA-only pin, and even relabeling in firmware leaves them on *different* controllers (GP5→I2C0, GP6→I2C1). **No hardware I²C configuration exists for this net assignment.** Since DMX_I2C is the STM32↔RP2040 inter-processor link and the RP2040 will most likely be the slave/peer, falling back to PIO I²C slave is possible but painful.

**Fix (Rev 2):** move `DMX_I2C.SDA` → GP6 and `DMX_I2C.SCL` → GP7 (valid I2C1 pair), and relocate `DMX_SPI.CS1` off GP7 (see 1.2 — GP9 is free and solves both problems). Alternatively SDA=GP4 / SCL=GP5 (I2C0) with ALERT# moved to GP6.

### 1.2 DMX_SPI.CS1 on GP7 is not a hardware SPI chip-select ⚠️ NEW

The populated solder-bridge configuration (SB2, SB5 ON; SB4, SB6 DNP) routes `DMX_SPI.MOSI` → GP8 (**SPI1 RX**) and `DMX_SPI.MISO` → GP11 (**SPI1 TX**), i.e. the RP2040 is wired as the **SPI slave** with the STM32 (SPI6) as master. GP10 = SPI1 SCK is correct. But `DMX_SPI.CS1` lands on **GP7**, which is SPI0 TX — not a CSn pin. The PL022 in slave mode requires CS on a hardware CSn pin (SPI1 CSn = GP9 or GP13). GP13 is taken by `RP2040_SCL`; **GP9 (module pin 12) is currently unconnected** and is the natural fix.

Until fixed, hardware SPI slave mode cannot frame transactions; you'd need a PIO SPI slave.

### 1.3 Drive-select matrix crosses RP_LED3.Data and RP_LED4.Data ⚠️ NEW

On the Drive Select sheet, the RP2040-side (pin 3) nets of the LED data jumpers are crossed relative to the clock jumpers:

| Jumper | Module net (pin 2) | STM32 side (pin 1) | RP2040 side (pin 3) |
|---|---|---|---|
| JP17 | LED3_CLK | STM32_LED3.Clock | RP_LED3.Clock ✓ |
| JP18 | LED3_DAT | STM32_LED3.Data | **RP_LED4.Data ✗** |
| JP19 | LED4_CLK | STM32_LED4.Clock | RP_LED4.Clock ✓ |
| JP20 | LED4_DAT | STM32_LED4.Data | **RP_LED3.Data ✗** |

JP12/13/15/16 (channels 1–2) are consistent. With the RP2040 selected as LED driver, channel 3's clock pairs with channel 4's data and vice versa. The RP2040 can absorb this in firmware (PIO pin mapping is free), but it will silently corrupt any code that assumes the net names — and the same crossover is reflected on the block-diagram sheet, so it propagates into documentation. Swap the pin-3 nets of JP18/JP20.

Note also: with the default build, the RP2040 cannot drive channel 4 at all — `RP_LED4.Clock`/`RP_LED4.Data` reach GP10/GP11 only through **DNP** bridges SB8/SB3, which are shared with the DMX_SPI pins. That's a deliberate either/or, but worth stating in the docs.

### 1.4 SmartLED DIR net anomalies — **VERIFY in Altium** ⚠️ NEW

The extracted netlist shows the SN74ACT245 `DIR` node (U10 pin 1, JP21 pin 2) also containing **J18 pin 3 and J20 pin 3**, and possibly **U13 pin 1 (EEPROM E0) plus JP24 pin 2**. Two problems if real:

1. J10/J13 carry V_LED on pins 1 *and* 3, but J18/J20 carry V_LED only on pin 1 — pin 3 lands on the DIR logic node. A strip plugged into J18/J20 expecting power on pin 3 would fight or glitch the transceiver direction. There's no evident reason for power connectors to carry DIR.
2. If E0/JP24 truly share the node: JP21 selects 5V_LVL_SHIFT/GND and JP24 selects 3V3/GND on the *same net*. Both at default (1-2) would short **5 V to GND**, and any high state puts 5 V on the M24C02's E0 pin (abs max VCC + 1 V = 4.3 V). The EEPROM address table ("default 0x56", E0=0) also stops being independently settable.

PDF net extraction can merge adjacent nets, so treat this as a must-verify rather than a confirmed fault — but the J18/J20 pin-3 assignment is suspicious even in the most charitable reading.

### 1.5 3V3 → RP_VSYS ideal diode (U14) non-functional — known

Confirmed: AP74700 requires ~4 V minimum anode voltage, so the 3V3 ORing leg into RP_VSYS is dead, as your sheet-1 note says. TPS2116 / LM66100 / Schottky-OR are the right class of fix. (RP_VSYS is happy from 1.8 V, so even a Schottky's drop from 3.3 V is fine.)

---

## 2. Damage and ratings risks

### 2.1 TLC59116 outputs are 17 V absolute max — 24 V supplies will kill the LP module ⚠️ NEW

The carrier explicitly supports 24 V Meanwell bricks (sheet 1 lists LSP-160-24T / UHP-200-24), and V_LED is bussed unchanged to the Low-Power module's J37 common-anode connectors. The TLC59116's OUTx pins are rated **17 V absolute maximum** in the off state; the sheet note itself says "(12V 3W)". Nothing electrically or mechanically prevents installing the LP module with a 24 V supply — the off-state sink pins will see 24 V and exceed abs max. At minimum, silkscreen + docs must restrict the LP module to ≤12 V (≤15 V with margin); a better Rev 2 fix is a module-local regulator/clamp or a keying/interlock convention per supply voltage.

### 2.2 Reverse direction on U10 drives 5 V into non-5V-tolerant RP2040 pins ⚠️ NEW

The SN74ACT245 runs at `5V_LVL_SHIFT`. JP21 in the DIR-low position (B→A, "read data back from strips") turns the A-side pins into ~5 V CMOS outputs driving straight through the module connector and drive-select into the MCU pins with no series resistance. STM32H5 FT pins tolerate this; **RP2040 GPIOs do not (max VDDIO + 0.3 V ≈ 3.6 V)**. If the reverse direction is ever a supported mode, add series resistors and/or clamp, or drop U10's VCC to 3.3 V for that use case. If it's not a supported mode, remove the JP21 GND option or document it as forbidden. Also note DIR is common to all 8 bits, and a direction flip while the MCU still drives its outputs creates transceiver-vs-MCU contention.

### 2.3 TVS and reverse-polarity protection on V_LED — known, plus one addition

SMAJ26A (D1/D2/D7) at 26 V standoff over a 24 V rail leaves ~2 V margin; supply turn-on overshoot can bias the TVS into leakage/heating. SMAJ28A/30A recommended. Additional point: the TVS conducts hard on reverse polarity but there is **no fuse or series element on any V_LED input** (J8/J30/J32) — a reversed 200 W brick will destroy the TVS and then the board. A polyfuse or input FET per module power connector is cheap insurance even on a dev board.

### 2.4 TLC59116 Rext floor — known, confirmed

R57 = 120 Ω fixed + VR1 (3306P, spec'd 30 Ω–1 kΩ but wiper can approach 0 Ω). At wiper ≈ 0, Rext → 120 Ω → ~156 mA/channel, above the 120 mA device limit. Raise R57 to ≥160 Ω so the minimum is safe regardless of trimmer position.

### 2.5 TLC59116 thermal budget — worth a calculation ⚠️ NEW

As a linear sink, per-channel dissipation is (V_LED − V_string) × I_OUT. At 12 V with short strings and ~117 mA, several channels active can put multiple watts into one TSSOP28. Run the worst-case sum for your intended loads; you may want to mandate series resistors or higher-Vf strings to move dissipation off-chip.

### 2.6 Jumper/e-fuse configuration hazards ⚠️ NEW

- **JP14 in the 5 V position with a 12/24 V supply** puts V_LED directly into U9 (LS0504, 6 V abs max) and onto `5V_LVL_SHIFT` → U10 and the strips. One wrong solder bridge = dead module. Consider an auto-select (comparator or wide-Vin buck) instead of a bridge, or at least prominent silkscreen.
- F1/F2 PTCs are 6 V parts. They never see >5 V while the AP74700s block correctly, but a shorted Q1/Q2 exposes them to 24 V, beyond their interrupt rating. Acceptable dev-board risk; noting for awareness.
- JP1/JP22/JP23 populated *with* an external brick present is actually safe (the ideal diodes block reverse into USB) — the silkscreen warnings are belt-and-braces, which is fine.

### 2.7 Multiple modules, multiple power inputs ⚠️ NEW

Each module carries its own brick input (J8, J30, J32) and all common onto the same V_LED through the carrier. Plugging *different* bricks into two installed modules (e.g., 5 V and 24 V) back-drives one supply from the other. Document "one brick per system," or add ORing/blocking per module input in a future rev.

---

## 3. Functional and performance findings

### 3.1 RP2040 DMX UART pins require PIO ⚠️ NEW (likely intentional — confirm)

`DMX.RX` = GP0, `DMX.EN` = GP2, `DMX.TX` = GP3. GP0 is UART0 **TX**-capable (not RX), and GP3 is UART0 CTS (not TX) — so the hardware UART cannot service DMX on these pins in either direction. Since DMX break/MAB timing is normally done with PIO on the RP2040 anyway (e.g., pico-dmx), this is fine *if intended* — but if you ever wanted the hardware UART as a fallback, RX belongs on GP1/GP13/GP17 and TX on GP0/GP4/GP8/GP12/GP16. Worth a schematic note either way.

### 3.2 Opto LED drive margin (RX path) — known, quantified

U8 input: (3.3 V − ~1.4 V Vf − VOL) / 470 Ω ≈ **3.8 mA**. Check this against the TLP2368 threshold input current across temperature; the recommended operating IF for these photo-IC couplers is typically ≥6.5 mA. Dropping R13 to ~270 Ω would put you comfortably in the recommended range (THVD1400 RO can sink it).

### 3.3 R-78K3.3-1.0 minimum input — known

With a 5 V brick, V_LED minus L2 DCR and ripple sits right at the R-78K3.3's ~4.75 V minimum. Fine at 12/24 V. If 5 V operation matters, consider a lower-Vin-min module or LDO bypass path.

### 3.4 STM32 hardware PWM availability for the HP module — refined ⚠️ NEW DETAIL

HP module PWM sources (via J17_H): PWM1 = LED1_DAT or LED1_CLK (JP26, default CLK), PWM2 = LED2_DAT, PWM3 = LED3_DAT, PWM4 = LED4_DAT. Mapping to STM32 pins:

| Signal | STM32 pin | SPI AF | Timer channel |
|---|---|---|---|
| LED1_CLK | PA5 | SPI1_SCK | **TIM2_CH1 ✓** (JP26 default — good) |
| LED1_DAT | PD7 | SPI1_MOSI | none (verify) |
| LED2_DAT | PC3 | SPI2_MOSI | none (verify) |
| LED3_DAT | PB2 | SPI3_MOSI | none (verify) |
| LED4_DAT | PE14 | SPI4_MOSI | **TIM1_CH4 ✓** |
| LED2_CLK | PB10 | SPI2_SCK | TIM2_CH3 ✓ |
| LED4_CLK | PE12 | SPI4_SCK | TIM1_CH3N ✓ |

So on the STM32, only PWM1 (via JP26→CLK) and PWM4 get hardware PWM; PWM2/PWM3 fall back to software toggling. If HP dimming from the STM32 matters, add JP26-style CLK/DAT select jumpers to PWM2 and PWM4 as well (LED2_CLK = PB10/TIM2_CH3 and LED4_CLK = PE12/TIM1_CH3N are usable), and verify the "none" entries against the H563 datasheet AF table. RP2040 side is a non-issue (PWM/PIO on any pin).

### 3.5 Series resistors on DAT only

R17–R20 (330 Ω, provisional) are in the data lines only; clocks go straight from U10 to the LED connectors. For APA102-class strips on any cable length, matching series R on CLK helps edges/ringing just as much. Add provisional footprints on the four CLK lines.

### 3.6 Dual 3V3 sources / Nucleo contention — known, with one addition

The carrier 3V3 (U11) ties to the Nucleo's 3V3 pin **through the unannotated `R?` 0 Ω** (see §6) — so the contention point is at least depopulatable. Document the required Nucleo solder-bridge configuration for external-3V3 powering, and give R? a real designator so it shows up in assembly docs.

### 3.7 No fusing on LED power outputs ⚠️ NEW

J10/J13/J18/J20, LED1–4, J37_PWMx, and J33 all distribute up to a 200 W brick with no per-output protection. A shorted strip cable turns the wiring into the fuse. Recommend polyfuses or fuse footprints per output group in Rev 2.

### 3.8 Module EEPROM addressing — known

U13/U15/U21 all default to 0x56 (E1/E2 strapped high via 0 Ω, E0 low). Fine with one module per connector position, but strapping each module *type* to a distinct 0x5x enables type detection by address probe. TLC59116 at 0x60 (+ all-call/sub-call/SWRST in the 0x68–0x70 region) doesn't collide with 0x56 — bus is clean.

---

## 4. Connector and pinout verification

Traced pin-for-pin through the netlist:

| Interface | Result |
|---|---|
| J7 (carrier, power) ↔ J9_S/L/H (modules) | ✓ Match: 1,2 = V_LED; 3,4 = GND; 5,6 = 3V3; 7,8 = GND |
| J11 (carrier, control) ↔ J12_S/L/H | ✓ Match: 1=ALERT#, 2=SCL, 3=RESET#, 4=SDA, 5=CS1, 6=SCK, 7=CS2, 8=MISO, 9=CS3, 10=MOSI |
| J16 (carrier, LED) ↔ J17_S/L/H | ✓ Match: odd pins = DATn, even pins = CLKn, n = 1..4 |
| HP PWM mapping | ✓ Consistent: J17_H pins 3/5/7 (= LED2/3/4_DAT) → PWM2/3/4; no odd/even swap found (resolves the earlier suspicion) |
| Micro-Fit+ keying (A-key LED1–4 vs D-key J10/13/18/20) | ✓ Prevents signal/power mis-mating |
| DMX XLR genders | ✓ J5 male = input, J15 female = output — matches DMX512 convention |
| SmartLED channel numbering | Confirmed mechanism of your silkscreen note: U10 A1/A2 carry **LED4**_DAT/CLK → B1/B2 → connector "LED1", etc. Pairing (DAT+CLK) is preserved per connector, so it's purely a labeling reversal — connector LEDn carries channel (5−n). Fix silkscreen or swap A-side order in Rev 2. |
| DMX pass-through | ✓ Pins 2/3 (data) and 4/5 (aux pair) bused straight between J5 and J15; JP28 + R15/R16 = switchable 120 Ω termination |
| J5/J15 rendered with "X" | **VERIFY** — confirm placeholder graphic vs. unintended DNP (carried over from previous notes) |
| Board-to-board mating orientation | **VERIFY mechanically** — dual-row RA female (carrier) to dual-row RA male (module) at facing edges should keep pin 1↔pin 1, but row-mirroring is the classic failure mode for this construction; check in 3D/first article |

---

## 5. Items verified OK

- **Isolation barrier crossing:** U3/U7 (TX/EN) LED-side on logic 3V3, output-side on 3V3ISO; U8 (RX) reversed — all correct. PS1 (1S7BE, 1 W) budget ≈ 50–60 mA worst case on 3V3ISO — ample.
- **RS-485 fail-safe bias:** 680 Ω / 120 Ω / 680 Ω gives ≈ 270 mV differential idle — above the 200 mV threshold, plus THVD1400's internal fail-safe. Good.
- **DE//RE tied and driven together** via U7 — correct for a DMX transmitter/transceiver; no local echo, which is fine.
- **CHGND scheme:** XLR shells → CHGND → R1 (0 Ω) → logic GND; XLR pin 1 → GNDISO. Reasonable; keep R1 as the tuning point.
- **I²C pull-up topology:** exactly one 2.2 kΩ set per physical bus segment on each of DMX_I2C, OUTPUT_I2C (STM32 side) / RP2040 I²C (RP side), DISPLAY_I2C — no double termination through the drive-select. 2.2 k at 3.3 V = 1.5 mA sink, fine for Fm.
- **STM32 I²C assignments spot-checked:** I2C1 = PB8/PB9 (DMX) ✓; I2C2 = PF1/PF0 (display) ✓; I2C4_SDA = PF15 and I2C4_SMBA = PF13 (output bus) ✓ — SCL should therefore be PF14; the PDF extraction was ambiguous on that one pin, quick check recommended.
- **RP2040 OUTPUT-side I²C:** GP12/GP13 = valid I2C0 SDA/SCL ✓ (which is what makes the DMX_I2C error in §1.1 stand out).
- **Drive-select SPI/I²C/CLK jumpers** (JP2–JP17, JP19, JP29–JP34): all pin-1/pin-3 source nets consistent; only JP18/JP20 are crossed.
- **RP2040 reset:** STM32 PG1 + J14 jumper onto RUN (internal pull-up on Pico) ✓. RP2040 SWD not broken out on the carrier, but the Pico's own debug pads remain accessible.
- **EEPROM write protection:** WC pulled high (10 k) on all three modules, ground-bridge to enable writes ✓.
- **TLC59116 address straps:** JP35–38 default to GND → 0x60, matching the note ✓.
- **HP driver (TPS922051) design:** 65 V part, PMEG100T030 (100 V/3 A) catch diode, VIN-referenced LED string with 0.2 Ω sense via 100 Ω/1 nF filter — matches the TI reference topology; component ratings fine at 24 V.
- **Bulk/output caps:** 470 µF/50 V electrolytics, 50 V ceramics on V_LED nodes — rated for the 24 V option ✓ (C12 tantalum is on 3V3ISO at 10 V rating — 3.3/10 derating fine).
- **Q1/Q2/Q3 (55–60 V FETs) and AP74700 (40 V max)** — fine against 24 V exposure.

---

## 6. Documentation / librarian issues

1. `R?` / value `0R?` on sheet 3 — this is the carrier-3V3-to-Nucleo-3V3 tie and it appears as "R?" in the BOM too. Re-run annotation before release.
2. Drive Select sheet has a completely blank title block (no title/number/sheet index).
3. Every sheet says "Sheet x of 22" but the set runs to 24 pages including the layout/BOM pages — reconcile.
4. Silkscreen: LED DAT/CLK flipped; DAT[1-4]/CLK[1-4] should read [4..1] (or fix electrically per §4).
5. Block diagram reflects the JP18/JP20 crossover (§1.3) — fix both together.
6. SmartLED layer to-do noted on the layout page ("add power plane to signal layer").
7. Sheet-1 red note boxes should be resolved/cleared as items close.

---

## 7. BOM review

- **GCM1885C1H101JA16D (C96_HPx, 100 pF) is marked "Not Recommended for New Design"** in your own BOM export — pick a replacement now (e.g., GRM1885C1H101JA01 or GCM equivalent in production status).
- F1 (0ZCG, 3 A) and F2 (0ZCJ, 1 A) are 6 V PTCs — see §2.6.
- VR1 3306P-1-102: single-turn trimmer setting a safety-relevant current limit — a fixed resistor + optional parallel trim, or a multi-turn part, is more robust (pairs with §2.4).
- J11 listed as Würth 613010243121 — confirm this is actually the *socket* variant; that part family number reads like a pin header. All other female/male pairings (Sullins/Adam Tech) look consistent.
- 33 solder-bridge jumpers all default 1-2: confirm that default is correct for **every** instance (it is for the drive-select = STM32 and TLC address = 0x60; verify JP21/JP24/JP26 intents against §1.4 and §3.4).

---

## 8. Prioritized Rev 2 change list

| # | Priority | Change |
|---|---|---|
| 1 | Blocker | Re-pin RP2040 DMX_I2C: SDA→GP6, SCL→GP7 (I2C1) |
| 2 | Blocker | Move DMX_SPI.CS1 GP7→GP9 (SPI1 CSn; GP9 currently free) |
| 3 | Blocker | Swap JP18/JP20 pin-3 nets (RP_LED3.Data ↔ RP_LED4.Data) |
| 4 | Blocker | Resolve SmartLED DIR net: remove J18/J20 pin 3 from DIR; confirm E0/JP24 is a separate net (§1.4) |
| 5 | High | Replace U14 AP74700 with low-voltage ORing (TPS2116/LM66100/Schottky) for 3V3→RP_VSYS |
| 6 | High | Enforce/inhibit LP module above ~15 V V_LED (TLC59116 17 V abs max) |
| 7 | High | Protect RP2040 from 5 V in U10 B→A mode (series R / clamp / forbid mode) |
| 8 | High | Raise R57 so Rext ≥ ~160 Ω at trimmer minimum |
| 9 | Medium | TVS → SMAJ28A/30A; add fusing on V_LED inputs and LED power outputs |
| 10 | Medium | Reduce R13 to ~270 Ω for opto RX margin; verify TLP2368 IF threshold |
| 11 | Medium | Add CLK/DAT select jumpers on HP PWM2/PWM4 for hardware timer channels; verify PD7/PC3/PB2 timer AFs |
| 12 | Medium | Series-R footprints on the four CLK lines to LED connectors |
| 13 | Medium | Distinct EEPROM addresses per module type |
| 14 | Low | Fix silkscreen numbering, annotation (R?), Drive Select title block, sheet counts; replace NRND C96 part; document one-brick rule and Nucleo power-bridge config |

---

*Method note: pin/net claims were cross-checked against the netlist embedded in the PDF (e.g., `PIU507 → DMX_I2C.SDA`, `PIJP1803 → RP_LED4.Data`). RP2040 GPIO function mapping is from the RP2040 datasheet Bank 0 function table. STM32H563 alternate-function claims marked "verify" should be checked against the datasheet AF tables before committing the Rev 2 netlist.*
