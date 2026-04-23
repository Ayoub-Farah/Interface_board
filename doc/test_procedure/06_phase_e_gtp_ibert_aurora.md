# 6. Phase E – GTP, IBERT, and Aurora

## E1. GTP physical-layer test with IBERT

**Objective:** validate the external SFP serial paths before Aurora. Phase D has already validated the clock-generation chain, the 125 MHz GT reference clock, and the basic transceiver clocking state. Phase E therefore focuses on the two real optical directions: `SFP0 -> SFP1` first, then `SFP1 -> SFP0`.

IBERT (Integrated Bit Error Ratio Tester) is the AMD flow for in-system serial I/O validation and debug. It is used here to exercise the GTP lanes through Vivado Serial I/O Analyzer, confirm that the selected SFP path is alive, and observe link error counters before adding the Aurora protocol layer. ([AMD UG908][amd-ibert])

**Scope boundary with Phase D:**

* Phase D validates `/Clock-WR/WR_CLK0_P/N` and the GTP reference-clock bring-up.
* Phase E does not repeat the Phase D clock or PLL acceptance checks.
* If the IBERT link does not initialize, first confirm that Phase D still passes before debugging the SFP path.

**Key terms used in this test:**

* IBERT: an AMD-generated debug design used with Vivado Serial I/O Analyzer to exercise and observe serial links in hardware. It is intended for clocking, connectivity, bit-error, and margin/debug work. ([AMD UG908][amd-ibert])
* Vivado Serial I/O Analyzer: the Vivado Hardware Manager interface used to configure and observe IBERT links at run time. ([AMD UG908][amd-serial-io-analyzer])
* SFP cross-link: an external optical connection from the transmitter of one SFP port to the receiver of the other SFP port.

**Practical meaning of the test stages in this procedure:**

1. `SFP0 -> SFP1` means the optical transmitter of `SFP0` is connected to the optical receiver of `SFP1`. This validates the `SFP0` transmit path, the optical interconnect, and the `SFP1` receive path.
2. `SFP1 -> SFP0` means the optical transmitter of `SFP1` is connected to the optical receiver of `SFP0`. This validates the reverse physical direction: `SFP1` transmit path, optical interconnect, and `SFP0` receive path.

**Signals and counters to observe during the IBERT test:**

* IBERT link status for the selected GTP channel pair.
* IBERT line rate and channel mapping reported by Vivado Serial I/O Analyzer.
* IBERT link error counters over the selected observation interval.

**Test procedure:**

1. Confirm that Phase D4 has already passed.
2. Program the IBERT bitstream (`ibert_gtp_test.bit`). Use the board reference clock configuration validated in Phase D.
3. Open Vivado Hardware Manager and Serial I/O Analyzer. Create the GT link in the analyzer and verify that the line rate, reference clock selection, and transceiver locations match the generated design. Vivado Serial I/O Analyzer is the runtime interface used to interact with IBERT designs. ([AMD UG908][amd-serial-io-analyzer])
4. Test the external `SFP0 -> SFP1` direction:

   * connect `SFP0` TX to `SFP1` RX;
   * start the IBERT link test for this direction;
   * confirm that the link is stable;
   * confirm that the IBERT error counter remains at zero over the selected observation interval.

   If `SFP0 -> SFP1` fails while Phase D is still passing, focus on `SFP0` TX, `SFP1` RX, SFP insertion, optical path, cage soldering, local PCB routing.
5. Test the reverse external direction:

   * connect `SFP1` TX to `SFP0` RX;
   * start the IBERT link test for this direction;
   * confirm that the link is stable;
   * confirm that the IBERT error counter remains at zero over the selected observation interval.

   If `SFP0 -> SFP1` passes but `SFP1 -> SFP0` fails, focus on the reverse-direction hardware path and on any asymmetric mapping or polarity issue.
6. For longer qualification, run IBERT long enough to reach the desired tested bit count. A practical engineering target is `10^12 transmitted bits without error per link direction`.

**What each stage proves:**

* `SFP0 -> SFP1` passes: the `SFP0` transmit path and `SFP1` receive path work through the external optical interconnect.
* `SFP1 -> SFP0` passes: the `SFP1` transmit path and `SFP0` receive path work through the external optical interconnect.
* Both directions pass: the two SFP ports, optical interconnect, polarity, and lane mapping are ready for protocol-level testing.

**Acceptance criteria:**

* `SFP0 -> SFP1` is stable in IBERT.
* `SFP1 -> SFP0` is stable in IBERT.
* IBERT link error counters remain at zero over the required test duration.
* No unstable or repetitive link reset behavior is observed.

