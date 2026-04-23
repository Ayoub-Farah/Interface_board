# 1. Introduction

## General Principle

The test sequence is strictly gated:

```text
→ Inspection / short-circuit check
→ Power supply only
→ Internal FPGA rails
→ Reset / power-good validation
→ USB / JTAG / FPGA configuration
→ Clocks and PLL
→ DAC + 25 MHz oscillator + 125 MHz PLL
→ SFP electrical / I2C / control checks
→ GTP / Aurora / optical link
```

Basic rule: do not move to the next step until the current step has been validated. In particular, no SFP, no external MCU, and no other module in the ring should be connected before the power supply and minimal FPGA operation have been fully validated.

## Required Test Equipment

| Equipment                                  | Use                                               |
| ------------------------------------------ | ------------------------------------------------- |
| 12 V bench power supply with current limit | Progressive bring-up, short-circuit detection     |
| Digital multimeter                         | Cold resistance checks, DC voltage measurements   |
| Oscilloscope                               | Rails, reset, SPI, I2C, 25 MHz clock              |
| Differential probe                         | MGT reference / 125 MHz if accessible             |
| Logic analyzer                             | MCU SPI, I2C, debug signals                       |
| PC with Vivado Hardware Manager            | JTAG, programming, IBERT, FPGA debug              |
| USB-C cable                                | FT2232 / JTAG / serial interface                  |
| SFP modules                                | Optical communication test                        |
| Optical fiber or SFP loopback              | Optical loopback test                             |
| Second Obsidian board                      | Board-to-board optical link test                  |
| Thermal camera                             | Locating hot components during overcurrent        |

## Useful Existing Test Points

The main test points are:

| Test Point               | Function                                 |
| ------------------------ | ---------------------------------------- |
| `TP1`                    | USB / FT2232 supply                      |
| `TP2`                    | I2C SCL bus                              |
| `TP3`                    | I2C SDA bus                              |
| `TP4`                    | Ethernet PHY interrupt / PMEB            |
| `TP5`                    | Global reset                             |
| `TP6`                    | Aggregated power-good / power diagnostic |
| `TP7`                    | 1.8 V rail                               |
| `TP8`                    | DDR rail                                 |
| `TP9`                    | Main 3.3 V rail                          |
| `TP10`                   | Peripheral 3.3 V rail                    |
| `TP11`                   | FPGA core 1.0 V                          |
| `TP12`                   | 1.2 V rail                               |
| `TP13`                   | 2.5 V rail                               |
| `TP14`                   | Power-good / diode node                  |
| `TP15 / TP23`            | External sync / debug                    |
| `TP16 / TP24`            | SPI MOSI                                 |
| `TP17 / TP25`            | SPI SCK                                  |
| `TP18 / TP26`            | SPI CS                                   |
| `TP19 / TP27`            | SPI MISO                                 |
| `TP20-TP22`, `TP28-TP32` | Ground references                        |
| `TP33`                   | MGT 1.0 V rail                           |
