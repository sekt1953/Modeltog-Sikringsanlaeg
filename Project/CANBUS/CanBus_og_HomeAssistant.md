# CanBus og HomeAssistant

## jeg har 2 esp32, 

* nr1 er forbundet til Home Assistant via ethernet og til canbus via SN65HVD230, 
* nr2 er forbundet til Canbus via SN65HVD230 og er til slutte et antal binary sensor. 
* Hvordan opfatter Home Assistant sensorene fra nr.2 esp32 ?

* Home Assistant opfatter ikke sensorerne fra ESP32 nr. 2 direkte som selvstændige enheder, da nr. 2 ikke har nogen direkte netværksforbindelse (hverken Wi-Fi eller Ethernet) til dit lokale netværk.
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
