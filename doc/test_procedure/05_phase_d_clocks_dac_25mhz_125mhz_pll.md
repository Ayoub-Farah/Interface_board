# 5. Phase D – Clocks, DAC, 25 MHz Oscillator, and 125 MHz PLL

## D1. Clean clock supply verification

**Objective:** verify the supply rails of the clock-generation chain before attempting any clock or transceiver test.

The supply rails checked in this step are not fast clocks. They are DC / low-frequency measurements whose goal is to confirm presence and correct level. The LP5951 used as `U12` is a low-noise 3.0 V LDO, the LM336-2.5 used as `U13` is a 2.49 V shunt reference, and the Si5351A allows `VDDO = 1.8/2.5/3.3 V`, which matches the use of `+2V5` on `U3 VDDO`. ([TI LP5951][ti-lp5951], [TI LM336-2.5][ti-lm336], [Skyworks Si5351][skyworks-si5351])

| Signal       | Function                                | Expected value                | Recommended instrument |
| ------------ | --------------------------------------- | ----------------------------: | ---------------------- |
| `+3V3_CLEAN` | Clean 3.3 V rail for the clock chain    |                       ≈ 3.3 V | DMM                    |
| `U12-VOUT`   | `U12` 3.0 V LDO output feeding `Y2 VDD` |                       ≈ 3.0 V | DMM                    |
| `U13-C`      | `U13` DAC reference rail                |                  ≈ 2.49–2.5 V | DMM                    |
| `+2V5`       | `U3 VDDO` rail                          |                       ≈ 2.5 V | DMM                    |
| `U16-CE`     | `U16` enable pin                        |                    logic high | DMM                    |
| `U16-~RSTN`  | `U16` reset pin                         | logic high (reset disabled) | DMM                    |

**Acceptance criteria:**

* All rails are within tolerance.
* No large ripple or intermittent collapse is observed.
* No component around `U12`, `U13`, `U14`, `U15`, or `U16` is abnormally hot.
* `U16 CE` is high and `U16 ~RSTN` is high (reset disabled).

---

## D2. DAC control-voltage and DAC-SPI test

**Objective:** verify that the FPGA drives the two DAC channels correctly and that the analog control voltages delivered to the oscillators are valid.

The DAC test is split into two domains:

1. the analog DAC outputs `WR_DAC_OUT1` and `WR_DAC_OUT2`, which are slow control voltages;
2. the digital SPI interface (`WR_DAC_SCLK`, `WR_DAC_DIN`, `WR_DAC1_SYNC`, `WR_DAC2_SYNC`), which can run up to the DAC interface limit.

The DAC8551 requires an external reference, powers up at 0 V, is monotonic, has a 10 µs settling time, and supports serial clock rates up to 30 MHz. ([TI DAC8551][ti-dac8551])

### Required test bitstream

Use a dedicated test bitstream named `clock_dac_test.bit`.

This bitstream must implement:

* two independent 16-bit write registers: `dac1_code[15:0]` and `dac2_code[15:0]`;
* one write per DAC;
* an optional automatic sweep mode cycling through
  `0x0000 → 0x4000 → 0x8000 → 0xC000 → 0xFFFF`;
* debug observation with an ILA (integrated logic analyzer). ([AMD Vivado][amd-vivado-hw-manager])

### Signals to verify

| Signal                   | Type    | What to verify                               | Recommended observation |
| ------------------------ | ------- | -------------------------------------------- | ----------------------- |
| `/Clock-WR/WR_DAC_OUT1`  | Analog  | Control voltage, slow-settling DAC output    | DMM                     |
| `/Clock-WR/WR_DAC_OUT2`  | Analog  | Control voltage, slow-settling DAC output    | DMM                     |
| `/Clock-WR/WR_DAC_SCLK`  | Digital | DAC interface clock                          | ILA                     |
| `/Clock-WR/WR_DAC_DIN`   | Digital | SPI data aligned with SCLK                   | ILA                     |
| `/Clock-WR/WR_DAC1_SYNC` | Digital | Control pulse associated with DAC1 SPI frame | ILA                     |
| `/Clock-WR/WR_DAC2_SYNC` | Digital | Control pulse associated with DAC2 SPI frame | ILA                     |

### Procedure

