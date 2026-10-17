# CanBus og HomeAssistant

## jeg har 2 esp32, 

* nr1 er forbundet til Home Assistant via ethernet og til canbus via SN65HVD230, 
* nr2 er forbundet til Canbus via SN65HVD230 og er til slutte et antal binary sensor. 
* Hvordan opfatter Home Assistant sensorene fra nr.2 esp32 ?

Home Assistant opfatter ikke sensorerne fra ESP32 nr. 2 direkte som selvstændige enheder, da nr. 2 ikke har nogen direkte netværksforbindelse (hverken Wi-Fi eller Ethernet) til dit lokale netværk.

* I stedet vil Home Assistant se sensorerne som entiteter, der tilhører ESP32 nr. 1. Det er ESP32 nr. 1, der fungerer som "bro" (gateway) mellem CAN-bussen og dit netværk.
* For at Home Assistant kan forstå, hvad der sker på ESP32 nr. 2, skal du konfigurere dem til at tale sammen via CAN-bussen i ESPHome. 

### Her er præcis, hvordan det fungerer i praksis:

Sådan sættes det op (Logikken)

1. ESP32 nr. 2 (Sensorenheden):
Hver gang en af dens fysiske sensorer ændrer status (f.eks. trykkes ned eller slippes), sender den en lille datapakke ud på CAN-bussen med et unikt CAN ID.
2. ESP32 nr. 1 (Gatewayen):
Den lytter uafbrudt på CAN-bussen. Når den opfanger en pakke fra nr. 2, opdaterer den en virtuel sensor (template binary_sensor) i sin egen kode.
3. Home Assistant:
Fordi ESP32 nr. 1 er forbundet via Ethernet, sender den øjeblikkeligt den nye status videre til Home Assistant via ESPHome-integrationen.

### Kodeeksempel til ESPHome

Her er et konkret eksempel på, hvordan koden skal se ud for begge enheder, hvis du f.eks. har 2 sensorer på ESP32 nr. 2. Vi tildeler dem CAN ID 0x10 og 0x20.

KODEN TIL ESP32 NR. 2 (Kun CAN + Fysiske sensorer)

Denne enhed har ikke api: eller ethernet: slået til, da den kun snakker CAN-bus.

```yaml
esphome:
  name: esp32-sensor-node

# Den indbyggede CAN-controller opsættes
canbus:
  - platform: esp32_can
    tx_pin: GPIO22  # Forbundet til CTX på din SN65HVD230
    rx_pin: GPIO23  # Forbundet til CRX på din SN65HVD230
    bit_rate: 250kbps
    id: can_bus

binary_sensor:
  # SENSOR 1
  - platform: gpio
    pin:
      number: GPIO14
      mode: INPUT_PULLUP
    on_press:
      - canbus.send: { canbus_id: can_bus, can_id: 0x10, data: [0x01] } # Send 1 ved tryk
    on_release:
      - canbus.send: { canbus_id: can_bus, can_id: 0x10, data: [0x00] } # Send 0 ved slip

  # SENSOR 2
  - platform: gpio
    pin:
      number: GPIO15
      mode: INPUT_PULLUP
    on_press:
      - canbus.send: { canbus_id: can_bus, can_id: 0x20, data: [0x01] }
    on_release:
      - canbus.send: { canbus_id: can_bus, can_id: 0x20, data: [0x00] }
```

KODEN TIL ESP32 NR. 1 (Gateway: Ethernet + CAN + Virtuelle sensorer)

Denne enhed modtager data fra bussen og opretter de sensorer, som Home Assistant rent faktisk kan se.

```yaml
esphome:
  name: esp32-gateway

ethernet:
  type: LAN8720  # Ret til det ethernet-board du bruger (f.eks. WT32-ETH01 eller Olimex)
  # ... dine ethernet-indstillinger ...

api: # Gør den synlig i Home Assistant

canbus:
  - platform: esp32_can
    tx_pin: GPIO22
    rx_pin: GPIO23
    bit_rate: 250kbps
    id: can_bus
    # Her lytter vi efter pakker fra nr. 2
    on_frame:
      - can_id: 0x10
        then:
          - lambda: |-
              if (x.size() > 0) id(sensor_node_1).publish_state(x[0] == 0x01);
      - can_id: 0x20
        then:
          - lambda: |-
              if (x.size() > 0) id(sensor_node_2).publish_state(x[0] == 0x01);

binary_sensor:
  # Disse dukker op i Home Assistant under "esp32-gateway"
  - platform: template
    name: "CAN Sensor 1 (fra node 2)"
    id: sensor_node_1

  - platform: template
    name: "CAN Sensor 2 (fra node 2)"
    id: sensor_node_2
```

### Det visuelle resultat i Home Assistant

* Når du går ind under Indstillinger -> Enheder og integrationer -> ESPHome i Home Assistant, vil du kun se én enhed: esp32-gateway.
Når du klikker på den, vil den indeholde entiteterne binary_sensor.can_sensor_1_fra_node_2 og binary_sensor.can_sensor_2_fra_node_2.  De vil opdatere sig fuldstændig i realtid, nøjagtigt som hvis de sad direkte på Ethernet-enheden.
* Hvis du vil have koden til at passe 100% til dit setup, må du gerne oplyse:
  * Hvilket Ethernet-board/chip bruger du på ESP32 nr. 1? (f.eks. WT32-ETH01, Olimex, PoE osv.)
  * Hvor mange binary sensorer har du i alt på ESP32 nr. 2?

