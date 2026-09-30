# Datasheet

## CanBus

### ESPHome

* [CAN Bus](https://esphome.io/components/canbus/)
* [ESP32 CAN](https://esphome.io/components/canbus/esp32_can/)
* [MCP2515](https://esphome.io/components/canbus/mcp2515/)

### Kabling

* [Kabling med STP for CanBus](./Kabeling/BrugSTPkabeltilCanBus.md)
* [3D printet Box for CanBus moduler](./Kabeling/3D-printetBox-for-CanBus_moduler.md)

### Some PCB

* [ESP32 CAN Bus Shield (v1.0)](https://store.mrdiy.ca/p/esp32-can-bus-shield-v1-0/)
  * [Schematic](./Modules/CanBus/ESP32_CAN_shield_schematic-1024x725.png)

## KiCad

### Track impedans

* [Sierra Circuits - Impedance](https://impedance.app.protoexpress.com/)
* [How to design 90 ohm differential traces in KiCad - Impedance](https://www.youtube.com/watch?v=ABJs4LKFSbA&t=233s)
* [How to design 90 ohm differential traces in KiCad - Ground plane clearance](https://www.youtube.com/watch?v=ABJs4LKFSbA&t=717s)
* [pcbway multi-layer-laminated-structure](https://www.pcbway.com/multi-layer-laminated-structure.html)

### Custom Symbol and Footprints

* [How to Create Custom KiCad Symbol and Footprints](https://youtu.be/xpBpxipfXFA?list=PLhEL_BCoP-PsyjJb1Pl8QnrqaBn_YuJ7R "DIY Hideout")
  * [Symbols](https://youtu.be/xpBpxipfXFA?list=PLhEL_BCoP-PsyjJb1Pl8QnrqaBn_YuJ7R&t=8 "DIY Hideout")
  * [Footprints](https://youtu.be/xpBpxipfXFA?list=PLhEL_BCoP-PsyjJb1Pl8QnrqaBn_YuJ7R&t=219 "DIY Hideout")
  * [Linking Symbol with Footprint](https://youtu.be/xpBpxipfXFA?list=PLhEL_BCoP-PsyjJb1Pl8QnrqaBn_YuJ7R&t=411 "DIY Hideout")
  * [Preview Symbol & Footprint](https://youtu.be/xpBpxipfXFA?list=PLhEL_BCoP-PsyjJb1Pl8QnrqaBn_YuJ7R&t=438 "DIY Hideout")
* [FreeCAD Export to KiCAD](https://youtu.be/JjDKCBUYoPU "mathcodeprint")

## Parts

### Semiconductor

* Optocobler
  * [PC847 - High Density Mounting Type Photocoupler](./Semiconductor/PC847.pdf)
* Diode
  * [S5AC - 5.0A SURFACE MOUNT GLASS PASSIVATED RECTIFIER](./Semiconductor/ds16007.pdf)
* CAN BUS
  * [NUP2105 - Dual Line CAN Bus Protector](./Semiconductor/NUP2105L-D.PDF)
  * [SN65HVD230 - 3.3-V CAN Bus Transceivers](./Semiconductor/sn65hvd230.pdf)

### Modules

* CANBUS
  * [CAN til TTL transceiver-modul med SN65HVD230](https://let-elektronik.dk/can-transceiver-modul-sn65hvd230)
* MCU
  * [ESP32-DOIT-DEVKIT-V1-Board-Pinout-30-GPIO.png](./Modules/MCU/ESP32-DOIT-DEVKIT-V1-Board-Pinout-30-GPIO.png)
    * [ESP32 DEVKIT V1 - DOIT 30 GPIOs - Pinout](./Modules/MCU/ESP32-DOIT-DEVKIT-V1-Board-Pinout-30-GPIO.webp)
    * [ESP32.FCStd](./Modules/MCU/ESP32.FCStd)
      * [ESP32.step](./Modules/MCU/ESP32.step)
  * [ESP32_38Pin](./Modules/MCU/ESP32_38Pin.png)
    * [ESP32_38Pin Pinout](./Modules/MCU/ESP-32_38_Pin_diagram_480x480.webp)
      * [ESP32_38Pin FCStd](./Modules/MCU/ESP32_38Pin.FCStd)
      * [ESP32_38Pin step](./Modules/MCU/ESP32_38Pin.step)
* DC-DC
  * [Mini560Pro - Step Down DC-DC](./Modules/Mini560Pro/Mini560Pro.pdf)
    * [Mini560Pro.FCStd](./Modules/Mini560Pro/Mini560Pro.FCStd)
    * [Mini560Pro.png](./Modules/Mini560Pro/Mini560Pro.png)
    * [Mini560Pro-BodyPad003.step](./Modules/Mini560Pro/Mini560Pro-BodyPad003.step)
      * [Mini560Pro_3D.png](./Modules/Mini560Pro/Mini560Pro_3D.png)

### Stik

* [PC_Test_Point-Minature.png](./Stik/TestPoint/PC_Test_Point-Minature.png)
  * [PC Test Point - Datasheet](./Stik/5000-5004.PDF)
  * [PC_Test_Point-Minature.FCStd](./Stik/TestPoint/PC_Test_Point-Minature.FCStd)
  * [PC_Test_Point-Minature.step](./Stik/TestPoint/PC_Test_Point-Minature.step)
* [RJ45_RJ064-LCP](./Stik/RJ45_RJ064-LCP/RJ45_RJ064-LCP_Pinout.png)
* [Skrueterminal 5,08mm](./Stik/SkrueTerminal/Skrueterminal_508.png)
  * [Skrueterminal_508.FCStd](./Stik/SkrueTerminal/Skrueterminal_508.FCStd)
  * [Skrueterminal_508.step](./Stik/SkrueTerminal/Skrueterminal_508.step)
* [KF2510_5_90.png](./Stik/Molex/KF2510_5_90.png)
  * [KF2510_5_90.FCStd](./Stik/Molex/KF2510_5_90.FCStd)
  * [KF2510_5_90.step](./Stik/Molex/KF2510_5_90.step)
