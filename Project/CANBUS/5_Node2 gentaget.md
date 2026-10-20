# flere kloner af node 2

## node 2, er en model for flere noder med samme opbygning, der skal nok bruges 4 eller 8 af dem, kan du telpasse dette

Når du skal skalere systemet til at have **4 eller 8 ens sensornoder** (som alle ligner Node 2), er det her, ESPHome Packages og bit-metoden for alvor viser deres værd.

For at du ikke skal skrive den samme kode igen og igen, og for at undgå pinlige tastefejl med bytes og bits, kan vi automatisere opsætningen. Vi ændrer strategien en smule:

1. **Én fælles pakke for alle sensornoder**: Hver node får tildelt sit eget unikke CAN ID (f.eks. 0x21, 0x22, 0x23 osv.) via en variabel (substitution).
2. **Gatewayen tilpasses til at modtage fra alle noder**: Gatewayen lytter blot efter rækken af CAN ID'er.

Her er det fuldstændige, skalerbare setup tilpasset til op til 8 sensornoder (Node 2, Node 4, Node 5 osv.):

### Step 1: Lav en ny fælles pakke til sensornoderne (can_sensor_node.yaml)

Opret denne nye fil i din config/esphome/-mappe. Den indeholder hele logikken for de 8 inputs og bit-pakningen. Bemærk, hvordan node_can_id bruges dynamic.

```yaml
# can_sensor_node.yaml
substitutions:
  node_can_id: "0x21" # Standard ID hvis intet andet angives på den enkelte node

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
          // Sender til det unikke CAN ID defineret i substitutions
          id(can_bus).send(${node_can_id}, data);

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

### Step 2: Sådan ser dine sensornoder ud nu (Ekstremt korte!)

Nu bliver det super let at oprette dine 4 eller 8 sensornoder. Deres filer skal blot se således ud:

Node 2 (Første sensornode - f.eks. esp32_node2.yaml):

```yaml
esphome:
  name: esp32-sensor-node-2

packages:
  sensor_node_base: !include can_sensor_node.yaml

substitutions:
  node_can_id: "0x21" # Unikt CAN ID for denne node
```

Node 4 (Næste sensornode - f.eks. esp32_node4.yaml):

```yaml
esphome:
  name: esp32-sensor-node-4

packages:
  sensor_node_base: !include can_sensor_node.yaml

substitutions:
  node_can_id: "0x22" # Næste ledige CAN ID
```

(Du fortsætter bare herfra op til Node 8 eller 9, hvor du blot tæller node_can_id op til 0x23, 0x24, 0x25 osv.)

### Step 3: Den komplette Waveshare Gateway (esp32_gateway.yaml)

Her er den fulde, færdige kode til din Waveshare ESP32-S3-ETH PoE. Jeg har lavet modtager-strukturen klar til 4 noder (0x21 til 0x24), så du kan se mønsteret. Hvis du vil udvide til 8 noder, kopierer du blot en hel can_id blok under on_frame og tilføjer de tilsvarende sensorer i bunden.

```yaml
esphome:
  name: esp32-can-gateway
  comment: "Waveshare ESP32-S3-ETH PoE Master Gateway"

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

# --- CAN-BUS MODTAGER CENTRAL ---
canbus:
  - id: can_bus
    on_frame:
      # --- MODTAG SENSORER FRA NODE 2 (ID 0x21) ---
      - can_id: 0x21
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(n2_s0).publish_state((x >> 0) & 1); id(n2_s1).publish_state((x >> 1) & 1);
                id(n2_s2).publish_state((x >> 2) & 1); id(n2_s3).publish_state((x >> 3) & 1);
                id(n2_s4).publish_state((x >> 4) & 1); id(n2_s5).publish_state((x >> 5) & 1);
                id(n2_s6).publish_state((x >> 6) & 1); id(n2_s7).publish_state((x >> 7) & 1);
              }

      # --- MODTAG SENSORER FRA NODE 4 (ID 0x22) ---
      - can_id: 0x22
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(n4_s0).publish_state((x >> 0) & 1); id(n4_s1).publish_state((x >> 1) & 1);
                id(n4_s2).publish_state((x >> 2) & 1); id(n4_s3).publish_state((x >> 3) & 1);
                id(n4_s4).publish_state((x >> 4) & 1); id(n4_s5).publish_state((x >> 5) & 1);
                id(n4_s6).publish_state((x >> 6) & 1); id(n4_s7).publish_state((x >> 7) & 1);
              }

      # --- MODTAG SENSORER FRA NODE 5 (ID 0x23) ---
      - can_id: 0x23
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(n5_s0).publish_state((x >> 0) & 1); id(n5_s1).publish_state((x >> 1) & 1);
                id(n5_s2).publish_state((x >> 2) & 1); id(n5_s3).publish_state((x >> 3) & 1);
                id(n5_s4).publish_state((x >> 4) & 1); id(n5_s5).publish_state((x >> 5) & 1);
                id(n5_s6).publish_state((x >> 6) & 1); id(n5_s7).publish_state((x >> 7) & 1);
              }

      # --- MODTAG SENSORER FRA NODE 6 (ID 0x24) ---
      - can_id: 0x24
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(n6_s0).publish_state((x >> 0) & 1); id(n6_s1).publish_state((x >> 1) & 1);
                id(n6_s2).publish_state((x >> 2) & 1); id(n6_s3).publish_state((x >> 3) & 1);
                id(n6_s4).publish_state((x >> 4) & 1); id(n6_s5).publish_state((x >> 5) & 1);
                id(n6_s6).publish_state((x >> 6) & 1); id(n6_s7).publish_state((x >> 7) & 1);
              }

      # --- MODTAG FEEDBACK FRA OUTPUT NODE 3 (ID 0x31) ---
      - can_id: 0x31
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(node3_sw_0).publish_state((x >> 0) & 1); id(node3_sw_1).publish_state((x >> 1) & 1);
                id(node3_sw_2).publish_state((x >> 2) & 1); id(node3_sw_3).publish_state((x >> 3) & 1);
                id(node3_sw_4).publish_state((x >> 4) & 1); id(node3_sw_5).publish_state((x >> 5) & 1);
                id(node3_sw_6).publish_state((x >> 6) & 1); id(node3_sw_7).publish_state((x >> 7) & 1);
              }

