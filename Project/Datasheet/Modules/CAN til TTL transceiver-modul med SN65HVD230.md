# CAN til TTL transceiver-modul med SN65HVD230

Kilde: [let-elektronik.dk](https://let-elektronik.dk/can-transceiver-modul-sn65hvd230)

## Beskrivelse

* Dette CAN til TTL transceiver-modul er det fysiske bindeled mellem en mikrocontrollers indbyggede CAN-controller og en rigtig CAN-bus. Det omsætter de logiske CAN_TX / CAN_RX-signaler til de differentielle CAN-H / CAN-L-niveauer, der kører på bussen – og den anden vej rundt.
* Modulet er bygget op omkring transceiveren SN65HVD230 (chippen er mærket VP230, som er mærkningen for SN65HVD230 i SOIC-8; de øvrige linjer 83M og AFC3 er dato- og lotkoder). SN65HVD230 er en af de mest brugte 3,3 V CAN-transceivere. På modulet sidder desuden en XC6206 3,3 V spændingsregulator, så modulet kan forsynes med både 3,3 V og 5 V, samt en indbygget 120 Ω termineringsmodstand.
* Modulet er ideelt til MCU'er, der allerede har en CAN-controller indbygget – fx ESP32 (TWAI), STM32 (bxCAN/FDCAN i klassisk CAN-tilstand) og lignende. Har din mikrocontroller ikke en CAN-controller (fx Arduino Uno eller Raspberry Pi), så skal du i stedet bruge et modul med både controller og transceiver, fx et MCP2515-modul.

## Egenskaber

* Kompatibel med ISO 11898-2 (high-speed CAN)
* Hastigheder op til 1 Mbit/s
* Forsyning 3,3-5 V via indbygget 3,3 V regulator (XC6206)
* Indbygget 120 Ω terminering mellem CAN-H og CAN-L (R3)
* Høj indgangsimpedans – op til 120 noder på samme bus
* ±16 kV ESD-beskyttelse (HBM) på busbenene
* Busbenene tåler fejlspændinger fra -4 V til +16 V, common mode-område -2 V til +7 V
* Termisk nedlukning og open-circuit fail-safe
* En node uden strøm forstyrrer ikke bussen, og chippen er beskyttet mod glitches ved tilslutning under drift (hot-plug)

# Spændinger – sådan bruger du modulet

* **Forsyning (VCC): 3,3-5 V.** Selve SN65HVD230 kører på 3,3 V, men modulets indbyggede XC6206-regulator sørger for det, så VCC kan tilsluttes enten 3,3 V eller 5 V. Ved 3,3 V forsyning får chippen lidt under 3,3 V pga. regulatorens spændingsfald – det er helt normalt, da chippen fungerer ned til 3,0 V.
* **Logikniveau (TX/RX): 3,3 V.** Uanset forsyningsspænding arbejder TX og RX på 3,3 V-niveau, fordi de går direkte til transceiver-chippen.
* **3,3 V mikrocontroller (ESP32, STM32, Teensy 4 m.fl.):** Forbind direkte. VCC til 3,3 V (eller 5 V), GND til GND, TX til MCU'ens CAN_TX og RX til CAN_RX. Ingen niveauomsætning nødvendig.
* **5 V mikrocontroller med indbygget CAN:** VCC kan tages direkte fra 5 V. RX-udgangen giver ca. 3,3 V som højt niveau, hvilket de fleste 5 V MCU'er læser som logisk 1 – men tjek din MCU's datablad. TX-signalet fra en 5 V MCU må ikke gå direkte ind i modulet; brug en spændingsdeler (fx 1 kΩ / 2 kΩ) eller en niveauomsætter, så TX max er 3,3 V.
* **Blandet bus (3,3 V og 5 V noder):** På bus-siden overholder SN65HVD230 ISO 11898-2 og kommunikerer uden problemer med noder, der bruger 5 V transceivere som TJA1050, MCP2551 eller PCA82C250 – det er den differentielle spænding mellem CAN-H og CAN-L, der tæller, ikke forsyningsspændingen.
* **Køretøjer og 12/24 V anlæg:** CAN-bussen i fx en bil kører på de samme differentielle niveauer, så modulet kan kobles på bussen. Forsyningen skal dog tages fra 3,3 V eller 5 V – aldrig direkte fra 12 V. Busbenene tåler op til +16 V ved fejl, så de er ikke beskyttet mod kortslutning til 24 V.

## Terminering – 120 Ω

* På MCU-siden forbindes **VCC** (3,3-5 V), **GND, TX** (CAN_TX) og **RX** (CAN_RX). På bus-siden sidder en **3-polet skrueterminal** til **H** (CAN-H), **L** (CAN-L) og **stel**. Forbind stel mellem noderne, så de har fælles reference.

## Specifikationer

|||
|:---|:---|
|Transceiver-chip|SN65HVD230 (mærket VP230), SOIC-8|
|Spændingsregulator|XC6206, 3,3 V LDO|
|Type|CAN-transceiver (CAN-bus ↔ CAN_TX/RX logikniveau)|
|Standard|ISO 11898-2 (high-speed CAN)|
|Hastighed|Op til 1 Mbit/s|
|Forsyning|3,3-5 V|
|Logikniveau TX/RX|3,3 V|
|Terminering|120 Ω indbygget (R3, kan fjernes)|
|Antal noder|Op til 120 på samme bus|
|ESD-beskyttelse|±16 kV HBM på busbenene|
|Busfejlspænding|-4 V til +16 V|
|Til brug med|MCU med indbygget CAN-controller (ESP32, STM32 m.fl.)|
|Bus-tilslutning|3-polet skrueterminal: H, L, stel|
|MCU-tilslutning|VCC, GND, TX, RX|
|Boardmål|40,7 × 13,76 mm|

Bemærk: Dette modul indeholder kun en CAN-transceiver – ikke en CAN-controller.