# 4. Phase C – USB, JTAG, and FPGA Configuration

## C1. USB back-power test

**Objective:** verify that USB does not unintentionally power the main rails.

**Procedure:**

1. Board not powered from 12 V.
2. Connect USB-C only.
3. Measure:

   * `TP1`: `+3V3_USB`
   * `TP9`: `+3V3`
   * `TP11`: `VCCINT_1V`

**Acceptance criteria:**

* `+3V3_USB` may be present.
* Main FPGA rails must not be significantly powered through USB leakage.
* If `+3V3` or `VCCINT_1V` rises substantially without 12 V, investigate FT2232, pull-ups, and protection paths.

---

## C2. USB interface bring-up and FT2232 enumeration

**Objective:** verify that the on-board USB-to-debug bridge is powered correctly and enumerates stably on the host PC.

**Procedure:**

| Step | Action |
|---:|---|
| 1 | Power the board from the 12 V bench supply. |
| 2 | Confirm from the previous checks that the board is in a valid powered state: no abnormal heating, no current-limit condition, and no unstable reset behavior. |
| 3 | Connect the USB-C cable to the board and to the PC. |
| 4 | Measure `TP1` and confirm that `+3V3_USB` is present after USB insertion. |
| 5 | On the PC, check that the FT2232H is detected and enumerates stably. |
| 6 | Confirm that the USB device does not repeatedly disconnect and reconnect. |
| 7 | If the FT2232 is not detected, or if enumeration is unstable, stop here and debug the USB/FT2232 path before attempting JTAG access. |

**Acceptance criteria:**

* `+3V3_USB` is present at `TP1` after USB insertion.
* The FT2232 enumerates stably on the host PC.
* No repeated disconnect/reconnect is observed.
* The expected USB debug interfaces are visible on the host PC.

**Failure interpretation:**

* No USB enumeration: check `U21`, USB connector, ESD device, `+3V3_USB`, and USB differential routing.
* Unstable enumeration: check USB power integrity, soldering around `U21`, and local USB supply.

---

## C3. First FPGA access through JTAG

**Objective:** verify that the host PC can reach the FPGA through the on-board USB/JTAG interface and that the FPGA can be configured for the first time through a volatile JTAG download.

**Important notes:**

* A blank or unprogrammed FT2232H EEPROM does not prevent USB enumeration. In that case, the FT2232H falls back to its built-in default USB descriptors (`VID 0403`, `PID 6010`) and exposes no serial number. ([FTDI][ftdi-ft2232h])
* Vivado Hardware Manager recognizes the FT2232H as a USB-to-JTAG cable only after the FTDI EEPROM has been programmed with `program_ftdi`. Keep the Manufacturer field set to `Xilinx`. ([AMD UG908][amd-program-ftdi])
* After a hardware target is opened in Vivado, the Hardware window should show the hardware server, the hardware target, and the hardware devices found on that target. ([AMD UG908][amd-open-target])

**Preconditions:**

* The board has already passed the previous power checks.
* USB connection is stable.
* No external MCU, SFP, or optional peripheral is required for this step.

**Procedure:**

