# Hvordan navn giver vi noder på canbusen

## jeg kunne tænke mig at gatewaye noden er "node type 1" og noder bygget på node 2 kaldes "node type2" og bruger ID fra 20 - 39 og noder byhggende på node 3 kaldes " node type 3" og bruger ID fra 50-69

Det er en yderst professionel og struktureret måde at organisere dit CAN-bus netværk på! Ved at tildele faste "ID-ranger" til de forskellige enhedstyper (f.eks. **Node Type 2 bruger ID 20-39 og Node Type 3 bruger ID 50-69**), undgår du helt adressekonflikter, og det bliver ekstremt let at tilføje nye noder i fremtiden.

Når du arbejder med CAN-bus hex-værdier (f.eks. 0x14 for 20), er det mest overskueligt at lade dine ID-ranger køre i decimaler, da det er lettere at tælle i hovedet.

Her er den fuldt opdaterede struktur, opdelt i dine 3 enhedstyper:

### Step 1: Den opdaterede fælles basispakke (can_base.yaml)

Vi beholder den simple grundpakke. De specifikke regler lægger vi ud på enhedstyperne.

```yaml
# can_base.yaml
substitutions:
  can_tx_pin: "GPIO22"
  can_rx_pin: "GPIO23"
  can_speed: "250kbps"

canbus:
  - platform: esp32_can
    tx_pin: ${can_tx_pin}
    rx_pin: ${can_rx_pin}
    bit_rate: ${can_speed}
    id: can_bus
```

### Step 2: Fælles pakke til Node Type 2 (can_type2_base.yaml)

Denne pakke inkluderes af alle dine sensornoder. Her opsætter vi de 8 inputs med den korrekte id_in_-notation, som sender alt afsted i én byte til det tildelte ID (f.eks. 20, 21, 22...).

```yaml
# can_type2_base.yaml
substitutions:
  node_id: "20" # Standard start-ID (Decimal) for Type 2 (Maks 39)

packages:
  can_network: !include can_base.yaml

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
          id(can_bus).send(${node_id}, data);

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

### Step 3: Fælles pakke til Node Type 3 (can_type3_base.yaml)

Denne pakke bruges til alle output-noder. Den lytter efter styring på sit tildelte ID (f.eks. 50), og sender sin sensor/feedback status tilbage på ID + 1 (f.eks. 51). Det holder tingene pænt parret!

```yaml
# can_type3_base.yaml
substitutions:
  node_cmd_id: "50"       # Styring modtages her (Decimal, Maks 68)
  node_feedback_id: "51"  # Feedback sendes herfra (node_cmd_id + 1, Maks 69)

packages:
  can_network: !include can_base.yaml

canbus:
  - id: can_bus
    on_frame:
      - can_id: ${node_cmd_id}
        then:
          - lambda: |-
              if (x.size() >= 2) {
                uint8_t index = x;
                bool state = x;
                if (index == 0) { if(state) id(out_0).turn_on(); else id(out_0).turn_off(); }
                else if (index == 1) { if(state) id(out_1).turn_on(); else id(out_1).turn_off(); }
                else if (index == 2) { if(state) id(out_2).turn_on(); else id(out_2).turn_off(); }
                else if (index == 3) { if(state) id(out_3).turn_on(); else id(out_3).turn_off(); }
                else if (index == 4) { if(state) id(out_4).turn_on(); else id(out_4).turn_off(); }
                else if (index == 5) { if(state) id(out_5).turn_on(); else id(out_5).turn_off(); }
                else if (index == 6) { if(state) id(out_6).turn_on(); else id(out_6).turn_off(); }
                else if (index == 7) { if(state) id(out_7).turn_on(); else id(out_7).turn_off(); }
              }

script:
  - id: send_feedback
    mode: restart
    then:
      - lambda: |-
          uint8_t feedback_byte = 0;
          if (id(id_in_0).state) feedback_byte |= (1 << 0);
          if (id(id_in_1).state) feedback_byte |= (1 << 1);
          if (id(id_in_2).state) feedback_byte |= (1 << 2);
          if (id(id_in_3).state) feedback_byte |= (1 << 3);
          if (id(id_in_4).state) feedback_byte |= (1 << 4);
          if (id(id_in_5).state) feedback_byte |= (1 << 5);
          if (id(id_in_6).state) feedback_byte |= (1 << 6);
          if (id(id_in_7).state) feedback_byte |= (1 << 7);
          
          std::vector<uint8_t> data = { feedback_byte };
          id(can_bus).send(${node_feedback_id}, data);

switch:
  - platform: gpio
    pin: GPIO1
    id: out_0
    internal: true
  # ... (gentag out_1 til out_7 som før) ...

