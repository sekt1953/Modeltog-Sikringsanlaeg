# Modeltogs Sikringsanlæg

Sikrings anlæg for modeltog med DCC system, ESPHome &amp; Home Assistant

## Datasheet

* CanBus
  * ESPHome
    * [CAN Bus](https://esphome.io/components/canbus/)
    * [ESP32 CAN](https://esphome.io/components/canbus/esp32_can/)
    * [MCP2515](https://esphome.io/components/canbus/mcp2515/)
  * [Kabling](./Project/Datasheet/README.md#kabling)
    * [Kabling med STP for CanBus](./Project/Datasheet/Kabeling/BrugSTPkabeltilCanBus.md)
    * [3D printet Box for CanBus moduler](./Project/Datasheet/Kabeling/3D-printetBox-for-CanBus_moduler.md)
  * KiCad
    * [Track impedans calculater](./Project/Datasheet/TrackImpedansCalculater.md)
      * [How to design 90 ohm differential traces in KiCad - Impedance](https://www.youtube.com/watch?v=ABJs4LKFSbA&t=233s)
      * [How to design 90 ohm differential traces in KiCad - Ground plane clearance](https://www.youtube.com/watch?v=ABJs4LKFSbA&t=717s)
      * [Sierra Circuits - Impedance](https://impedance.app.protoexpress.com/)
      * [pcbway multi-layer-laminated-structure](https://www.pcbway.com/multi-layer-laminated-structure.html)
    * Custom Symbol and Footprints
      * [How to Create Custom KiCad Symbol and Footprints](https://youtu.be/xpBpxipfXFA?list=PLhEL_BCoP-PsyjJb1Pl8QnrqaBn_YuJ7R)
        * [Footprints](https://youtu.be/xpBpxipfXFA?list=PLhEL_BCoP-PsyjJb1Pl8QnrqaBn_YuJ7R&t=219)
  * Semiconductor
    * [NUP2105 - Dual Line CAN Bus Protector](./Project/Datasheet/Semiconductor/NUP2105L-D.PDF)
    * [S5AC - 5.0A SURFACE MOUNT GLASS PASSIVATED RECTIFIER](./Project/Datasheet/Semiconductor/ds16007.pdf)
  * Stik
    * [PC Test Point - Miniature](./Project/Datasheet/Stik/5000-5004.PDF)

## Projekter

* [Sporbesat](./Project/Sikring/Sporbesat/README.md)
  * Detektor kredsløb
    * Transistor version
      * KiCAD
      * FreeCAD
      * ESPHome
      * Home Assistant
  * Transmitterbus
    * Canbus
      * KiCAD
      * FreeCAD
      * ESPHome
      * Home Assistant
    * Modbus
      * KiCAD
      * FreeCAD
      * ESPHome
      * Home Assistant
* Sporskifte styring
* Signal styring