| Step | Action                                                                                                                                                                  | Expected observation on the PC / in Vivado                                                                                                                                                                                                                |
| ---: | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|    1 | Keep the board powered from 12 V and keep the USB-C cable connected.                                                                                                    | The board remains in a stable powered state. No abnormal heating, no repeated USB disconnect/reconnect.                                                                                                                                                   |
|    2 | On the host PC, confirm that a stable FTDI USB device appears after USB insertion.                                                                                      | The operating system shows a persistent FTDI/USB device. If nothing appears at all, stop here and debug USB first.                                                                                                                                        |
|    3 | Open Vivado Hardware Manager.                                                                                                                                           | Hardware Manager opens normally.                                                                                                                                                                                                                          |
|    4 | Open a local hardware target in Vivado.                                                                                                                                 | Vivado should show a hardware server and then a hardware target in the Hardware window. After the target is opened, the Hardware window should populate with the server, the target, and the devices detected on that target. ([AMD UG908][amd-hw-server]) |
|    5 | If the PC sees the FTDI device but Vivado does not show a usable JTAG target/cable, stop the JTAG attempt and program the FTDI EEPROM using Vivado `program_ftdi`.      | This is the expected corrective action on a fresh board if the FTDI EEPROM has not yet been programmed for Vivado JTAG recognition. ([AMD UG908][amd-program-ftdi])                                                                                         |
|    6 | Program the FTDI EEPROM with the Vivado-supported configuration for FT2232H.                                                                                             | Use `program_ftdi` with the correct FTDI part and a valid serial number. The utility supports `FT2232H`; typical syntax is `program_ftdi -write -ftdi FT2232H -serial <serial> ...`. ([AMD UG908][amd-program-ftdi])                                       |
|    7 | Unplug and reconnect USB after FTDI EEPROM programming.                                                                                                                 | The host PC re-enumerates the FTDI with its new EEPROM configuration. This is necessary because the USB descriptors are read by the host at enumeration/reset time. ([FTDI][ftdi-ft2232h])                                                               |
|    8 | Re-open the hardware target in Vivado.                                                                                                                                  | A usable hardware target/cable should now appear in the Hardware window.                                                                                                                                                                                  |
|    9 | Scan the JTAG chain.                                                                                                                                                    | The FPGA device should appear in the Hardware window. The expected FPGA is `XC7A35T-CSG325`.                                                                                                                                                              |
|   10 | If the FPGA is detected, right-click the device and select Program Device with a minimal test bitstream.                                                                | Vivado should complete the programming operation without error. Programming can be launched from the Hardware window with Program Device or from Tcl with `program_hw_devices`. ([AMD UG908][amd-program-device])                                           |
|   11 | Check that the FPGA has entered the configured state.                                                                                                                   | The simplest checks are the `DONE` status in the Hardware Device Properties view, a visible LED blink, or a simple counter output. ([AMD UG908][amd-program-device])                                                                                             |
|   12 | Monitor input current and board temperature during and after configuration.                                                                                             | No abnormal current increase, no abnormal heating, no smell, no reset loop.                                                                                                                                                                               |
|   13 | Optionally power-cycle the board once after the JTAG test.                                                                                                              | If the external configuration flash has not yet been programmed, the FPGA is not expected to boot autonomously after power-cycle. This is normal at this stage.                                                                                            |

**Recommended minimal bitstream:**

* LED blink
* free-running counter
* visible reset behavior
* optional debug outputs on accessible pins

**Acceptance criteria:**

* The USB FTDI interface appears stably on the host PC.
* Vivado Hardware Manager can open a hardware target.
* The FPGA appears in the JTAG chain.
* The minimal bitstream downloads successfully through JTAG.
* The FPGA shows a valid configured-state indication (`DONE` or equivalent visible behavior).
* No abnormal current increase or abnormal heating occurs after configuration.

**Failure interpretation:**

| Symptom                                                                              | Meaning                                                                             | Probable causes                                                                                                                   |
| ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| No FTDI device appears on the PC                                                     | The USB link to the FT2232 is not active                                            | USB connector, ESD protection, FT2232 power, FT2232 soldering, USB differential pair                                              |
| FTDI device appears on the PC, but no usable hardware target/cable appears in Vivado | The FTDI enumerates, but is not yet recognized by Vivado as a USB-to-JTAG interface | FTDI EEPROM not programmed for Vivado, wrong FTDI EEPROM content, wrong Manufacturer field, host driver issue ([AMD UG908][amd-program-ftdi]) |
| Hardware target opens, but no FPGA device appears in the chain                       | Vivado reaches the cable, but not the FPGA through JTAG                             | `VCCINT_1V`, `+1V8`, `+3V3`, FPGA reset/config pins, TCK/TMS/TDI/TDO connectivity                                                 |
| FPGA appears, but programming fails                                                  | JTAG path exists, but reliable FPGA configuration is not achieved                   | unstable power, reset/config pin issue, JTAG signal integrity, wrong bitstream/device mismatch                                    |
| JTAG programming succeeds, but nothing boots after power-cycle                       | Normal if the flash is still blank                                                  | external configuration flash not yet programmed                                                                                   |

