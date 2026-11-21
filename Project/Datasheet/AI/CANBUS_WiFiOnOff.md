# CanBus Wifi On/Off tilføjet til: ***base_s3_can.yaml***

## Det vil være rart hvis base_s3_can.yaml indeholder styringen af wifi On/Off styret af canbus signal for alle noder på denne canbus

Det er en rigtig smart tilføjelse. Ved at styre WiFi via et globalt CAN-bus signal kan du holde alle 40+ noder helt uden radiostøj under normal drift, og tænde WiFi på alle noder på én station samtidig, når du skal opdatere firmware (OTA) eller fejlfinde via web/log.

Her er den opdaterede packages/base_s3_can.yaml, som lytter på et fælles broadcast CAN ID (f.eks. 0x001):

* Send [0x01] på ID 0x001 fra LAN-gatewayen → Tænder WiFi på alle stationens noder.
* Send [0x00] på ID 0x001 fra LAN-gatewayen → Slukker WiFi igen.

### 📦 Opdateret grundpakke: packages/base_s3_can.yaml

```yaml
# packages/base_s3_can.yaml
substitutions:
  can_id_wifi_broadcast: "0x001" # Fælles broadcast ID for alle noder på stationen

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf

# WiFi og OTA er klar, men slukkes som standard 2 sekunder efter opstart
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  on_connect:
    - logger.log: "WiFi forbundet og klar til OTA"

ota:
  - platform: esphome

logger:

esphome:
  on_boot:
    priority: -100
    then:
      - delay: 2s
      - wifi.disable:

# Lokal CAN-bus og I2C
canbus:
  - id: lokal_can
    platform: esp32_can
    rx_pin: GPIO4
    tx_pin: GPIO5
    bit_rate: 125kbps
    on_frame:
      # Global WiFi On/Off styring for alle noder på stationen
      - can_id: ${can_id_wifi_broadcast}
        then:
          - lambda: |-
              if (x.size() > 0) {
                if (x[0] == 0x01) {
                  id(wifi_enable_script).execute();
                } else if (x[0] == 0x00) {
                  id(wifi_disable_script).execute();
                }
              }

script:
  - id: wifi_enable_script
    then:
      - wifi.enable:
  - id: wifi_disable_script
    then:
      - wifi.disable:

i2c:
  sda: GPIO1
  scl: GPIO2
```

Vær forsigtig med at bruge denne kode.

### 🌐 Tilføjelse til station01_lan.yaml (Knap i Home Assistant)

For at du nemt kan tænde og slukke for WiFi på hele stationen inde fra Home Assistant, tilføjer du blot denne ene switch i din station01_lan.yaml:

```yaml
switch:
  - platform: template
    name: "Station WiFi (OTA Service)"
    icon: "mdi:wifi-cog"
    optimistic: true
    turn_on_action:
      - canbus.send: { canbus_id: lokal_can, can_id: 0x001, data: [0x01] }
    turn_off_action:
      - canbus.send: { canbus_id: lokal_can, can_id: 0x001, data: [0x00] }
```

Vær forsigtig med at bruge denne kode.

### 💡 To vigtige detaljer i den nye kode

1. !secret wifi_ssid: Ved at bruge !secret i grundpakken slipper du for at skrive dit WiFi-kodeord i koden. Du lægger blot wifi_ssid og wifi_password i din standard secrets.yaml fil i ESPHome.
2. Sikkerhed ved opstart (on_boot): Koden sørger for, at alle noder automatisk slukker for WiFi 2 sekunder efter, at strømmen til anlægget tændes. Hvis du glemmer at slukke service-switchen i Home Assistant efter en opdatering, slukker noderne altså selv for WiFi næste gang I tænder banen.

Skal jeg generere en opdateret PDF-fil (v2) med denne nye WiFi-styring i base_s3_can.yaml og station01_lan.yaml indarbejdet, eller vil du have tilføjet en automatisk timeout (f.eks. at WiFi selv slukker igen efter 15 minutter)?

AI-svar kan indeholde fejl. Få flere oplysninger