##  Jeg har en 3. esp32 med en del output, forbundet kun til CANBUsen, har du løsning på det ?

* Det har jeg helt bestemt! Princippet for din ESP32 nr. 3 (Output-enheden) er præcis det samme som med sensorerne, bare i omvendt rækkefølge.
* Da ESP32 nr. 3 kun sidder på CAN-bussen, kan Home Assistant ikke kontrollere den direkte. I stedet skal ESP32 nr. 1 (Gatewayen) modtage kommandoer fra Home Assistant (f.eks. "Tænd lyset") og sende en besked ud på CAN-bussen. ESP32 nr. 3 lytter efter denne besked og tænder eller slukker for sit fysiske output.

I Home Assistant vil disse outputs (switches/relæer) optræde som entiteter under din ESP32 nr. 1 (Gateway).
Her er den komplette løsning og kode til, hvordan du sætter det op i ESPHome:

### 1. KODEN TIL ESP32 NR. 1 (Gateway: Ethernet + CAN)

Her opretter vi virtuelle kontakter (template switch). Når du trykker på kontakten i Home Assistant, sender gatewayen en CAN-besked.
* Vi bruger CAN ID 0x30 til Output 1 og 0x40 til Output 2.
* Datapakken [0x01] betyder TÆND, og [0x00] betyder SLUK.

```yaml
esphome:
  name: esp32-gateway

ethernet:
  # ... dine ethernet-indstillinger som før ...

api: # Gør enheden synlig i Home Assistant

canbus:
  - platform: esp32_can
    tx_pin: GPIO22
    rx_pin: GPIO23
    bit_rate: 250kbps
    id: can_bus

switch:
  # VIRTUEL KONTAKT 1
  - platform: template
    name: "CAN Output 1 (på node 3)"
    id: can_out_1
    turn_on_action:
      - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: [0x01] }
    turn_off_action:
      - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: [0x00] }

  # VIRTUEL KONTAKT 2
  - platform: template
    name: "CAN Output 2 (på node 3)"
    id: can_out_2
    turn_on_action:
      - canbus.send: { canbus_id: can_bus, can_id: 0x40, data: [0x01] }
    turn_off_action:
      - canbus.send: { canbus_id: can_bus, can_id: 0x40, data: [0x00] }
```

### 2. KODEN TIL ESP32 NR. 3 (Kun CAN + Fysiske Outputs)

Denne enhed har ingen netværksforbindelse. Den lytter på CAN-bussen efter ID 0x30 og 0x40. Når den modtager en pakke, tjekker den indholdet (data) og styrer de fysiske relæer/outputs på f.eks. GPIO12 og GPIO13.

```yaml
esphome:
  name: esp32-output-node

canbus:
  - platform: esp32_can
    tx_pin: GPIO22  # Forbundet til CTX på SN65HVD230
    rx_pin: GPIO23  # Forbundet til CRX på SN65HVD230
    bit_rate: 250kbps
    id: can_bus
    # Her lytter vi efter styresignaler fra gatewayen
    on_frame:
      # Lyt efter kommandoer til Output 1
      - can_id: 0x30
        then:
          - lambda: |-
              if (x.size() > 0) {
                if (x == 0x01) {
                  id(fysisk_relae_1).turn_on();
                } else {
                  id(fysisk_relae_1).turn_off();
                }
              }
      # Lyt efter kommandoer til Output 2
      - can_id: 0x40
        then:
          - lambda: |-
              if (x.size() > 0) {
                if (x == 0x01) {
                  id(fysisk_relae_2).turn_on();
                } else {
                  id(fysisk_relae_2).turn_off();
                }
              }

# De reelle fysiske udgange på ESP32 nr. 3
switch:
  - platform: gpio
    pin: GPIO12
    id: fysisk_relae_1
    # name er udeladt eller sat til internal, da HA ikke skal se den direkte
    internal: true 

  - platform: gpio
    pin: GPIO13
    id: fysisk_relae_2
    internal: true
```

### Avanceret tip: Status-feedback (Tovejs-kommunikation)

* I eksemplet ovenfor "gætter" Home Assistant på, at outputtet tænder på ESP32 nr. 3, når du trykker på knappen. Hvis du vil være 100% sikker på status (f.eks. hvis ESP32 nr. 3 genstarter), kan du lade ESP32 nr. 3 sende en bekræftelse tilbage på bussen, når relæet rent faktisk skifter tilstand.
* Hvis det bliver aktuelt, kan vi udvide koden med on_turn_on og on_turn_off handlinger på ESP32 nr. 3, som gatewayen så lytter efter.

Hvis vi skal finpudse koden, kan du fortælle mig:

* Hvor mange outputs skal du styre på ESP32 nr. 3?
* Er det vigtigt for dig med status-feedback, så Home Assistant ved med sikkerhed, om relæet er tændt på den anden side af bussen?

