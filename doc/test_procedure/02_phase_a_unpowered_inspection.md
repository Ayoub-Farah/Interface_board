# 2. Phase A – Unpowered Inspection

## A1. Visual inspection before any electrical testing

**Objective:** detect obvious manufacturing and assembly defects before applying any power.

| Category | References | Visual checks |
|---|---|---|
| Diodes – 2-pin polarized parts | `D2`, `D3` | Verify the orientation of each diode: the cathode stripe on the package must match the cathode marking on the PCB footprint. Check that the part is centered, not rotated by 180°, not tombstoned, not cracked, and that there is no solder bridge to adjacent pads or nearby nets. |
| Diode array / ESD protection device | `D1` | Verify package orientation and pin 1 alignment. This part must not be rotated. Check that the package marker matches the PCB marker, that all pins are soldered correctly, and that there is no bridge between pins. |
| Status LEDs | `LD1`, `LD2`, `LD3`, `LD4`, `LD5` | Verify the LED polarity: the cathode side of the LED must match the PCB cathode marking. Check that the LED is not reversed, not cracked, not lifted, and that both pads are properly soldered. A reversed LED may not destroy the board, but it can make bring-up and debug misleading. |
| Main regulators supplied from the 12 V rail | `U4`, `U5`, `U6`, `U10`, `U11` | Verify package orientation and pin 1 alignment. Check that the body is correctly aligned with the footprint, fully seated on the PCB, and not rotated. Inspect carefully for solder bridges or poor solder joints on the most critical pins: VIN, VOUT, SW, FB, EN, PG, PGND. Also check that the nearby capacitors and feedback resistors are present and correctly soldered. |
| Secondary regulators / LDOs | `U18`, `U19`, `U20`, `U23` | Verify the orientation and check for lifted pins, poor solder joints, or shorts between pins. For DFN/QFN-style packages (`U19`, `U23`), pay special attention to package alignment and signs of poor reflow, since defects can be harder to see once powered. |
| Clean clock supply regulator | `U12` | Verify the orientation and inspect soldering carefully. This regulator powers the clock section, so even if the main board powers up, a defect here can later cause misleading clock / PLL failures. |
| Local USB regulator | `U8` | Verify the orientation and solder quality, especially on VIN, SW, FB, EN, GND. A defect here may not immediately affect the 12 V power tree, but it can later create false USB / JTAG bring-up issues. |

**Acceptance criteria:**
- No component is missing.
- No polarized component is reversed.
- No visible solder bridge is present.
- No pin, pad, or corner is lifted.
- No package is cracked or mechanically damaged.
- No regulator is rotated or visibly misaligned.
- No obvious assembly defect is present around the regulator cells (missing capacitor, missing resistor, shifted passive, poor soldering).

---

## A2. Cold Resistance Between Rails and Ground

**Objective:** detect a hard short before applying power.

With the board unpowered, use an ohmmeter to measure between each test point and GND.

| Rail         |                         Point | Expected      |
| ------------ | ----------------------------: | ------------- |
| `+12V` input | `J16` power input or 12 V input | No hard short |
| `VCCINT_1V`  |                        `TP11` | No hard short |
| `+1V8`       |                         `TP7` | No hard short |
| `+3V3`       |                         `TP9` | No hard short |
| `+3V3_P`     |                        `TP10` | No hard short |
| `DDRVCC`     |                         `TP8` | No hard short |
| `+1V2`       |                        `TP12` | No hard short |
| `+2V5`       |                        `TP13` | No hard short |
| `MGT_1V`     |                        `TP33` | No hard short |
| `+3V3_USB`   |                         `TP1` | No hard short |

A stable, very low resistance, typically below 1 Ω, must block the test.

**If failure occurs:**

1. Do not power the board.
2. Inject a low voltage with a current limit on the suspicious rail, for example 0.5 V / 100 mA.
3. Locate the hot spot.
4. Inspect the nearby decoupling capacitors on that rail.
5. Repeat A2 after correction.