1. Program `clock_dac_test.bit` through JTAG.
2. Write the following codes successively to the selected DAC:

   * `0x0000`
   * `0x4000`
   * `0x8000`
   * `0xC000`
   * `0xFFFF`
3. Measure:

   * `/Clock-WR/WR_DAC_OUT1` near `R103 / C190`
   * `/Clock-WR/WR_DAC_OUT2` near `R102 / C189`
4. Analyze the SPI activity with an integrated logic analyzer (ILA) in Vivado:

   * `/Clock-WR/WR_DAC_SCLK`
   * `/Clock-WR/WR_DAC_DIN`
   * `/Clock-WR/WR_DAC1_SYNC`
   * `/Clock-WR/WR_DAC2_SYNC`

### Acceptance criteria

* `WR_DAC_OUT1` and `WR_DAC_OUT2` are monotonic.
* The expected analog range is approximately 0 V to VREF, therefore about 0 V to 2.5 V if `U13` is healthy. ([TI DAC8551][ti-dac8551])
* No output is permanently stuck at 0 V or 3.3 V.
* SPI activity is clean and consistent with the selected DAC and code.

### Debug guide

| Symptom                  | Check                                                                       |
| ------------------------ | --------------------------------------------------------------------------- |
| DAC output stuck at 0 V  | `+3V3_CLEAN`, SYNC polarity, SPI activity                                   |
| DAC output non-monotonic | SPI timing, wrong bit order, wrong clock polarity                           |
| DAC1/DAC2 appear swapped | Verify `/Clock-WR/WR_DAC1_SYNC` vs `/Clock-WR/WR_DAC2_SYNC` mapping in FPGA |

---

## D3. 25 MHz oscillator test at `U16 XIN`

**Objective:** verify that the 25 MHz signal reaching `U16 XIN` is present and stable before attempting any 125 MHz output or GTP reference-clock test.

The TI CDCM61002 accepts a single crystal/LVCMOS reference input in the range 21.875 MHz to 28.47 MHz, which includes 25 MHz, and supports generating a 125 MHz output. ([TI CDCM61002][ti-cdcm61002])

### Important clarification

In this procedure, the 25 MHz signal under test is the signal that reaches `U16 XIN`.
On the board, this path runs from `Y2 OUT` toward `U16 XIN` through the surrounding selection / coupling network in the Clock-WR sheet.

### Required test bitstream

Continue using `clock_dac_test.bit` so that:

* `WR_DAC_OUT1` can be forced to mid-scale first,
* then swept around mid-scale to confirm that the oscillator control path responds.

### Signals to verify

| Signal    | Type  | What to verify                  | Recommended observation |
| --------- | ----- | ------------------------------- | ----------------------- |
| `Y2-OUT`  | Clock | 25 MHz clock output             | Scope near source       |
| `U16-XIN` | Clock | 25 MHz clock entering `U16 XIN` | Scope if needed         |

### Procedure

1. Program `clock_dac_test.bit`.
2. Force `/Clock-WR/WR_DAC_OUT1` to mid-scale.
3. Measure first at `Net-(Y2-OUT)`, ideally on the `Y2` side of `R104`.
4. If needed, measure at `Net-(U16-XIN)`.
5. Sweep `WR_DAC_OUT1` slightly around mid-scale and observe whether the oscillation remains present and changes smoothly.

### Acceptance criteria

* A periodic signal at approximately 25 MHz is present.
* The signal is stable enough to be accepted as the `U16` reference input.
* No discontinuity or dead zone appears while sweeping the DAC in the intended operating range.

---

## D4. `U16` 125 MHz output and FPGA-side reference-clock validation

**Objective:** verify that `U16` generates the differential 125 MHz reference clock and that this reference clock is accepted by the FPGA transceiver clocking path.

The CDCM61002 supports 125 MHz as a common LVDS/LVPECL/LVCMOS output frequency, and the board routes `U16 OUTP0/OUTN0` through AC-coupling to `/Clock-WR/WR_CLK0_P/N`, which then feed the FPGA `MGTREFCLK0P/N` pins. ([TI CDCM61002][ti-cdcm61002])

### Acceptance approach

Use FPGA-based validation for the differential 125 MHz reference clock. The test must prove both physical arrival at the FPGA reference-clock pins and usability by the GTP clocking path.

### Test file 1 — `wr_refclk_odiv2_test.bit`

