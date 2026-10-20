# 4_CanBus_jeg vil gerne hav en hurtig introduktion til hvordan du udruller den fælles pakkefil i ESPHome

## jeg vil gerne hav en hurtig introduktion til hvordan du udruller den fælles pakkefil i ESPHome?

Her er en hurtig guide til, hvordan du placerer og udruller din fælles pakkefil (can_base.yaml), så dine tre ESP32-enheder kan finde den.

### Step 1: Placer den fælles fil det rigtige sted

ESPHome leder efter pakker lokalt i det bibliotek, hvor dine enhedskonfigurationer (.yaml-filer) ligger.

1. Åbn din fil-editor i Home Assistant (f.eks. File Editor- eller VS Code-add-on).
2. Gå til mappen config/esphome/.
3. Opret en ny fil og navngiv den nøjagtigt can_base.yaml.
4. Indsæt den fælles CAN-bus kode i filen og gem:

```yaml
# can_base.yaml
substitutions:
  can_tx_pin: "GPIO22" # Standard pinde, hvis andet ikke angives
  can_rx_pin: "GPIO23"
  can_speed: "250kbps"

canbus:
  - platform: esp32_can
    tx_pin: ${can_tx_pin}
    rx_pin: ${can_rx_pin}
    bit_rate: ${can_speed}
    id: can_bus
```

### Step 2: Forbind dine enheder til pakken

I dine tre andre enhedsfiler (f.eks. esp32_gateway.yaml, esp32_node2.yaml, esp32_node3.yaml) refererer du nu til pakken øverst i koden vha. !include:

```yaml
packages:
  can_network: !include can_base.yaml
```

Når ESPHome skal kompilere koden til en af dine ESP32'ere, tager den automatisk indholdet fra can_base.yaml og fletter det usynligt sammen med enhedens egen kode.

### Step 3: Kompiler og udrul (Flash)

Nu gør du fuldstændig, som du plejer i ESPHome-brugerfladen:

1. Åbn ESPHome-dashboardet i Home Assistant.
2. Find din enhed (f.eks. din Waveshare Gateway) og klik på de tre prikker og vælg Install (eller klik Install på det samlede overblik).
3. ESPHome vil nu hente den fælles pakke i baggrunden, tjekke for pin-konflikter, kompilere firmwaren og udrulle den til din ESP32 (enten via netværket/OTA eller USB).

💡 Det smarte i fremtiden (Når alt kører)

Hvis du en dag finder ud af, at 250kbps er for langsomt til din CAN-bus, eller du har brug for at ændre hastigheden til 500kbps, skal du kun ændre det ét sted:

1. Åbn can_base.yaml.
2. Ret can_speed: "250kbps" til can_speed: "500kbps".
3. Gem filen, og klik Install på dine 3 ESP32'ere i dashboardet. De henter alle tre den nye hastighed automatisk.

Er du klar til at oversætte og udrulle den første enhed, eller vil du have mig til at kigge på, hvordan du overvåger CAN-bus-trafikken i loggen for at sikre dig, at de tre ESP32'ere rent faktisk snakker sammen?

## jeg vil gerne have at node 2 har 8 input, hvad ændre dette

Når **ESP32 nr. 2 (Sensorenheden)** skal udvides til at have 8 inputs, ændrer det sig til at fungere på fuldstændig samme smarte måde, som vi opsatte for feedback-signalerne på enhed nr. 3.

I stedet for at sende 8 forskellige, tunge beskeder ud på CAN-bussen, pakker vi status på de 8 inputs sammen til én enkelt byte (f.eks. 00101001) og sender den afsted på **ét samlet CAN ID (0x21)**.

Her er præcis, hvad det ændrer i dine filer:

### 1. Det ændrer koden på ESP32 nr. 2 (esp32_node2.yaml)

Vi indfører den ensartede id_in_-notation for alle 8 inputs, og tilføjer et script, der samler og sender data i en pakke på CAN ID 0x21 hver gang en sensor ændrer status.

