# 3. Phase B – Main Power Supply

## B1. First 12 V power-up with current-limited bench supply

**Objective:** perform a safe first power-up while limiting the energy available in case of a major assembly fault.

**Conditions:**
- No SFP inserted
- No USB connected
- No external MCU connected
- No PMOD / FLEX / Arduino connected
- Bench supply connected to the 12 V input

**Procedure:**

| Step | Action |
|---:|---|
| 1 | Set the bench supply to 12 V, current limit 50 mA |
| 2 | Power the board briefly while observing input voltage and input current |
| 3 | Check whether the supply remains in constant-voltage mode or immediately enters current-limit mode |
| 4 | If the supply enters current limit at 50 mA, turn power off, but do not classify this alone as a board failure |
| 5 | Increase the current limit to 150 mA and repeat the test |
| 6 | If needed, increase the current limit to 500 mA and repeat again |
| 7 | At each step, monitor for abnormal heating, smell, or any obvious sign of distress |
| 8 | Only if the previous steps are clean, increase the current limit to a maximum of 1 A for the next bring-up checks |
| 9 | If the board shows persistent immediate current limiting, input voltage collapse, or abnormal heating across several current-limit levels, cut power immediately and treat this as a suspected fault |

**Acceptance criteria:**
- No component heats abnormally during the first power-up attempts.
- No smell, smoke, or visible distress appears.
- Entering current-limit mode at 50 mA alone is not sufficient to conclude that the board is faulty.
- A persistent immediate current-limit condition over multiple current-limit settings, especially when combined with voltage collapse, abnormal heating, or suspicious cold-resistance measurements, is a strong indication of a real fault.

---

## B2. Rail Verification

Measure the DC rails after the board has stabilized.

| Rail        |     TP |                     Nominal | Bring-up tolerance |
| ----------- | -----: | --------------------------: | -----------------: |
| `VCCINT_1V` | `TP11` |                       1.0 V |               ±5 % |
| `+1V8`      |  `TP7` |                       1.8 V |               ±5 % |
| `+3V3`      |  `TP9` |                       3.3 V |               ±5 % |
| `+3V3_P`    | `TP10` |                       3.3 V |               ±5 % |
| `+1V2`      | `TP12` |                       1.2 V |               ±5 % |
| `+2V5`      | `TP13` |                       2.5 V |               ±5 % |
| `MGT_1V`    | `TP33` |                       1.0 V |               ±5 % |

**Acceptance criteria:**

* All voltages are within range.
* No abnormal local heating.

---

## B3. Power-Up Sequence on Oscilloscope

**Objective:** verify that the regulators start in a coherent order.

Expected sequence according to the schematic:

```text
+12V
→ U4 : VCCINT_1V
→ U5 : +1V8
→ U10 : +3V3
→ U6 : +3V3_P
→ U23 : +1V2 from +1V8
→ U19 : MGT_1V from +1V8
→ U18 : +2V5 from +3V3
```

Measure at least:

| Signal                | Point          |
| --------------------- | -------------- |
| `+12V`                | input          |
| `VCCINT_1V`           | `TP11`         |
| `+1V8`                | `TP7`          |
| `+3V3`                | `TP9`          |
| `+3V3_P`              | `TP10`         |
| `MGT_1V`              | `TP33`         |
| `Reset`               | `TP5`          |
| Aggregated power-good | `TP6` / `TP14` |

**Acceptance criteria:**

* No oscillation or repeated restart.
* `Reset` behavior is coherent during rail ramp-up.
* Power-good does not toggle repeatedly.