Purpose: verify that the 125 MHz differential reference clock from the board reaches the FPGA transceiver reference-clock pins. This is a clock presence test only. It does not yet prove that the GTP transceiver PLL can lock correctly; that must be checked in the next step with `CPLLLOCK`. ([AMD GT Wizard][amd-gt-debug])

Background: `/Clock-WR/WR_CLK0_P/N` is the differential reference-clock pair connected to the FPGA dedicated transceiver reference-clock input pins, not to ordinary user I/O pins. The design must instantiate the `IBUFDS_GTE2` primitive on that path. ([AMD 7 Series transceiver documentation][amd-ibufds-gte2])

Reason for using `IBUFDS_GTE2`: `IBUFDS_GTE2` is the 7-series input buffer used for GT reference clocks. The primitive exposes two outputs, `O` and `ODIV2`; `ODIV2` is suitable for fabric-side observation of the reference clock. ([AMD 7 Series transceiver documentation][amd-ibufds-gte2], [AMD PB043][amd-util-ds-buf])

Test principle: the bitstream does not try to use the transceiver yet. It takes the external differential clock, passes it through `IBUFDS_GTE2`, and uses the `ODIV2` output as the clock for a simple counter in the FPGA fabric. A high-order bit of that counter drives a heartbeat LED. In this test, if the incoming reference clock is 125 MHz, the internal observation clock is expected to be about 62.5 MHz before the counter divides it down further to a visible blink rate. ([AMD PB043][amd-util-ds-buf])

Implementation:

* instantiate `IBUFDS_GTE2` on `/Clock-WR/WR_CLK0_P/N`;
* use `ODIV2` as the clock for a free-running fabric counter;
* drive a heartbeat LED with one high-order bit of that counter;
* optionally connect the same clock and counter to an ILA for hardware observation.

Expected behavior:

* if `/Clock-WR/WR_CLK0_P/N` is present and valid, the counter runs continuously;
* the heartbeat LED toggles steadily;
* the LED is not driven directly at 62.5 MHz — the counter first divides the clock down to a human-visible rate;
* optionally, an ILA should show continuous clock-driven activity inside the FPGA.

Interpretation:

* Pass: the LED blinks and/or the ILA shows activity. This means the board-level 125 MHz reference clock is physically reaching the FPGA reference-clock input path.
* Fail: the LED stays static and the ILA shows no activity. This suggests a problem such as:

  * missing 125 MHz clock from the board,
  * wrong FPGA pin assignment,
  * constraint error,
  * or a hardware issue in the path between `U16` and the FPGA reference-clock pins.


### Test file 2 — `gtp_cpll_lock_test.bit`

Purpose: verify that the GTP reference clock is usable by the transceiver, not just physically present at the package pins.

Implementation:

* instantiate the 7 Series Transceivers Wizard for the Artix-7 GTP lane using `/Clock-WR/WR_CLK0_P/N` as REFCLK,
* expose the following status signals to LEDs or ILA:

  * `CPLLLOCK`
  * `TXRESETDONE`
  * `RXRESETDONE`

Monitor `CPLLLOCK` and `TXRESETDONE` / `RXRESETDONE` during transceiver bring-up and debug. ([AMD GT Wizard][amd-gt-debug])

Expected behavior:

* `CPLLLOCK = 1`
* `TXRESETDONE = 1`
* `RXRESETDONE = 1`

For this board family (`XC7A35T`, GTP transceivers), `CPLLLOCK` is the primary lock indicator to use in the acceptance test. The goal here is not generic FPGA lock, but specifically transceiver reference-clock lock.

### Signals and acceptance method

| Signal                | Type               | What is verified                         | Acceptance method   |
| --------------------- | ------------------ | ---------------------------------------- | ------------------- |
| `/Clock-WR/WR_CLK0_P` | Differential clock | 125 MHz reference clock reaches the FPGA | FPGA test bitstream |
| `/Clock-WR/WR_CLK0_N` | Differential clock | 125 MHz reference clock reaches the FPGA | FPGA test bitstream |
| `CPLLLOCK`            | FPGA status        | GTP CPLL is locked                       | LED / ILA           |
| `TXRESETDONE`         | FPGA status        | GTP TX reset sequence completed          | LED / ILA           |
| `RXRESETDONE`         | FPGA status        | GTP RX reset sequence completed          | LED / ILA           |

### Procedure