```yaml
# esp32_node2.yaml
esphome:
  name: esp32-sensor-node
  comment: "8-kanals Fysisk Sensor Node på CAN-bus"

packages:
  can_network: !include can_base.yaml

# Script der indsamler status fra de 8 sensorer og sender dem som 1 byte over CAN
script:
  - id: send_sensors
    mode: restart
    then:
      - lambda: |-
          uint8_t sensor_byte = 0;
          if (id(id_in_0).state) sensor_byte |= (1 << 0);
          if (id(id_in_1).state) sensor_byte |= (1 << 1);
          if (id(id_in_2).state) sensor_byte |= (1 << 2);
          if (id(id_in_3).state) sensor_byte |= (1 << 3);
          if (id(id_in_4).state) sensor_byte |= (1 << 4);
          if (id(id_in_5).state) sensor_byte |= (1 << 5);
          if (id(id_in_6).state) sensor_byte |= (1 << 6);
          if (id(id_in_7).state) sensor_byte |= (1 << 7);
          
          std::vector<uint8_t> data = { sensor_byte };
          id(can_bus).send(0x21, data);

# --- FYSISKE BINÆRE SENSORER (INPUTS) ---
# Husk at rette GPIOXX til dine faktiske fysiske pinde
binary_sensor:
  - platform: gpio
    pin: { number: GPIO12, mode: INPUT_PULLUP }
    id: id_in_0
    on_state: [ - script.execute: send_sensors ]
  - platform: gpio
    pin: { number: GPIO13, mode: INPUT_PULLUP }
    id: id_in_1
    on_state: [ - script.execute: send_sensors ]
  - platform: gpio
    pin: { number: GPIO14, mode: INPUT_PULLUP }
    id: id_in_2
    on_state: [ - script.execute: send_sensors ]
  - platform: gpio
    pin: { number: GPIO15, mode: INPUT_PULLUP }
    id: id_in_3
    on_state: [ - script.execute: send_sensors ]
  - platform: gpio
    pin: { number: GPIO16, mode: INPUT_PULLUP }
    id: id_in_4
    on_state: [ - script.execute: send_sensors ]
  - platform: gpio
    pin: { number: GPIO17, mode: INPUT_PULLUP }
    id: id_in_5
    on_state: [ - script.execute: send_sensors ]
  - platform: gpio
    pin: { number: GPIO18, mode: INPUT_PULLUP }
    id: id_in_6
    on_state: [ - script.execute: send_sensors ]
  - platform: gpio
    pin: { number: GPIO19, mode: INPUT_PULLUP }
    id: id_in_7
    on_state: [ - script.execute: send_sensors ]
```

### 2. Det ændrer koden på din Waveshare Gateway (esp32_gateway.yaml)

I din Gateway skal du fjerne de gamle modtagere (can_id: 0x10 og 0x20) og erstatte dem med en ny modtager for can_id: 0x21. Samtidig opretter vi 8 virtuelle sensorer, som dukker op i Home Assistant.

Du skal rette afsnittet for canbus: og binary_sensor: i din Gateway-fil til dette:

```yaml
# Sæt dette ind i din eksisterende esp32_gateway.yaml under canbus: -> on_frame:
    on_frame:
      # --- MODTAGELSE FRA ESP32 NR. 2 (Nu som 1 samlet byte på ID 0x21) ---
      - can_id: 0x21
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(node2_sensor_0).publish_state((x >> 0) & 1);
                id(node2_sensor_1).publish_state((x >> 1) & 1);
                id(node2_sensor_2).publish_state((x >> 2) & 1);
                id(node2_sensor_3).publish_state((x >> 3) & 1);
                id(node2_sensor_4).publish_state((x >> 4) & 1);
                id(node2_sensor_5).publish_state((x >> 5) & 1);
                id(node2_sensor_6).publish_state((x >> 6) & 1);
                id(node2_sensor_7).publish_state((x >> 7) & 1);
              }

      # --- (Behold din eksisterende on_frame kode for can_id: 0x31 her) ---


# Sæt dette ind under binary_sensor: i din esp32_gateway.yaml
binary_sensor:
  - platform: template
    name: "Node 2 - Sensor 1"
    id: node2_sensor_0
  - platform: template
    name: "Node 2 - Sensor 2"
    id: node2_sensor_1
  - platform: template
    name: "Node 2 - Sensor 3"
    id: node2_sensor_2
  - platform: template
    name: "Node 2 - Sensor 4"
    id: node2_sensor_3
  - platform: template
    name: "Node 2 - Sensor 5"
    id: node2_sensor_4
  - platform: template
    name: "Node 2 - Sensor 6"
    id: node2_sensor_5
  - platform: template
    name: "Node 2 - Sensor 7"
    id: node2_sensor_6
  - platform: template
    name: "Node 2 - Sensor 8"
    id: node2_sensor_7
```

### Resultatet

Med denne ændring holdes datatrafikken på din CAN-bus på et absolut minimum. Hver gang en kontakt trykkes ned på ESP32 nr. 2, skydes der lynhurtigt en byte afsted på ID 0x21, og alle 8 sensorer opdateres øjeblikkeligt i Home Assistant.

Vil du have mig til at samle den fuldstændige, endelige esp32_gateway.yaml-fil for din Waveshare, så du er sikker på, at de nye 8 sensorer fra node 2 og de 8 relæer fra node 3 er flettet helt fejlfrit sammen?

