# ESPHome and TI TCAN33x/TCAN34x family

## esphome TCAN340

* TCAN340 (or similar chips in the Texas Instruments TCAN33x/TCAN34x family) is a 3.3V CAN transceiver. It is fully compatible with ESPHome’s CAN Bus component, specifically paired with an ESP32.
* Because standard ESP32 chips feature an integrated CAN controller (SJA1000 compatible), you do not need an external controller chip (like an MCP2515). You only need the TCAN340 transceiver to bridge the ESP32's logic levels to the physical CAN High (CANH) and CAN Low (CANL) lines.

### Hardware Wiring

Since the TCAN340 operates entirely on 3.3V VCC, it is perfectly suited for the ESP32 without needing any logic-level shifters.

||||
|:---|:---|:---|
|TCAN340 Pin|ESP32 Pin|Notes|
|VCC|3.3V|Power supply|
GND	GND	Ground
|TXD|Any GPIO (e.g., GPIO22)|Connects to ESP32 CAN TX|
|RXD|Any GPIO (e.g., GPIO21)|Connects to ESP32 CAN RX|
CANH|Bus CANH|Network CAN High wire|
CANL|Bus CANL|Network CAN Low wire|

* Note: Ensure your CAN network has a 120-Ohm termination resistor on both physical ends of the bus line.

### ESPHome Configuration Example

* Add the following to your YAML configuration file using the ESPHome CAN Bus Component:

```yaml
canbus:
  - platform: esp32_can
    id: my_can_bus
    tx_pin: GPIO22
    rx_pin: GPIO21
    bit_rate: 125kbps   # Options range from 1kbps to 1000kbps (1Mbps)
    can_id: 1           # Default transmitting ID for this node

# Example: Automating actions when receiving a specific CAN frame
    on_frame:
      - can_id: 0x123
        then:
          - logger.log: "Received frame from ID 0x123!"
```

### Key Configuration Tips

* Bit Rate: Make sure your bit_rate exactly matches the speed of the other nodes on your CAN bus network. The default is 125kbps.
* Extended IDs: If your network uses 29-bit extended frames instead of 11-bit standard frames, add use_extended_id: true under the canbus platform.

What type of device or automotive/industrial system are you planning to connect to this CAN bus network? Knowing this will help provide the right YAML logic for sending or receiving data.

## hvat to do with STB input

* The STB (Standby) pin on the TCAN340 controls the operating mode of the transceiver. To make the CAN bus function under normal ESPHome operations, you must handle this pin correctly.
* You have two choices depending on how you want to manage power:

## Option 1: Always Active (Easiest / Recommended)

If you don't need low-power sleep modes and want the CAN bus to be active all the time, connect the STB pin directly to GND (Ground).
* Logic: A LOW level on the STB pin forces the TCAN340 into Normal Mode, enabling full transmission and reception.

## Option 2: Control via ESPHome (Power Saving)

If you want the ESP32 to put the transceiver into low-power standby mode when it isn't using the bus, connect the STB pin to an available ESP32 GPIO pin (e.g., GPIO5).

You can then control it natively in your ESPHome YAML using a Template Switch or a standard GPIO switch:

```yaml
switch:
  - platform: gpio
    pin: GPIO5
    id: can_transceiver_stb
    inverted: true # Inverted because LOW = Active, HIGH = Standby
    name: "CAN Transceiver Power"
    restore_mode: ALWAYS_ON
```

## What happens if you leave it disconnected?

The STB pin has an internal pull-up resistor. If you leave it disconnected (floating), it will default to HIGH, putting the chip into Standby Mode. In this state, the transceiver cannot transmit data, and your ESPHome node will not be able to communicate on the network.