1. Confirm D1, D2, and D3 are already passing.
2. Use FPGA-side validation as the acceptance method for `/Clock-WR/WR_CLK0_P/N`.
3. Program `wr_refclk_odiv2_test.bit`.
4. Confirm that the divided fabric clock is alive.
5. Program `gtp_cpll_lock_test.bit`.
6. Confirm:

   * `CPLLLOCK = 1`
   * `TXRESETDONE = 1`
   * `RXRESETDONE = 1`
7. Continue with Phase E for external SFP serial-link validation.

### Acceptance criteria

* `wr_refclk_odiv2_test.bit` proves that `/Clock-WR/WR_CLK0_P/N` reaches the FPGA clocking input.
* `gtp_cpll_lock_test.bit` proves that the GTP reference-clock path is valid enough for transceiver initialization.

---

## Source notes for this section

* DAC8551: external reference DAC, monotonic, 10 µs settling time, SPI clock rate up to 30 MHz. ([TI DAC8551][ti-dac8551])
* LM336-2.5: fixed 2.49 V shunt reference. ([TI LM336-2.5][ti-lm336])
* LP5951: fixed 3.0 V version available; low-noise LDO. ([TI LP5951][ti-lp5951])
* Si5351A: `VDDO` can be 1.8 / 2.5 / 3.3 V. ([Skyworks Si5351][skyworks-si5351])
* CDCM61002: reference input range 21.875–28.47 MHz, common supported outputs include 125 MHz. ([TI CDCM61002][ti-cdcm61002])
* Vivado / DONE status: Hardware Manager programming and `DONE` status check. ([AMD UG908][amd-program-device])
* Transceiver debug: use `CPLLLOCK`, `TXRESETDONE`, `RXRESETDONE` as documented GT bring-up indicators. ([AMD GT Wizard][amd-gt-debug])
* IBUFDS_GTE2: `ODIV2` is the divide-by-2 fabric-accessible output of the GT reference-clock input buffer. ([AMD 7 Series transceiver documentation][amd-ibufds-gte2])

## References
- [TI LP5951][ti-lp5951]
- [TI LM336-2.5][ti-lm336]
- [Skyworks Si5351A/B/C-B datasheet][skyworks-si5351]
- [TI DAC8551][ti-dac8551]
- [AMD UG910 - Opening the Hardware Manager][amd-vivado-hw-manager]
- [TI CDCM61002][ti-cdcm61002]
- [AMD GT Wizard - Debugging Using Serial I/O Analyzer][amd-gt-debug]
- [AMD UG908 - Programming the Hardware Device][amd-program-device]
- [AMD 7 Series transceiver documentation][amd-ibufds-gte2]
- [AMD PB043 - Utility Buffer I/O Signals][amd-util-ds-buf]

[ti-lp5951]: https://www.ti.com/product/LP5951 "TI LP5951"
[ti-lm336]: https://www.ti.com/product/LM336-2.5 "TI LM336-2.5"
[skyworks-si5351]: https://www.skyworksinc.com/-/media/Skyworks/SL/documents/public/data-sheets/Si5351-B.pdf "Skyworks Si5351A/B/C-B datasheet"
[ti-dac8551]: https://www.ti.com/product/DAC8551 "TI DAC8551"
[amd-vivado-hw-manager]: https://docs.amd.com/r/2024.1-English/ug910-vivado-getting-started/Opening-the-Hardware-Manager "AMD UG910 - Opening the Hardware Manager"
[ti-cdcm61002]: https://www.ti.com/product/CDCM61002 "TI CDCM61002"
[amd-gt-debug]: https://docs.amd.com/r/en-US/pg168-gtwizard/Debugging-Using-Serial-I/O-Analyzer "AMD GT Wizard - Debugging Using Serial I/O Analyzer"
[amd-program-device]: https://docs.amd.com/r/2024.1-English/ug908-vivado-programming-debugging/Programming-the-Hardware-Device "AMD UG908 - Programming the Hardware Device"
[amd-ibufds-gte2]: https://docs.amd.com/api/khub/documents/SAXb2rXapMfInryXnPr5NQ/content "AMD 7 Series transceiver documentation"
[amd-util-ds-buf]: https://docs.amd.com/r/en-US/pb043-util-ds-buf/I/O-Signals "AMD PB043 - Utility Buffer I/O Signals"
