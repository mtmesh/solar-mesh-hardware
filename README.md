# LoRa Mesh* Solar MPPT Charger

An open-source solar MPPT charge controller for unattended Mesh* node deployments. Integrates solar charging, battery protection, power monitoring, and environmental sensing with the RAK4630 LoRa module and fits in the RAK Unify solar enclosue with space for a battery sled, notch for N-type bulkhead and vent.

![Full Package Render](badc5f8c-f985-4a23-8027-aa8029b5097e.PNG)

**Current Revision:** K v1.2  

## Why This Exists

**The inspration:** MSPMesh posted a cool idea for a [RAK Unify 150 node build](https://mspmesh.org/rak-unify-150-node-build/), and from there it was about building a tight package to deliver a similar outcome with a little more control of the components and features.

**The enclosure:** RAK Wireless makes the [Solar Unify Enclosure](https://store.rakwireless.com/products/unify-enclosure-ip67-150x100x45mm-with-pre-mounted-solar-panel) (IP67, 150×100×45mm) with a built-in 5V solar panel. This board fits in the top section, with space for a 3× 18650 battery sled in the lower section. Add a 3dBi N-Male antenna on top and a vent at the bottom, and you have a complete weatherproof solar node with ~10,000mAh capacity.

**The problem:** nRF52-based Mesh* boards (like the RAK4630) have a firmware issue ([#4378](https://github.com/Mesh*/firmware/issues/4378)) where they cannot reliably enter deep sleep at low battery voltages. This causes a destructive cycle:

1. Battery drops to ~3.0V → board attempts sleep
2. Board freezes or browns out
3. Watchdog resets → LoRa TX spike → voltage sag → brownout
4. Repeat until battery is damaged

Standard charge controllers with ~100mV hysteresis make this worse. This board implements 900mV hysteresis (3.1V cutoff, 4.0V release) so the battery has enough stored energy for reliable cold starts.

## Features

![Board Render](Render.png)

- BQ24650 synchronous buck MPPT charger (90-95% efficiency)
- 5V to 20V panel support via jumper selection
- LTC1540 ultra-low-power UVLO comparator (~1.4µA quiescent when tripped)
- INA3221 3-channel power monitor (solar + battery)
- BME280 environmental sensor (temperature, humidity, pressure)
- TPS63000 buck-boost regulator (3.3V output)
- USB-C for power and firmware updates
- Dual I2C buses (power monitoring separate from sensors)
- Two Qwiic/STEMMA QT expansion ports
- Fits RAK Solar Enclosure IP67 (85×45mm)

## Specifications

| Parameter | Value |
|-----------|-------|
| Input Voltage (Solar) | 5V – 20V Vmpp |
| Max Charge Current | 2A |
| Output Voltage | 3.3V regulated |
| UVLO Cutoff (Li-Ion) | 3.12V ±50mV |
| UVLO Release (Li-Ion) | 4.00V ±100mV |
| Hysteresis | 880mV |
| Quiescent (UVLO tripped) | ~1.4µA |
| Switching Frequency | 600kHz |

## Schematic

[Schematic](SchematicRevK.pdf)

## Supported Configurations

### Solar Panels

| Jumper | Panel Vmpp | Notes |
|--------|------------|-------|
| JP11 (default) | 5V | HX-140X90 from RAK Unify enclosure |
| JP10 | 12V | |
| JP9 | 18V | |
| JP8 | 20V | |
| JP7 | Custom | User-defined resistor |

Two identical panels can be wired in parallel for increased current. Fuses are rated for 750mA hold to support dual 5V panels (~760mA combined).

### Battery Chemistry

| Jumper | Chemistry | Charge Voltage | UVLO |
|--------|-----------|----------------|------|
| JP6 (default) | 1S/2P Li-Ion | 4.20V | External (LTC1540) |
| JP5 + JP3 | 2S LTO | 5.40V | Internal (TPS63000) |
| JP4 + JP3 | 1S LiFePO4 | ~3.6V | Internal (TPS63000) |

### UVLO modes

External (LTC1540): Dedicated comparator with 900mV hysteresis (3.1V cutoff, 4.0V release). Used for Li-Ion to prevent the brownout loop described above.
Internal (TPS63000): The buck-boost regulator's built-in UVLO with 200mV hysteresis (1.5V cutoff, 1.7V release). Bridging JP3 bypasses the LTC1540 circuit. Required for LTO and LiFePO4 because their voltage ranges are incompatible with the 4.0V release threshold.

**Recommended:** 3× 18650 cells in parallel (1S/3P). ~10000mAh capacity, 7-9 days runtime at typical Mesh* loads.  BOM includes an example [Adafruit Lithium Ion Battery - 3.7V 10050mAh](https://www.adafruit.com/product/5035) with the correct PH connector, which fits into the sled below and has worked very well in trials.

### 3D Printed Battery Sleds

- [STL file for 3x18650 cell pack](Unify_3x18650_Battery_Sled.stl)


## I2C Addresses

| Bus | Address | Device |
|-----|---------|--------|
| I2C1 | 0x42 | INA3221 |
| I2C2 | 0x77 | BME280 (onboard, default) |
| I2C2 | 0x76 | BME280 (external via Qwiic) |

The INA3221 address 0x42 is required for Mesh* firmware compatibility.

## Quick Start

### Default Configuration

The board is fabricated with JP11 anmd JP6 bridged for:
- 5V panel (JP11 bridged)
- Li-Ion chemistry (JP6 bridged)
- BME280 at 0x77 (JP1 position A)

No changes needed for standard HX-140X90 panel + 18650 battery setups.  Other choices need you to break the fabricated links and solder the JP* of choice.

### First Power-Up

Li-Ion cells usually ship at 30-50% charge (~3.6-3.8V). With a 4.0V release threshold, the board won't start on a new battery alone.

**USB Bootstrap:**
1. Connect USB-C — system powers up via D7 Schottky diode
2. Connect solar panel — BQ24650 charges battery
3. Wait for battery to reach 4.0V — UVLO releases
4. Disconnect USB — system continues on battery

**Solar-only deployment:**
1. Install battery and connect solar
2. Battery charges with system load disconnected (UVLO keeps it off)
3. System auto-starts when battery reaches 4.0V

## Jumper Reference

### Panel Selection (bridge ONE)

| Jumper | Vmpp |
|--------|------|
| JP11 | 5V (default fabrication bridge) |
| JP10 | 12V |
| JP9 | 18V |
| JP8 | 20V |
| JP7 | Custom |

### Battery Chemistry (bridge ONE)

| Jumper | Chemistry |
|--------|-----------|
| JP6 | Li-Ion (default fabrication bridge) |
| JP5 | 2S LTO |
| JP4 | Custom |

### Other Jumpers

| Jumper | Function | Default |
|--------|----------|---------|
| JP1 | BME280 address | 0x77 (default fabrication bridge) |
| JP3 | UVLO bypass (for LTO/LiFePO4) | Open |

## Connectors

| Ref | Type | Function |
|-----|------|----------|
| J1 | ZH1.5-2P | Solar input (matches HX-140X90 in the Unify 150 case) |
| J6 | Screw terminal | Solar input (alt) |
| J2, J3 | JST-PH | Battery |
| J7 | Screw terminal | Battery (alt) |
| J4, J5 | Qwiic | I2C2 expansion |
| J10 | USB-C | Power + data, no charging |
| J8 | 1×04 header | UART debug (4x2.54mm pitch header position) |
| J9 | 1×04 header | SWD programming (4x2.54mm pitch header position) |

## Mesh* Configuration

```yaml
telemetry:
  environment_measurement_enabled: true
  power_measurement_enabled: true
  environment_update_interval: 900
  power_update_interval: 900

power:
  device_battery_ina_address: 66  # 0x42
```

## Optional Components

Not every node needs every feature. The board is designed as a common platform for Mesh* deployments—populate what you need, skip what you don't.

| Component | Function | Skip if... |
|-----------|----------|------------|
| U1 (BME280) + R1, R2, R3 | Temperature, humidity, pressure | Node doesn't need environmental data |
| U3 (INA3221) + R_shunt1, R_shunt2 | Solar/battery current monitoring | You only need basic voltage sensing (AIN0 still works), but you need to solder 0Ω bridges |
| J4, J5 (Qwiic) | I2C expansion ports | No external sensors planned |
| J6, J7 (screw terminals) | Solar/battery screw connections | Soldering wires directly to tht pads |
| J2, J3 (JST-PH) | Battery connectors | Using screw terminals or soldering |
| J1 (ZH1.5) | Solar connector | Using screw terminals or soldering |
| J8, J9 (debug headers) | UART/SWD access | Not doing firmware development |
| D2, D3, Q1, Q3, R12, R14 | GPIO indicator LEDs | Don't need visual status beyond charge LEDs |

**Minimum viable build:** Core charging (BQ24650, LTC1540, TPS63000, protection FETs), charge status LEDs (D5, D6), and the RAK4630. Solder battery and panel wires directly. This gets you a working solar-charged Meshtastic node without the monitoring and expansion features.

**Typical build:** Add INA3221 for power telemetry, bme280 for enviro, ZH for the enclosure inbuilt solar panel connector, and PH on the battery for common packs.  This is the most useful optional feature for remote deployments—knowing your solar harvest and battery state without visiting the site.

**Full build:** Everything populated. Useful for environmental monitoring nodes or when you want maximum flexibility for future sensor additions.

## BOM Notes

- All components available from DigiKey
- Hand assembly via hot plate or oven reflow (no PCBA service required)
- 0603 or larger passives throughout (no 0201) for hand placement
- See full BOM in [DigiKey list](https://www.digikey.com/en/mylists/list/6RO23TL9P5)

## References

- [Mesh* Firmware Issue #4378](https://github.com/Mesh*/firmware/issues/4378) — nRF52 deep sleep failure
- [Voltaic MCSBC-SVR](https://docs.voltaicenclosures.com/mcsbc-svr/) — Reference design for UVLO hysteresis values
- [uart.cz Rev E](https://pcb.uart.cz/) — Initial inspiration
- [BQ24650 Datasheet](https://www.ti.com/product/BQ24650) — MPPT charger
- [LTC1540 Datasheet](https://www.analog.com/en/products/ltc1540.html) — UVLO comparator
- [TPS63000 Datasheet](https://www.ti.com/product/TPS63000) — Buck-boost regulator

## License

[GNU GPL v3 license](LICENSE)

## Acknowledgments

- MSPMesh [RAK Unify 150 node build](https://mspmesh.org/rak-unify-150-node-build/)
- Vlastimil Slinták — uart.cz base design
- YYCMesh community — Cold weather field testing
- Austin Mesh, Mesh Coordinators, Mesh* and MeshCore Discords — Feedback and validation