binary_sensor:
  - platform: gpio
    pin: { number: GPIO12, mode: INPUT_PULLUP }
    id: id_in_0
    on_state: [ - script.execute: send_feedback ]
  # ... (gentag id_in_1 til id_in_7 som før) ...
```

### Step 4: Opret dine individuelle noder (Eksempler)

En "Node Type 2" enhed (f.eks. esp32_sensor_20.yaml):

```yaml
esphome:
  name: type2-sensor-node-20

packages:
  type2_base: !include can_type2_base.yaml

substitutions:
  node_id: "20" # Bruger første ID i din 20-39 range
Vær forsigtig med at bruge denne kode.
En "Node Type 3" enhed (f.eks. esp32_output_50.yaml):
yaml
esphome:
  name: type3-output-node-50

packages:
  type3_base: !include can_type3_base.yaml

substitutions:
  node_cmd_id: "50"      # Styring på ID 50
  node_feedback_id: "51" # Feedback sendes automatisk tilbage på ID 51
```

### Step 5: Node Type 1 (Din Master Waveshare Gateway)

Her er koden til din master-gateway. Læg mærke til, hvordan den lytter på decimal-adresser i on_frame: i stedet for hex, så det matcher dine substitutions perfekt:

```yaml
esphome:
  name: esp32-type1-gateway
  comment: "Waveshare ESP32-S3-ETH Node Type 1 Gateway"

esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  framework:
    type: esp-idf

packages:
  can_network: !include can_base.yaml

substitutions:
  can_tx_pin: "GPIO4"
  can_rx_pin: "GPIO5"

ethernet:
  type: W5500
  clk_pin: GPIO13
  mosi_pin: GPIO11
  miso_pin: GPIO12
  cs_pin: GPIO14
  interrupt_pin: GPIO10
  reset_pin: GPIO9

api:
ota:
  - platform: esphome

canbus:
  - id: can_bus
    on_frame:
      # ==========================================
      # SENSOR NODE MODTAGERE (Node Type 2: ID 20-39)
      # ==========================================
      - can_id: 20  # Lytter efter Node Type 2 på ID 20
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(t2_n20_s0).publish_state((x >> 0) & 1); id(t2_n20_s1).publish_state((x >> 1) & 1);
                id(t2_n20_s2).publish_state((x >> 2) & 1); id(t2_n20_s3).publish_state((x >> 3) & 1);
                id(t2_n20_s4).publish_state((x >> 4) & 1); id(t2_n20_s5).publish_state((x >> 5) & 1);
                id(t2_n20_s6).publish_state((x >> 6) & 1); id(t2_n20_s7).publish_state((x >> 7) & 1);
              }

      # ==========================================
      # OUTPUT FEEDBACK MODTAGERE (Node Type 3: ID 50-69)
      # ==========================================
      - can_id: 51  # Lytter efter feedback fra output-node på ID 51 (hørende til kommando ID 50)
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(t3_n50_sw0).publish_state((x >> 0) & 1); id(t3_n50_sw1).publish_state((x >> 1) & 1);
                id(t3_n50_sw2).publish_state((x >> 2) & 1); id(t3_n50_sw3).publish_state((x >> 3) & 1);
                id(t3_n50_sw4).publish_state((x >> 4) & 1); id(t3_n50_sw5).publish_state((x >> 5) & 1);
                id(t3_n50_sw6).publish_state((x >> 6) & 1); id(t3_n50_sw7).publish_state((x >> 7) & 1);
              }

# --- ENTITETER DER SENDS TIL HOME ASSISTANT ---
binary_sensor:
  # Virtuelle sensorer for den første Type 2 node (ID 20)
  - platform: template
    name: "Type2 Node 20 - Sensor 1"
    id: t2_n20_s0
  - platform: template
    name: "Type2 Node 20 - Sensor 2"
    id: t2_n20_s1
  - platform: template
    name: "Type2 Node 20 - Sensor 3"
    id: t2_n20_s2
  - platform: template
    name: "Type2 Node 20 - Sensor 4"
    id: t2_n20_s3
  - platform: template
    name: "Type2 Node 20 - Sensor 5"
    id: t2_n20_s4
  - platform: template
    name: "Type2 Node 20 - Sensor 6"
    id: t2_n20_s5
  - platform: template
    name: "Type2 Node 20 - Sensor 7"
    id: t2_n20_s6
  - platform: template
    name: "Type2 Node 20 - Sensor 8"
    id: t2_n20_s7