**Debug guide:**

| Symptom                                      | Meaning                                                        | Probable causes                                                                                                         |
| -------------------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| IBERT link does not initialize in either direction | The issue is probably not specific to one SFP direction.       | Phase D prerequisite not actually passing, wrong IBERT bitstream, wrong GT channel selection, SFP power issue.          |
| `SFP0 -> SFP1` fails                         | The SFP0 TX to SFP1 RX path has an issue.                    | `SFP0` TX issue, `SFP1` RX issue, damaged cable or module path. |
| `SFP1 -> SFP0` fails                         | The SFP1 TX to SFP0 RX path has an issue.                  | `SFP1` TX issue, `SFP0` RX issue, damaged cable or module path.               |
| Link comes up but IBERT errors accumulate    | The link is alive but not clean.                               | Signal-integrity issue, noisy reference clock source, marginal SFP, wrong line rate.                    |

---

## E2. Aurora point-to-point test

**Objective:** validate the Aurora protocol layer only after the external physical paths have already been proven by E1. At this stage, the goal is no longer just "can the transceiver move bits?" but "can the two endpoints initialize an Aurora channel, exchange packets correctly, and remain stable over time?" ([AMD Aurora 8B/10B][amd-aurora-flow-control])

**What changes compared to E1:**
E1 checks the external serial paths using IBERT. E2 checks the protocol running on top of those paths. Therefore, E2 can fail even if E1 passes: the physical link may be good while the Aurora configuration, reset handling, lane ordering, flow, or framing is wrong. Validate the external transceiver paths with IBERT first. ([AMD Aurora 8B/10B][amd-aurora-hardware-debug])

**Required firmware / debug signals:**

* `lane_up`: asserted for each lane that has completed lane initialization. In a single-lane Aurora design, this is expected to become `1` for that one lane. ([AMD Aurora 8B/10B][amd-aurora-flow-control])
* `channel_up`: asserted only after the whole Aurora channel initialization is complete and the channel is ready for data transfer. Aurora cannot receive valid data before `channel_up = 1`. ([AMD Aurora 8B/10B][amd-aurora-flow-control])
* `soft_err`: indicates an error detected in the incoming serial stream. Soft errors alone do not necessarily reset the core, but repeated bursts can escalate. ([AMD Aurora 8B/10B][amd-aurora-error-status])
* `hard_err`: indicates a serious hardware-side error such as transceiver-related faults or elastic-buffer overflow/underflow. When a hard error is detected, the Aurora core automatically resets and reinitializes. ([AMD Aurora 8B/10B][amd-aurora-error-status])
* `frame_err`: indicates an Aurora frame/protocol error, for example malformed frame boundaries. ([AMD Aurora 8B/10B][amd-aurora-error-status])
* `tx_count` / `rx_count`: user-defined counters used to confirm that transmitted and received packet counts are coherent at the system level.
* `reset_count`: user-defined counter used to detect spontaneous Aurora reinitialization events during long runs.

**Recommended procedure:**

1. Confirm that E1 passed in both directions.
2. Test Aurora over the external `SFP0 -> SFP1` direction.
3. Test Aurora over the reverse external `SFP1 -> SFP0` direction.
4. Send fixed-size packets first. This removes one variable and makes counter checking easier.
5. Then send variable-size packets.
6. Run continuous traffic for at least 30 minutes and watch:

   * `lane_up`
   * `channel_up`
   * `soft_err`
   * `hard_err`
   * `frame_err`
   * `tx_count`
   * `rx_count`
   * `reset_count`

**Practical meaning of the Aurora statuses:**

* If `lane_up = 0`, the Aurora lane has not initialized. This usually still points to a physical-link or reset/clock problem.
* If `lane_up = 1` but `channel_up = 0`, the individual lane is alive but the full Aurora channel has not completed initialization.
* If `channel_up = 1`, the Aurora channel is operational and ready to carry payload data. ([AMD Aurora 8B/10B][amd-aurora-flow-control])
* If `soft_err` pulses occasionally, the link is seeing bit errors but the channel may still continue operating.
* If `hard_err` asserts, the Aurora core treats the condition as severe enough to reset and reinitialize the channel. ([AMD Aurora 8B/10B][amd-aurora-error-status])
* If `frame_err` appears, the issue is no longer limited to random bit corruption; the protocol framing itself is being violated. ([AMD Aurora 8B/10B][amd-aurora-error-status])

**Acceptance criteria:**

