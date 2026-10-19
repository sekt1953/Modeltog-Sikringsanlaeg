# CanBuss og Home Assistant

## kan home assistant se status på en esp32 gpio som er placeret på en canbus

*Ja, Home Assistant kan godt se status på en ESP32 GPIO, der befinder sig på en CAN-bus. Det kræver dog, at der laves en form for "oversættelse" undervejs.
En GPIO-pind er en lokal hardware-pind, og en CAN-bus er en netværksbus til dataoverførsel. Home Assistant kan ikke læse GPIO-pinden direkte gennem bussen uden kodning, men du kan nemt løse det på to måder, typisk ved hjælp af firmwaren ESPHome.
Her er de to mest almindelige scenarier for, hvordan det sættes op:

### Scenarie A: ESP32'eren sidder fysisk på CAN-bussen og skal sende sin egen GPIO-status

Hvis du har en ESP32 forbundet til CAN-bussen via en transceiver (f.ex. SN65HVD230), og du vil sende status på en af dens egne GPIO-pinde ud til Home Assistant (eller andre enheder) via bussen, gør du følgende i ESPHome:

1. Opret en lokal binary_sensor på GPIO-pinden.
2. Brug canbus-komponenten til at sende en datapakke (et CAN-id og et payload), hver gang pinden ændrer status.

### Scenarie B: En ESP32 modtager GPIO-status fra en anden CAN-enhed

Hvis GPIO-pinden sidder på en helt anden enhed på CAN-bussen (f.eks. en styreboks i en bil, et pillefyr eller en anden microcontroller), og din ESP32 blot lytter på bussen:
1. ESP32'eren konfigureres med en canbus-komponent, som lytter efter det specifikke CAN ID, som bærer informationen om GPIO-pinden.
2. I ESPHome opretter du en virtuel template binary_sensor.
3. Når ESP32 opsnapper det rigtige CAN ID på bussen, udtrækker den status (f.eks. bit 0 i et byte-array) via en lille lambda-kode og opdaterer sensoren.
4. Sensoren dukker herefter automatisk op i Home Assistant via ESPHomes indbyggede API.

Eksempel: ESPHome-kode (Modtagelse af status via CAN)

Hvis en enhed sender status på et CAN-id (f.eks. 0x500), kan koden på din ESP32 se således ud for at sende det videre til Home Assistant:

```yaml
canbus:
  - platform: esp32_can
    tx_pin: GPIO22
    rx_pin: GPIO23
    bit_rate: 125kbps
    on_frame:
      - can_id: 0x500
        then:
          - lambda: |-
              // Hvis første byte i CAN-beskeden er 1, sættes sensoren til ON (true)
              if (x.size() > 0) {
                id(can_gpio_sensor).publish_state(x[0] == 1);
              }

binary_sensor:
  - platform: template
    name: "CAN-bus GPIO Status"
    id: can_gpio_sensor
```

Tip: Hvis du vil gøre det endnu mere avanceret og standardiseret, findes der også færdige community-komponenter som f.eks. esphome-canopen, som automatisk kan mappe enheder og GPIO-switche på en CAN-bus direkte over i Home Assistant-entiteter.
Hvis du vil have hjælp til at skrive koden, må du meget gerne fortælle:

* Skal ESP32'eren sende sin egen GPIO-status, eller skal den lytte efter en anden enhed?
* Kender du CAN ID'et og dataformatet (payload), som statussen sendes på?
* Hvilken CAN-transceiver / controller (f.eks. indbygget ESP32-CAN eller en ekstern MCP2515) bruger du?

## jeg bruger den indbyggede can controller

Det gør projektet en hel del simplere! Den indbyggede CAN-controller i ESP32 (ofte kaldet TWAI – Two-Wire Automotive Interface) er rigtig kraftfuld og understøttes direkte i ESPHome uden behov for eksterne chipsets som MCP2515. Du skal blot tilslutte en fysisk CAN-transceiver (f.eks. SN65HVD230) mellem ESP32'eren og dit CAN-netværk.
Her er, hvordan du griber det an i ESPHome, afhængigt af om du skal sende eller modtage statussen:

1. Hvis ESP32'eren skal SENDE sin egen GPIO-status ud på CAN-bussen

Hvis du har en kontakt eller en sensor siddende på f.eks. GPIO14 på din ESP32, og du vil have den til at sende status (0x00 eller 0x01) på CAN-id 0x100, ser din ESPHome-konfiguration således ud:

```yaml
canbus:
  - platform: esp32_can
    tx_pin: GPIO22  # CTX på din transceiver
    rx_pin: GPIO23  # CRX på din transceiver
    bit_rate: 250kbps  # Tilpas til dit netværks hastighed
    id: my_can_bus

binary_sensor:
  - platform: gpio
    pin:
      number: GPIO14
      mode: INPUT_PULLUP
    name: "Lokal GPIO Kontakt"
    on_press:
      then:
        - canbus.send:
            canbus_id: my_can_bus
            can_id: 0x100
            data: [0x01]  # Sender 1 (Tændt)
    on_release:
      then:
        - canbus.send:
            canbus_id: my_can_bus
            can_id: 0x100
            data: [0x00]  # Sender 0 (Slukket)
```

I Home Assistant dukker sensoren "Lokal GPIO Kontakt" op automatisk via ESPHome-integrationen.

2. Hvis ESP32'eren skal LYTTE efter status fra en anden enhed på CAN-bussen

Hvis en anden enhed på bussen spytter statusser ud (f.eks. på CAN-id 0x200), og du vil fange det og vise det i Home Assistant, bruger du en virtuel sensor (en template binary_sensor).
Her lytter den indbyggede controller efter ID'et og læser den første byte i datapakken:

```yaml
canbus:
  - platform: esp32_can
    tx_pin: GPIO22
    rx_pin: GPIO23
    bit_rate: 250kbps
    id: my_can_bus
    on_frame:
      - can_id: 0x200
        then:
          - lambda: |-
              // Hvis modtaget data er større end 0, tjekker vi den første byte (index 0)
              if (x.size() > 0) {
                if (x[0] == 0x01) {
                  id(can_extern_sensor).publish_state(true);  # ON i Home Assistant
                } else {
                  id(can_extern_sensor).publish_state(false); # OFF i Home Assistant
                }
              }

binary_sensor:
  - platform: template
    name: "CAN-bus Ekstern GPIO Status"
    id: can_extern_sensor
```

Tre vigtige ting at huske med den indbyggede controller:

1. Transceiver er nødvendig: ESP32'erens indbyggede controller kan ikke kobles direkte til CAN-bussens ledninger (CAN_H / CAN_L). Du skal have et lille breakout-print (f.eks. med en SN65HVD230 eller VP230 chip) imellem.
2. Husk 120 Ohms modstand: Dit CAN-bus netværk skal være termineret med en 120 ohms modstand i hver ende af de to yderste enheder for at undgå datarefleksioner. Viele SN65HVD230-moduler har denne modstand indbygget (ofte en lille SMD-modstand mærket "121").
3. GPIO-valg: Du kan frit vælge, hvilke GPIO-pinde du vil bruge til tx_pin og rx_pin på ESP32, så længe de understøtter almindelig digital I/O.

For at hjælpe dig helt i mål med koden, kan du fortælle mig:
* Skal ESP32'eren sende eller modtage data?
* Hvilken hastighed (bit rate) kører dit CAN-bus netværk med? (f.eks. 125kbps, 250kbps eller 500kbps)
* Hvis den skal modtage data, kender du så det specifikke CAN ID og hvordan dataen ser ud i pakken?