switch:
  # Virtuelle kontakter for den første Type 3 node (Styres via ID 50)
  - platform: template
    name: "Type3 Node 50 - Relæ 1"
    id: t3_n50_sw0
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
  - platform: template
    name: "Type3 Node 50 - Relæ 2"
    id: t3_n50_sw1
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
  - platform: template
    name: "Type3 Node 50 - Relæ 3"
    id: t3_n50_sw2
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
  - platform: template
    name: "Type3 Node 50 - Relæ 4"
    id: t3_n50_sw3
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
  - platform: template
    name: "Type3 Node 50 - Relæ 5"
    id: t3_n50_sw4
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
  - platform: template
    name: "Type3 Node 50 - Relæ 6"
    id: t3_n50_sw5
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
  - platform: template
    name: "Type3 Node 50 - Relæ 7"
    id: t3_n50_sw6
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
  - platform: template
    name: "Type3 Node 50 - Relæ 8"
    id: t3_n50_sw7
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 50, data: }
```

Nu hvor adresseringsstrukturen er helt skarp, vil du så have mig til at forklare, hvordan flere Type 2 eller Type 3 noder tilføjes i Gatewayen, når du begynder at udvide over de kommende uger?


## jeg har intet imod at bruge hex værdier, kom bare med et navne forslag

Hvis du foretrækker at arbejde med hex-værdier, gør det faktisk koden endnu mere strømlinet, da det er CAN-bussens oprindelige sprog.

Et oplagt og meget struktureret navne- og adreseforslag, der passer til dine ranges (20-39 decimal \(\rightarrow \) 0x14–0x27 og 50-69 decimal \(\rightarrow \) 0x32–0x45), ser således ud:

🏷️ Navne- og adresseforslag (Hex)


|Node Type|Funktion|Range (Hex)|Navneforslag (YAML / Home Assistant)|
|:---|:---|:---|:---|
|Node Type 1|PoE Gateway|Central|type1_poe_gateway|
|Node Type 2|8x Input Modul|0x14 til 0x27|type2_input_0x14, type2_input_0x15...|
|Node Type 3|8x Output (Kommando)|	0x32 til 0x44|type3_output_0x32, type3_output_0x34...|
|Node Type 3|8x Feedback (Svar)|0x33 til 0x45|(Parret automatisk som Kommando-ID + 1)|

Ved at bruge dette mønster for Node Type 3, bruger den første output-node 0x32 til at modtage tænd/sluk-kommandoer, og svarer automatisk tilbage til gatewayen på 0x33. Den næste output-node vil så bruge 0x34 (kommando) og 0x35 (feedback).

Her er de opdaterede, rå type-pakker samt eksempler på individuelle noder, så du kan udrulle dit system direkte med hex-værdier:

### 1. Pakke til Node Type 2 (can_type2_base.yaml)

```yaml
# can_type2_base.yaml
substitutions:
  node_id: "0x14" # Første ID i din 20-39 range (Hex 0x14)

packages:
  can_network: !include can_base.yaml

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
          id(can_bus).send(${node_id}, data);

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

### 2. Pakke til Node Type 3 (can_type3_base.yaml)

```yaml
# can_type3_base.yaml
substitutions:
  node_cmd_id: "0x32"       # Kommando-modtager (Hex 0x32 = 50 decimal)
  node_feedback_id: "0x33"  # Feedback-sender (Hex 0x33 = 51 decimal)

packages:
  can_network: !include can_base.yaml

canbus:
  - id: can_bus
    on_frame:
      - can_id: ${node_cmd_id}
        then:
          - lambda: |-
              if (x.size() >= 2) {
                uint8_t index = x;
                bool state = x;
                if (index == 0) { if(state) id(out_0).turn_on(); else id(out_0).turn_off(); }
                else if (index == 1) { if(state) id(out_1).turn_on(); else id(out_1).turn_off(); }
                else if (index == 2) { if(state) id(out_2).turn_on(); else id(out_2).turn_off(); }
                else if (index == 3) { if(state) id(out_3).turn_on(); else id(out_3).turn_off(); }
                else if (index == 4) { if(state) id(out_4).turn_on(); else id(out_4).turn_off(); }
                else if (index == 5) { if(state) id(out_5).turn_on(); else id(out_5).turn_off(); }
                else if (index == 6) { if(state) id(out_6).turn_on(); else id(out_6).turn_off(); }
                else if (index == 7) { if(state) id(out_7).turn_on(); else id(out_7).turn_off(); }
              }

script:
  - id: send_feedback
    mode: restart
    then:
      - lambda: |-
          uint8_t feedback_byte = 0;
          if (id(id_in_0).state) feedback_byte |= (1 << 0);
          if (id(id_in_1).state) feedback_byte |= (1 << 1);
          if (id(id_in_2).state) feedback_byte |= (1 << 2);
          if (id(id_in_3).state) feedback_byte |= (1 << 3);
          if (id(id_in_4).state) feedback_byte |= (1 << 4);
          if (id(id_in_5).state) feedback_byte |= (1 << 5);
          if (id(id_in_6).state) feedback_byte |= (1 << 6);
          if (id(id_in_7).state) feedback_byte |= (1 << 7);
          
          std::vector<uint8_t> data = { feedback_byte };
          id(can_bus).send(${node_feedback_id}, data);

switch:
  - platform: gpio
    pin: GPIO1
    id: out_0
    internal: true
# ... (Behold out_1 til out_7 herfra) ...

binary_sensor:
  - platform: gpio
    pin: { number: GPIO12, mode: INPUT_PULLUP }
    id: id_in_0
    on_state: [ - script.execute: send_feedback ]
# ... (Behold id_in_1 til id_in_7 herfra) ...
```