* E1 has passed in both directions before starting the Aurora test.
* `lane_up = 1` for the lane under test.
* `channel_up = 1`.
* `tx_count` and `rx_count` remain coherent.
* No `hard_err`.
* No `frame_err`.
* No spontaneous link reset during the observation window.
* Stable point-to-point latency in FPGA clock cycles.
* `SFP0 -> SFP1` passes.
* `SFP1 -> SFP0` passes.
* No progressive drift in error counters over the long-duration run. ([AMD Aurora 8B/10B][amd-aurora-flow-control])

**Debug guide:**

| Symptom                                     | Meaning                                                                   | Probable causes                                                                                             |
| ------------------------------------------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `lane_up = 0`                               | Aurora lane never initializes.                                            | Underlying GTP issue, missing refclk, reset polarity error, cable, fiber, or optical module problem, wrong mapping. |
| `lane_up = 1`, `channel_up = 0`             | Physical lane starts, but Aurora channel does not complete initialization. | Protocol-side init issue, configuration mismatch, lane ordering issue, unstable received symbols.           |
| `channel_up` toggles up/down                | Channel comes up but is not stable.                                       | Possible RX buffer overflow/underflow, clock-correction issue, unstable external link, or reset issue.      |
| `channel_up = 1`, but `soft_err` increments | Channel is up, but the data stream is corrupted.                          | Marginal signal integrity, noisy clock, external optical path issue, line-rate mismatch, polarity issue.    |
| `hard_err` asserted                         | Severe error detected; Aurora will reinitialize.                          | Burst of soft errors, elastic-buffer overflow/underflow, hardware-side instability.                         |
| `frame_err` asserted                        | Protocol framing is being violated.                                       | Corrupted symbols, malformed traffic, channel instability, reset/clock issue during traffic.                |
| `tx_count` and `rx_count` diverge           | Packets are being lost or not accepted.                                   | Dropped packets, resets during traffic, framing errors, counters placed at different datapath points.       |

## Source notes for this section

* IBERT and Vivado Serial I/O Analyzer: AMD flow for in-system serial I/O validation and debug. ([AMD UG908][amd-ibert], [AMD UG908][amd-serial-io-analyzer])
* GT serial-link debug: use Vivado Serial I/O Analyzer to configure and observe the IBERT links in hardware. ([AMD GT Wizard][amd-gt-debug])
* Aurora hardware debug: validate the transceiver path first when the Aurora channel is unstable in hardware. ([AMD Aurora 8B/10B][amd-aurora-hardware-debug])
* Aurora status and error signals: `lane_up`, `channel_up`, `soft_err`, `hard_err`, and `frame_err`. ([AMD Aurora 8B/10B][amd-aurora-flow-control], [AMD Aurora 8B/10B][amd-aurora-error-status])

## References

- [AMD UG908 - IBERT][amd-ibert]
- [AMD GT Wizard - Debugging Using Serial I/O Analyzer][amd-gt-debug]
- [AMD UG908 - Using Vivado Serial I/O Analyzer to Debug the Design][amd-serial-io-analyzer]
- [AMD Aurora 8B/10B - Native Flow Control][amd-aurora-flow-control]
- [AMD Aurora 8B/10B - Hardware Debug Step 10][amd-aurora-hardware-debug]
- [AMD Aurora 8B/10B - Error Status Signals][amd-aurora-error-status]

[amd-ibert]: https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/IBERT "AMD UG908 - IBERT"
[amd-gt-debug]: https://docs.amd.com/r/en-US/pg168-gtwizard/Debugging-Using-Serial-I/O-Analyzer "AMD GT Wizard - Debugging Using Serial I/O Analyzer"
[amd-serial-io-analyzer]: https://docs.amd.com/r/2025.1-English/ug908-vivado-programming-debugging/Using-Vivado-Serial-I/O-Analyzer-to-Debug-the-Design "AMD UG908 - Using Vivado Serial I/O Analyzer to Debug the Design"
[amd-aurora-flow-control]: https://docs.amd.com/r/en-US/pg046-aurora-8b10b/Native-Flow-Control "AMD Aurora 8B/10B - Native Flow Control"
[amd-aurora-hardware-debug]: https://docs.amd.com/r/en-US/pg046-aurora-8b10b/Step-10-Channel-comes-up-in-simulation-but-not-in-hardware "AMD Aurora 8B/10B - Channel comes up in simulation but not in hardware"
[amd-aurora-error-status]: https://docs.amd.com/r/en-US/pg046-aurora-8b10b/Error-Status-Signals "AMD Aurora 8B/10B - Error Status Signals"