---

## C4. External configuration flash programming

**Objective:** program the external non-volatile configuration memory so that the FPGA boots automatically after power-up.

**Important notes:**

* Vivado programs configuration memory indirectly via JTAG: it first programs the FPGA with a temporary configuration that creates a data path between JTAG and the flash interface, then it uses that path to erase, program, and verify the configuration memory. ([AMD UG908][amd-config-memory])
* The overall flow is:
  generate the device image → create the configuration memory file (`.mcs` or `.bin`) → connect to the hardware target → add the configuration memory device → program the configuration memory → boot the device. ([AMD UG908][amd-config-memory])
* After adding the configuration memory device, Vivado opens the Program Configuration Memory Device flow. The selected configuration-memory file is typically a `.mcs` or `.bin` file. ([AMD UG908][amd-program-config-memory])

**Preconditions:**

* C3 has already succeeded.
* JTAG communication with the FPGA is stable.
* The intended boot image has already been built.

**Procedure:**

| Step | Action                                                                                                                         | Expected observation on the PC / in Vivado                                                                                                                                     |
| ---: | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
|    1 | Keep the board powered and connected through USB.                                                                              | USB and JTAG remain stable.                                                                                                                                                    |
|    2 | Open Vivado Hardware Manager and re-open the hardware target if needed.                                                        | The hardware server, hardware target, and FPGA device are visible again in the Hardware window. ([AMD UG908][amd-open-target])                                                   |
|    3 | Confirm that the FPGA is accessible through JTAG.                                                                              | The FPGA remains visible and selectable in the Hardware window.                                                                                                                |
|    4 | Prepare the configuration-memory image corresponding to the intended boot design.                                              | The required file is a configuration-memory file such as `.mcs` or `.bin`. ([AMD UG908][amd-config-memory])                                                                      |
|    5 | In the Hardware window, right-click the hardware target and select Add Configuration Memory Device.                            | Vivado opens the Add Configuration Memory Device dialog. ([AMD UG908][amd-add-config-memory])                                                                                    |
|    6 | Select the exact flash memory device fitted on the board, then click OK.                                                       | The configuration memory device is added under the hardware target in Vivado. If the memory type is unknown, stop here and identify it before programming. ([AMD UG908][amd-add-config-memory]) |
|    7 | When Vivado prompts to program the configuration memory, continue and open the Program Configuration Memory Device dialog.      | The dialog asks for the configuration file and programming options. ([AMD UG908][amd-program-config-memory])                                                                     |
|    8 | Select the correct `.mcs` or `.bin` file and launch programming.                                                               | Vivado performs the indirect flash-programming sequence: temporary FPGA configuration, then erases, programs, and verifies the attached flash. ([AMD UG908][amd-config-memory])   |
|    9 | Wait for programming and verification to complete.                                                                             | Vivado reports successful completion of the erase, program, and verify sequence.                                                                                                |
|   10 | Power-cycle the board.                                                                                                         | The FPGA should now boot from flash automatically, without requiring a fresh JTAG download.                                                                                    |
|   11 | Observe the post-boot behavior.                                                                                                | The same simple test behavior used in C3 should reappear automatically after power-up.                                                                                         |

**Acceptance criteria:**

* The configuration memory device can be added successfully in Vivado.
* The correct `.mcs` or `.bin` file can be selected.
* Flash erase, programming, and verification complete successfully.
* After power-cycle, the FPGA boots autonomously from the programmed flash image.
* The expected post-boot indicator (`DONE`, LED blink, simple test behavior) appears without a manual JTAG download.

**Failure interpretation:**