### 3. Eksempel på en individuel Node Type 2 (type2_input_0x14.yaml)

```yaml
esphome:
  name: type2-input-0x14

packages:
  type2_base: !include can_type2_base.yaml

substitutions:
  node_id: "0x14"
```

### 4. Eksempel på en individuel Node Type 3 (type3_output_0x32.yaml)

```yaml
esphome:
  name: type3-output-0x32

packages:
  type3_base: !include can_type3_base.yaml

substitutions:
  node_cmd_id: "0x32"
  node_feedback_id: "0x33"
```

### 5. Den opdaterede centrale Gateway (esp32_gateway.yaml)

Her modtager gatewayen nu data via de reelle hex-adresser. Den parrer input-noder og relæ-kontakter i Home Assistant baseret på dit nye navneforslag:

```yaml
esphome:
  name: type1-poe-gateway
  comment: "Waveshare ESP32-S3-ETH Master Gateway"

esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  framework: { type: esp-idf }

packages:
  can_network: !include can_base.yaml

substitutions:
  can_tx_pin: "GPIO4"
  can_rx_pin: "GPIO5"

ethernet:
  type: W5500
  clk_pin: GPIO13
  mosi_pin: GPIO11
  miso_pin: GPIO12
  cs_pin: GPIO14
  interrupt_pin: GPIO10
  reset_pin: GPIO9

api:
ota:
  - platform: esphome

canbus:
  - id: can_bus
    on_frame:
      # --- MODTAG SENSORER FRA TYPE 2 (ID 0x14) ---
      - can_id: 0x14
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(t2_14_s0).publish_state((x >> 0) & 1); id(t2_14_s1).publish_state((x >> 1) & 1);
                id(t2_14_s2).publish_state((x >> 2) & 1); id(t2_14_s3).publish_state((x >> 3) & 1);
                id(t2_14_s4).publish_state((x >> 4) & 1); id(t2_14_s5).publish_state((x >> 5) & 1);
                id(t2_14_s6).publish_state((x >> 6) & 1); id(t2_14_s7).publish_state((x >> 7) & 1);
              }

      # --- MODTAG FEEDBACK FRA TYPE 3 (Svarer på ID 0x33) ---
      - can_id: 0x33
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(t3_32_r0).publish_state((x >> 0) & 1); id(t3_32_r1).publish_state((x >> 1) & 1);
                id(t3_32_r2).publish_state((x >> 2) & 1); id(t3_32_r3).publish_state((x >> 3) & 1);
                id(t3_32_r4).publish_state((x >> 4) & 1); id(t3_32_r5).publish_state((x >> 5) & 1);
                id(t3_32_r6).publish_state((x >> 6) & 1); id(t3_32_r7).publish_state((x >> 7) & 1);
              }

# --- HOME ASSISTANT VISNING ---
binary_sensor:
  - platform: template
    name: "Type 2 [0x14] - Sensor 1"
    id: t2_14_s0
  # ... (kopier s1 til s7 herfra på samme form) ...
  - platform: template
    name: "Type 2 [0x14] - Sensor 8"
    id: t2_14_s7

switch:
  - platform: template
    name: "Type 3 [0x32] - Relæ 1"
    id: t3_32_r0
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x32, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x32, data: }
  # ... (kopier r1 til r7 herfra på samme form) ...
  - platform: template
    name: "Type 3 [0x32] - Relæ 8"
    id: t3_32_r7
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x32, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x32, data: }
```

Dette fuldender hex-arkitekturen! Har du brug for flere eksempler på, hvordan Home Assistant Dashboardet automatisk grupperer disse navne, eller vil du i gang med at oprette filerne lokalt på din server nu?

