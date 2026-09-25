# Bare-Chip-ESP32-Custom-Development-Board-with-8MB-Flash-and-PCB-Antenna
Designed a custom ESP32 development board around the bare chip in EasyEDA: external QSPI flash, CP2102N USB-UART bridge, transistor auto reset, dual buck power (12V/USB-C to 5V to 3.3V) with ideal diode ORing, and an on PCB 2.4 GHz MIFA antenna. The board boots and runs reliably, detects Wi-Fi down to -96 dBm, and runs flash in DIO mode at 40 MHz.

# Bare Chip ESP32 Custom Development Board

> Hardware design and learning notes for a custom ESP32 development board built directly around the bare ESP32 chip, without a module.

**Author:** Devansh Sharma, 1st Year Integrated MSc, NISER Bhubaneswar
**Version:** V2.0

## Overview

This board uses the ESP32 IC directly instead of a precertified module like the WROOM. That means power integrity, RF layout, boot configuration and clock stability all had to be designed from scratch, which makes it a full system level design exercise on a 4 layer PCB.

**Supports:**

1. USB Type C and external VIN (for example 12V) power input
2. Automatic reset and boot control circuit for one click flashing
3. On board PCB trace antenna (MIFA) for 2.4GHz WiFi and Bluetooth
4. External QSPI flash for firmware storage and XIP execution

## ESP32 Core

The ESP32 is not forgiving like simpler MCUs. Its stable operation depends on power integrity, boot configuration, clock stability and PCB layout all working together.

### Power Distribution and Decoupling

The 3.3V supply feeds several internal domains:

<table>
<tr><th>Pin</th><th>Domain</th></tr>
<tr><td>VDD3P3</td><td>Digital core logic</td></tr>
<tr><td>VDD3P3_RTC</td><td>RTC subsystem</td></tr>
<tr><td>VDD_SDIO</td><td>Flash interface</td></tr>
</table>

Each pin is decoupled individually with **100nF** for high frequency noise and **1 to 10µF** for transient current. Capacitors sit as close as possible to the VDD pins with short vias to the ground plane. Without this, WiFi current spikes cause voltage dips that trigger the brownout detector.

### Enable (EN) Pin

EN must be HIGH for normal operation. It is pulled up with a 10kΩ resistor, filtered with a 100nF capacitor to GND for a stable power up delay, and controlled through RTS for auto reset.

### Boot Strapping

<table>
<tr><th>GPIO0 State</th><th>Boot Mode</th></tr>
<tr><td>LOW</td><td>Flashing / programming mode</td></tr>
<tr><td>HIGH</td><td>Normal execution</td></tr>
</table>

GPIO0 is pulled HIGH by default and driven LOW through DTR during upload. Floating strapping pins lead to boot failures and unreliable programming.

### Clock System

**40MHz Crystal (Main Clock):** mandatory for CPU, peripherals and RF. It is placed right next to the ESP32 crystal pins with symmetric routing on both pins, a solid GND plane below and no signal routing underneath. The load capacitors are **8.2pF**, calculated from the specific crystal's datasheet.

**32.768kHz Crystal (RTC):** optional, improves sleep accuracy and timekeeping. Uses 22pF load capacitors.

### VDD_SDIO

This supply is generated internally by the ESP32 to power the external flash. It is routed directly to the flash VCC with local decoupling at both ends. Since code runs directly from flash, this is treated as a **boot critical power domain**.

### Grounding and Thermal Pad

The exposed thermal pad of the QFN package is stitched to the internal ground plane through a via array. It acts as the heat path, the central ground reference and the noise return path. A solid GND plane is kept under the whole chip.

## Flash Interface

The bare ESP32 has no internal flash, so all firmware lives on an external **W25Q64JVSFIQ** (64Mbit / 8MB, QSPI) connected through CLK, CS# and IO0 to IO3.

**Why it is boot critical:** the ESP32 uses XIP (Execute In Place), meaning instructions are fetched from flash in real time. Any instability on this bus crashes the processor.

**Routing rules followed:**

1. Flash placed as close as possible to the ESP32 on the same side
2. Short and direct routing for all SPI lines
3. No vias on the CLK line
4. Matched trace lengths on CLK and data lines to minimise skew
5. 100nF local decoupling on VCC
6. Test points on CLK, CS and IO lines for oscilloscope and logic analyser bring up

## USB to UART Interface

**IC:** CP2102N (Silicon Labs, 20 pin QFN)

It bridges USB and UART for serial communication, firmware upload and debug logging.

