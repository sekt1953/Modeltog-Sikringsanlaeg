# CAN-transceiver-modul med SN65HVD230 (VP230)

Kort modulnote til brug i eget projekt.
Fuld beskrivelse, billeder og køb: [let-elektronik.dk/can-transceiver-modul-sn65hvd230](https://let-elektronik.dk/can-transceiver-modul-sn65hvd230)

## Hvad modulet gør

Modulet er **kun en transceiver** — det fysiske lag mellem en MCU's CAN-controller og
bussen. Det oversætter CAN_TX/CAN_RX til de differentielle CAN-H/CAN-L-niveauer.
Du skal derfor bruge en MCU med CAN-controller indbygget (ESP32 = TWAI, STM32 = bxCAN/FDCAN).
Arduino Uno og Raspberry Pi har ingen CAN-controller — brug et MCP2515-modul i stedet.

Ombord: SN65HVD230 (mærket **VP230**, SOIC-8), XC6206 3,3 V LDO, 120 Ω terminering (R3).

## Tilslutning

| Modul | Forbindes til | Bemærkning |
|:---|:---|:---|
| VCC | 3,3 V eller 5 V | LDO ombord, så begge virker |
| GND | GND | |
| TX | MCU CAN_TX | **3,3 V logik** — se advarsel nedenfor |
| RX | MCU CAN_RX | afgiver ca. 3,3 V højt niveau |
| H / L / stel | skrueterminal til bussen | træk altid stel med rundt i anlægget |

## Terminering — det der oftest går galt

En CAN-bus skal have 120 Ω i **hver ende**, og kun der. Måler du mellem H og L med
strømmen slukket, skal du se ca. **60 Ω** på en korrekt termineret bus.

* Modul som **endepunkt**: lad R3 sidde.
* Modul **midt på bussen**: lød R3 af. Tre moduler med R3 i giver ca. 40 Ω, og så
  begynder bussen at give fejl — typisk sporadiske error frames, ikke total stilhed,
  hvilket gør det svært at finde.
* To moduler direkte sammen = en korrekt termineret test-bus.

## 3,3 V vs. 5 V — en faldgrube

TX og RX kører på 3,3 V uanset hvad VCC er, fordi de går direkte på transceiveren.

* 3,3 V MCU (ESP32, STM32, Teensy 4): forbind direkte, ingen niveauomsætning.
* 5 V MCU: VCC må gerne være 5 V, og RX læses normalt fint som logisk 1 — men
  **TX fra en 5 V MCU må ikke gå direkte ind i modulet.** Brug spændingsdeler
  (fx 1 kΩ/2 kΩ) eller en niveauomsætter, så modulets TX-ben max ser 3,3 V.
* Blandet bus er intet problem: på bus-siden er niveauerne ISO 11898-2, så modulet
  taler uden videre med 5 V transceivere som TJA1050, MCP2551 og PCA82C250.
* 12/24 V anlæg: bussen kan kobles på, men forsyningen skal komme fra 3,3/5 V —
  aldrig fra 12 V. Busbenene tåler +16 V ved fejl, ikke kortslutning til 24 V.

## Tjekliste når bussen ikke kører

1. Måler du ca. 60 Ω mellem H og L (strøm slukket)? Ellers er termineringen forkert.
2. Er H og L byttet om et sted i anlægget?
3. Er stel trukket med mellem noderne?
4. Kører alle noder samme bitrate?
5. Er der en 5 V node hvis TX går direkte ind i et 3,3 V modul?
6. Sidder TX/RX byttet om mellem MCU og modul?

## ESP32 (ESP-IDF, TWAI) — minimalt eksempel

Tilpas GPIO-numre og bitrate til dit eget anlæg.

```c
twai_general_config_t g = TWAI_GENERAL_CONFIG_DEFAULT(GPIO_NUM_21,  // -> modulets TX
                                                      GPIO_NUM_22,  // <- modulets RX
                                                      TWAI_MODE_NORMAL);
twai_timing_config_t  t = TWAI_TIMING_CONFIG_500KBITS();
twai_filter_config_t  f = TWAI_FILTER_CONFIG_ACCEPT_ALL();

twai_driver_install(&g, &t, &f);
twai_start();
```

## Specifikationer

|||
|:---|:---|
|Transceiver|SN65HVD230 (mærket VP230), SOIC-8|
|Regulator|XC6206, 3,3 V LDO|
|Standard|ISO 11898-2 (high-speed CAN)|
|Hastighed|op til 1 Mbit/s|
|Forsyning|3,3-5 V|
|Logikniveau TX/RX|3,3 V|
|Terminering|120 Ω indbygget (R3, kan fjernes)|
|Antal noder|op til 120|
|ESD|±16 kV HBM på busbenene|
|Busfejlspænding|-4 V til +16 V (common mode -2 V til +7 V)|
|Bus-tilslutning|3-polet skrueterminal: H, L, stel|
|Boardmål|40,7 × 13,76 mm|

---
Modulnoten er skrevet af [Let-Elektronik](https://let-elektronik.dk/can-transceiver-modul-sn65hvd230)
og må frit bruges og tilpasses i dette projekt.