| Symptom                                                                   | Meaning                                                                                      | Probable causes                                                                                  |
| ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Vivado cannot add or identify the configuration memory device             | The flash type is not selected correctly or the indirect programming path cannot be prepared | wrong flash model selection, unsupported memory, power issue                                     |
| Vivado starts the operation, but erase, programming, or verification fails | JTAG access to FPGA exists, but the FPGA-to-flash path is not working reliably               | SPI flash connectivity, `CCLK`, `CS`, `D0`, `D1`, flash power, wrong memory selection            |
| JTAG programming works, but autonomous boot still fails after power-cycle | FPGA can be configured manually, but the boot chain from flash is not correct                | flash not actually programmed, wrong boot image, wrong mode straps, SPI flash connectivity issue |
| Board boots only when JTAG is connected                                   | The volatile JTAG configuration path works, but non-volatile boot does not                   | blank flash, incorrect flash image, boot configuration issue                                     |

## Source notes for this section

* AMD UG908 — Programming FTDI Devices for Vivado Hardware Manager Support: FTDI EEPROM programming for Vivado JTAG recognition; `program_ftdi` utility; `Manufacturer = Xilinx` requirement. ([AMD UG908][amd-program-ftdi])
* FTDI FT2232H Datasheet: external `93LC46/56/66` EEPROM usage; behavior when EEPROM is blank or absent; default VID/PID and no serial number. ([FTDI][ftdi-ft2232h])
* AMD UG908 — Opening a Hardware Target / Programming the Hardware Device: what appears in the Hardware window after target opening; use of Program Device; checking `DONE` status after programming. ([AMD UG908][amd-open-target], [AMD UG908][amd-program-device])
* AMD UG908 — Programming Configuration Memory Devices / Adding a Configuration Memory Device / Programming a Configuration Memory Device: indirect flash programming through FPGA over JTAG; `.mcs` / `.bin` configuration-memory file; Add Configuration Memory Device flow. ([AMD UG908][amd-config-memory], [AMD UG908][amd-add-config-memory], [AMD UG908][amd-program-config-memory])

## References
- [FTDI FT2232H Datasheet][ftdi-ft2232h]
- [AMD UG908 - Programming FTDI Devices for Vivado Hardware Manager Support][amd-program-ftdi]
- [AMD UG908 - Opening a Hardware Target][amd-open-target]
- [AMD UG908 - Connecting to a Hardware Target Using hw_server][amd-hw-server]
- [AMD UG908 - Programming the Hardware Device][amd-program-device]
- [AMD UG908 - Programming Configuration Memory Devices][amd-config-memory]
- [AMD UG908 - Programming a Configuration Memory Device][amd-program-config-memory]
- [AMD UG908 - Adding a Configuration Memory Device][amd-add-config-memory]

[ftdi-ft2232h]: https://ftdichip.com/wp-content/uploads/2020/07/DS_FT2232H.pdf "FT2232H Datasheet"
[amd-program-ftdi]: https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Programming-FTDI-Devices-for-Vivado-Hardware-Manager-Support "AMD UG908 - Programming FTDI Devices for Vivado Hardware Manager Support"
[amd-open-target]: https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Opening-a-Hardware-Target-Using-Tcl-Commands "AMD UG908 - Opening a Hardware Target"
[amd-hw-server]: https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Connecting-to-a-Hardware-Target-Using-hw_server "AMD UG908 - Connecting to a Hardware Target Using hw_server"
[amd-program-device]: https://docs.amd.com/r/2024.1-English/ug908-vivado-programming-debugging/Programming-the-Hardware-Device "AMD UG908 - Programming the Hardware Device"
[amd-config-memory]: https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Programming-Configuration-Memory-Devices "AMD UG908 - Programming Configuration Memory Devices"
[amd-program-config-memory]: https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Programming-a-Configuration-Memory-Device "AMD UG908 - Programming a Configuration Memory Device"
[amd-add-config-memory]: https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Adding-a-Configuration-Memory-Device "AMD UG908 - Adding a Configuration Memory Device"