**Signal chain:** USB Type C → ESD protection (USBLC6) → 22Ω series resistors → CP2102N → ESP32

The ESD diode must come before the IC, and the series resistors match the line to about 90Ω differential. CP2102N TX connects to ESP32 RX and vice versa.

**Key pins:**

1. Powered from the 3.3V rail with 100nF + 4.7µF decoupling
2. VBUS sensed through a 22.1kΩ / 47.5kΩ divider for USB detection
3. RSTb pulled HIGH through 2kΩ
4. TX and RX status LEDs
5. Test points on DTR and RTS

## Auto Reset and Boot Control

Two **SS8050** NPN transistors translate the DTR and RTS signals from the CP2102N into controlled pull downs on GPIO0 and EN.

<table>
<tr><th>Signal</th><th>Controls</th></tr>
<tr><td>DTR</td><td>GPIO0 (boot mode)</td></tr>
<tr><td>RTS</td><td>EN (reset)</td></tr>
</table>

**Upload sequence:**

1. DTR pulls GPIO0 LOW
2. RTS toggles EN to reset the chip
3. ESP32 enters the bootloader
4. Both lines return HIGH and the new firmware runs

10kΩ pull ups hold EN and GPIO0 HIGH by default, and 10kΩ base resistors limit transistor base current.

## Power Block

A dual input, multi stage architecture produces a clean 3.3V rail from either USB Type C or external VIN.

### Power Flow

```
USB Type C (5V) → Polyfuse → Ferrite Bead ────────────→ USB_5V ─┐
                                                                ├→ LM66100 ×2 (Ideal Diode OR) → 5V_rail → TPS62162 Buck → 3.3V_rail
External VIN (12V) → SS14 → SMBJ15A TVS → TPS62163 Buck → 5Vin ─┘
```

### USB Type C Path

Polyfuse for overcurrent protection, a 600Ω at 100MHz ferrite bead to block cable noise, 100nF + 4.7µF decoupling, and **5.1kΩ resistors on CC1 and CC2**, which are mandatory for the host to supply power.

### External VIN Path

SS14 Schottky for reverse polarity protection, SMBJ15A TVS for transient and ESD protection, then a TPS62163 buck with a 2.2µH inductor to generate 5V. 47µF bulk capacitors at input and output absorb WiFi current spikes.

### Power Selection

Two **LM66100** ideal diodes perform power ORing. They automatically select the higher 5V source with near zero voltage drop and prevent back feeding between sources.

### 3.3V Rail

A **TPS62162** buck converts 5V to 3.3V using a 2.2µH inductor and 47µF + 22µF + 100nF output filtering. A buck was chosen over an LDO for higher efficiency and better handling of WiFi transient loads.

### PCB Power Distribution

Layer 2 is a continuous GND plane and Layer 3 is a dedicated 3.3V power plane. Test points are provided on 5V_rail, 3.3V_rail, GND and VIN.

## PCB Antenna

**Type:** Meandered Inverted F Antenna (MIFA) for 2.4GHz WiFi and Bluetooth.

It is fed from the ESP32 RF pin through a pi matching network (targeting 50Ω), and the silkscreen over the antenna was removed before fabrication to expose the copper.

**Layout rules followed:**

1. No copper on any layer beneath the antenna, to prevent detuning
2. GND and power planes stop at the antenna feed boundary
3. A via fence along the boundary for a stable RF reference
4. Antenna placed at the board edge, away from digital circuitry

## Design Decisions

<table>
<tr><th>Decision</th><th>Reason</th></tr>
<tr><td><b>Buck over LDO for 3.3V</b></td><td>Higher efficiency and better handling of WiFi current spikes, reducing brownout risk</td></tr>
<tr><td><b>CP2102N for USB to UART</b></td><td>Follows the official ESP32 reference design</td></tr>
<tr><td><b>PCB trace antenna</b></td><td>RF layout learning exercise, lower cost, acceptable 2.4GHz performance</td></tr>
<tr><td><b>LM66100 over Schottky</b></td><td>Near zero voltage drop and active back feed prevention</td></tr>
<tr><td><b>External flash</b></td><td>Not a choice, since the bare ESP32 has no internal flash</td></tr>
</table>

## Components