# --- HOME ASSISTANT SENSOR-ENTITETER ---
binary_sensor:
  # Node 2 Sensorer
  - platform: template
    name: "Node 2 - Sensor 1"
    id: n2_s0
  - platform: template
    name: "Node 2 - Sensor 2"
    id: n2_s1
  - platform: template
    name: "Node 2 - Sensor 3"
    id: n2_s2
  - platform: template
    name: "Node 2 - Sensor 4"
    id: n2_s3
  - platform: template
    name: "Node 2 - Sensor 5"
    id: n2_s4
  - platform: template
    name: "Node 2 - Sensor 6"
    id: n2_s5
  - platform: template
    name: "Node 2 - Sensor 7"
    id: n2_s6
  - platform: template
    name: "Node 2 - Sensor 8"
    id: n2_s7

  # Node 4 Sensorer
  - platform: template
    name: "Node 4 - Sensor 1"
    id: n4_s0
  - platform: template
    name: "Node 4 - Sensor 2"
    id: n4_s1
  - platform: template
    name: "Node 4 - Sensor 3"
    id: n4_s2
  - platform: template
    name: "Node 4 - Sensor 4"
    id: n4_s3
  - platform: template
    name: "Node 4 - Sensor 5"
    id: n4_s4
  - platform: template
    name: "Node 4 - Sensor 6"
    id: n4_s5
  - platform: template
    name: "Node 4 - Sensor 7"
    id: n4_s6
  - platform: template
    name: "Node 4 - Sensor 8"
    id: n4_s7

  # Node 5 Sensorer
  - platform: template
    name: "Node 5 - Sensor 1"
    id: n5_s0
  - platform: template
    name: "Node 5 - Sensor 2"
    id: n5_s1
  - platform: template
    name: "Node 5 - Sensor 3"
    id: n5_s2
  - platform: template
    name: "Node 5 - Sensor 4"
    id: n5_s3
  - platform: template
    name: "Node 5 - Sensor 5"
    id: n5_s4
  - platform: template
    name: "Node 5 - Sensor 6"
    id: n5_s5
  - platform: template
    name: "Node 5 - Sensor 7"
    id: n5_s6
  - platform: template
    name: "Node 5 - Sensor 8"
    id: n5_s7

  # Node 6 Sensorer
  - platform: template
    name: "Node 6 - Sensor 1"
    id: n6_s0
  - platform: template
    name: "Node 6 - Sensor 2"
    id: n6_s1
  - platform: template
    name: "Node 6 - Sensor 3"
    id: n6_s2
  - platform: template
    name: "Node 6 - Sensor 4"
    id: n6_s3
  - platform: template
    name: "Node 6 - Sensor 5"
    id: n6_s4
  - platform: template
    name: "Node 6 - Sensor 6"
    id: n6_s5
  - platform: template
    name: "Node 6 - Sensor 7"
    id: n6_s6
  - platform: template
    name: "Node 6 - Sensor 8"
    id: n6_s7

# --- HOME ASSISTANT STYRING TIL NODE 3 ---
switch:
  - platform: template
    name: "Node 3 - Relæ 1"
    id: node3_sw_0
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
  - platform: template
    name: "Node 3 - Relæ 2"
    id: node3_sw_1
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
  - platform: template
    name: "Node 3 - Relæ 3"
    id: node3_sw_2
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
  - platform: template
    name: "Node 3 - Relæ 4"
    id: node3_sw_3
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
  - platform: template
    name: "Node 3 - Relæ 5"
    id: node3_sw_4
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
  - platform: template
    name: "Node 3 - Relæ 6"
    id: node3_sw_5
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
  - platform: template
    name: "Node 3 - Relæ 7"
    id: node3_sw_6
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
  - platform: template
    name: "Node 3 - Relæ 8"
    id: node3_sw_7
    turn_on_action:  - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
    turn_off_action: - canbus.send: { canbus_id: can_bus, can_id: 0x30, data: }
```

Nu er dit system fuldt skalerbart! Skal du have hjælp til at udvide gateway-koden med de sidste noder (op til 8 noder i alt), eller har du fanget mønsteret og vil gå i gang med at teste?