<table>
<tr><th>Ref</th><th>Part</th><th>Function</th></tr>
<tr><td>U7</td><td>ESP32</td><td>Dual core LX6 MCU with WiFi and Bluetooth</td></tr>
<tr><td>U9</td><td>W25Q64JVSFIQ</td><td>64Mbit QSPI NOR flash</td></tr>
<tr><td>U8</td><td>CP2102N (QFN20)</td><td>USB to UART bridge</td></tr>
<tr><td>U2</td><td>TPS62163DSGR</td><td>Buck regulator, VIN to 5V</td></tr>
<tr><td>U1</td><td>TPS62162DSGT</td><td>Buck regulator, 5V to 3.3V</td></tr>
<tr><td>U3, U4</td><td>LM66100DCKR</td><td>Ideal diode controllers</td></tr>
<tr><td>D1</td><td>USBLC6</td><td>USB ESD protection</td></tr>
<tr><td>D2</td><td>SMBJ15A</td><td>TVS diode on VIN</td></tr>
<tr><td>D3</td><td>SS14</td><td>Reverse polarity protection</td></tr>
<tr><td>Q1, Q2</td><td>SS8050</td><td>Auto reset transistors</td></tr>
<tr><td>X3</td><td>40MHz Crystal</td><td>Main system clock</td></tr>
<tr><td>X1</td><td>32.768kHz Crystal</td><td>RTC clock</td></tr>
<tr><td>F1, F2</td><td>Polyfuse</td><td>Overcurrent protection</td></tr>
<tr><td>L2</td><td>Ferrite Bead</td><td>USB VBUS noise filtering</td></tr>
<tr><td>R1, R2</td><td>5.1kΩ</td><td>USB Type C CC resistors</td></tr>
<tr><td>C34, C35</td><td>270pF</td><td>SENSOR_VP / SENSOR_VN filtering</td></tr>
</table>

## Key Concepts

**Brownout Reset:** the ESP32 resets itself if the supply dips below about 2.5 to 2.7V, even momentarily. WiFi bursts can cause such dips if decoupling is weak.

**XIP (Execute In Place):** instructions run directly from external flash without being copied to RAM, so the flash bus is part of the execution path.

**USB to UART Conversion:** USB uses differential, packet based signalling on D+ and D−, while UART uses simple TX/RX voltage levels. The CP2102N contains a USB engine and a UART engine that translate between the two, so the PC sees a virtual COM port and the ESP32 sees plain UART.

## Results

<table>
<tr><th>Test</th><th>Outcome</th></tr>
<tr><td>Power up</td><td>Works reliably, with none of the wake up issues seen on an earlier WROOM 32D build</td></tr>
<tr><td>Manual boot and reset</td><td>Works like a standard ESP32 board</td></tr>
<tr><td>WiFi reception</td><td>Detects networks from −7 dBm to −96 dBm</td></tr>
<tr><td>Flash mode</td><td>Stable at DIO 40MHz, unstable at QIO 80MHz</td></tr>
<tr><td>Auto reset</td><td>Not functional</td></tr>
</table>

**Known issues:**

1. The 20 pin CP2102N has no DTR pin, and none of its GPIOs can be configured as DTR in Simplicity Studio, so programming currently needs the manual BOOT and EN buttons.
2. QIO at 80MHz is unstable, which shows the flash routing can be improved.

## Version History

<table>
<tr><th>Version</th><th>Changes</th></tr>
<tr><td>V1.0</td><td>Initial schematic. DTR and IO0 nets were wrongly tied to the same 10kΩ pull up in the auto reset circuit.</td></tr>
<tr><td>V2.0</td><td>Fixed the V1.0 bug and completed the PCB layout. Traces use 5mil near the ESP32 and widen to 6mil or more wherever space allows. Silkscreen removed over the antenna.</td></tr>
</table>

## References

1. [ESP32 DevKitC V4 Official Schematic](https://dl.espressif.com/dl/schematics/esp32_devkitc_v4-sch.pdf)
2. [ESP32 WROOM 32 Datasheet](https://documentation.espressif.com/esp32-wroom-32_datasheet_en.pdf)
3. [TPS62160 Family Datasheet](https://www.ti.com/lit/ds/symlink/tps62160.pdf)
4. [LM66100 Datasheet](https://www.ti.com/lit/ds/symlink/lm66100.pdf)
5. [CP2102N Datasheet](https://www.silabs.com/documents/public/data-sheets/cp2102n-datasheet.pdf)
6. [USBLC6 Datasheet](https://www.st.com/resource/en/datasheet/usblc6-2.pdf)
7. [W25Q Series Flash Datasheet](https://www.mouser.com/datasheet/2/949/w25q128jv_revf_03272018_plus1489608.pdf)
8. Other component datasheets sourced from LCSC through EasyEDA
