# Beskrivelse af kode

* Det er en helt naturlig og meget fornuftig beslutning at gå tilbage til 10 adskilte CAN-busser – én for hver station. Når et anlæg har den størrelse, er fejlsikkerhed og nem fejlfinding guld værd, og med kablet LAN til hver station mister du reelt ikke noget mærkbart i hastighed.
* Her er det opdaterede overblik over, hvordan dit system nu hænger sammen, og hvordan du bevarer den lynhurtige synkronisering mellem signaler, oversigtstavler og stationspulter:

## 1. Den nye netværksstruktur (10 separate "øer")

* 10 selvstændige netværk: Hver station er sin egen lukkede "ø" med sin egen korte CAN-bus (kun et par meter lang mellem stationens 4+ ESP32-S3 noder).
* Terminering: Du skal nu huske, at hver af de 10 korte CAN-busser skal have en 120 Ω modstand i hver ende (20 modstande i alt på hele anlægget).
* Genbrug af CAN ID'er: Da de 10 busser ikke længere er forbundet fysisk, kan du bruge nøjagtigt de samme CAN ID'er på Station 1, Station 2, Station 3 osv. (f.eks. 0x010 for Node 1, 0x020 for Node 2). Det gør dine ESPHome-filer ekstremt ensartede og nemme at vedligeholde.

## 2. Sådan bevarer du synkroniseringen af signaler og tavler

* Tidligere klarede den lange CAN-bus broadcast-funktionen automatisk. Nu er det Home Assistant, der agerer "central hjerne" og binder de 10 LAN-gateways sammen:
* Fra spor til Home Assistant: Når en DCC-sporbesatmelder på Station 2 registrerer et tog, sender den beskeden over Station 2's korte CAN-bus op til Station 2's LAN-gateway, som straks melder status til Home Assistant via netværkskablet.
* Fra Home Assistant til tavler og pulter: Home Assistant har en simpel automatisering (eller et script for hvert signalbillede). Når et signal skal vise "Kør" (0x01), sender Home Assistant kommandoen ud via LAN til de relevante stationers gateways på én gang (f.eks. til Station 2's hovedsignal, til oversigtstavlens gateway og til rangerpulten).
* Hastighed: Over kablet Ethernet tager denne runde gennem Home Assistant typisk under 10–20 millisekunder. For det menneskelige øje vil signalet på anlægget, lokoførernes oversigtstavle og rangerpersonalets pult stadig skifte fuldstændig samtidigt.

## 3. Fordele ved denne løsning i hverdagen

* Ingen "single point of failure": Hvis en mus gnaver en ledning over på Station 4, eller en transceiver brænder af på Station 7, kører resten af anlægget videre uden problemer.
* Lynhurtig fejlfinding: Hvis en node ikke svarer, ved du med 100 % sikkerhed, at fejlen ligger lokalt under dén specifikke station og ikke et tilfældigt sted på 200 meter kabel.
* Enklere kabelføring: Du slipper for at trække følsomme CAN-bus kabler over lange afstande mellem stationerne og nøjes med almindelige, skærmede netværkskabler (LAN) hen til hver stations LAN-gateway.
* Nu hvor arkitekturen ligger fast med 10 separate LAN-gateways, vil du så have et eksempel på, hvordan du opsætter Home Assistant-automatiseringen, der sender signalbilledet ud til flere gateways på én gang, eller skal vi kigge på knap-inputs til stationspulterne?

## jeg vil gerne have et overblik over koden til noderne

Nu hvor du har valgt den opdelte model med 10 separate CAN-busser (én pr. station), bliver din kode struktur utrolig ren, overskuelig og genbrugelig. Fordi hver station er sit eget lukkede netværk, kan du bruge nøjagtigt den samme kode-struktur på tværs af alle 10 stationer.
Her er det komplette overblik over de tre filtyper, du skal bruge i ESPHome for at drive hele dit anlæg.

### 📂 Filstruktur i ESPHome

For at holde styr på dine 40+ noder og 10 gateways, opretter du blot disse filer i din ESPHome-mappe:
1. station_sub_node.yaml (1 fil i alt): Fællespakken med alt logikken til de decentrale ESP32-S3 noder.
2. stX_nodeY.yaml (40+ filer): Ultrakorte filer til de enkelte noder på stationerne (f.eks. st1_node1.yaml).
3. stationXX_lan.yaml (10 filer): Koden til hver stations LAN-gateway, som oversætter mellem stationens CAN-bus og Home Assistant.

### 1. Fællespakken: station_sub_node.yaml

Denne fil gemmes én gang og deles af alle dine ESP32-S3 noder på tværs af samtlige stationer. Den håndterer sporskiftestyring (ULN2803A), mekanisk dobbelt-feedback (optokoblere), DCC-sporbesatmelder (med dit hardware-debounce filter) og 3-aspekt lyssignaler.

```yaml

# station_sub_node.yaml (DELES AF ALLE ESP32-S3 NODER PÅ ALLE STATIONER)

esphome:
  comment: "Decentral ESP32-S3 CAN-node"

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf

# Standardværdier, som overskrives af den enkelte nodes egen fil
substitutions:
  can_id_sporskifte: "0x010" # Modtag kommando til sporskifte
  can_id_signal:     "0x011" # Modtag kommando til signalbillede
  can_id_fb_spor:    "0x012" # Send mekanisk position retur
  can_id_fb_bod:     "0x013" # Send DCC sporbesat retur

# Lokal CAN-bus på stationen
canbus:
  - id: lokal_can
    platform: esp32_can
    rx_pin: GPIO4
    tx_pin: GPIO5
    bit_rate: 125kbps
    on_frame:
      # 1. Skift sporskifte (0x01 = Normal, 0x00 = Afvigende)
      - can_id: ${can_id_sporskifte}
        then:
          - lambda: |-
              if (x.size() > 0) {
                if (x[0] == 0x01) id(transversal_motor).turn_on();
                else if (x[0] == 0x00) id(transversal_motor).turn_off();
              }

      # 2. Skift signalbillede (0x00 = Rød, 0x01 = Grøn, 0x02 = Gul)
      - can_id: ${can_id_signal}
        then:
          - lambda: |-
              if (x.size() > 0) {
                if (x[0] == 0x00) {
                  id(signal_roed).turn_on();
                  id(signal_groen).turn_off();
                  id(signal_gul).turn_off();
                } else if (x[0] == 0x01) {
                  id(signal_roed).turn_off();
                  id(signal_groen).turn_on();
                  id(signal_gul).turn_off();
                } else if (x[0] == 0x02) {
                  id(signal_roed).turn_off();
                  id(signal_groen).turn_off();
                  id(signal_gul).turn_on();
                }
              }

# I2C til MCP23017
i2c:
  sda: GPIO1
  scl: GPIO2

mcp23017:
  - id: mcp_hub
    address: 0x20

# Styring af Tortoise via ULN2803A (med 330 Ohm pull-up til 12V)
switch:
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 0, mode: OUTPUT }
    id: uln_a
    internal: true
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 1, mode: OUTPUT }
    id: uln_b
    internal: true

  - platform: template
    id: transversal_motor
    turn_on_action:
      - switch.turn_off: uln_b
      - switch.turn_on: uln_a
    turn_off_action:
      - switch.turn_off: uln_a
      - switch.turn_on: uln_b

# 3-Aspekt Lyssignaler (Tilsluttet ESP32-S3 GPIO med modstande)
light:
  - platform: binary
    id: signal_roed
    output: out_r
  - platform: binary
    id: signal_groen
    output: out_g
  - platform: binary
    id: signal_gul
    output: out_y

output:
  - platform: gpio
    pin: GPIO6
    id: out_r
  - platform: gpio
    pin: GPIO7
    id: out_g
  - platform: gpio
    pin: GPIO15
    id: out_y

# Tilbagemeldinger fra banen (Optokoblere + Analog DCC-føler)
binary_sensor:
  # Mekanisk position: Normal
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 8, mode: INPUT_PULLUP, inverted: true }
    id: fb_normal
    filters:
      - delayed_on: 0.1s
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x01] }

  # Mekanisk position: Afvigende
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 9, mode: INPUT_PULLUP, inverted: true }
    id: fb_afvigende
    filters:
      - delayed_on: 0.1s
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x02] }

  # DCC Sporbesatmelder (Hardware-debounce + software-filter mod snavs)
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 10, mode: INPUT_PULLUP, inverted: true }
    id: dcc_bod
    filters:
      - delayed_on: 0.1s
      - delayed_off: 2.5s
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_bod}, data: [0x01] }
    on_release:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_bod}, data: [0x00] }
```

### 2. Den enkelte nodes fil: st1_node1.yaml

Når du opretter en node, skal filen kun indeholde navnet og de fire lokale CAN ID'er. Fordi hver station har sin egen CAN-bus, kan Node 1 på Station 1 og Node 1 på Station 2 bruge præcis de samme CAN ID'er (0x010 – 0x013).

```yaml
# st1_node1.yaml (STATION 1 - NODE 1)

esphome:
  name: st1-node1

packages:
  base:
    file: station_sub_node.yaml

substitutions:
  can_id_sporskifte: "0x010"
  can_id_signal:     "0x011"
  can_id_fb_spor:    "0x012"
  can_id_fb_bod:     "0x013"
```

For Node 2 på samme station ændrer du blot det midterste ciffer til 2 (0x020, 0x021, 0x022, 0x023), og så videre for Node 3 og Node 4.

### 3. LAN-Gatewayen pr. station: station01_lan.yaml

Denne kode lægges på stationens Waveshare ESP32-S3-ETH. Den oversætter CAN-trafikken på den lokale station til rigtige knapper, sensorer og valgmuligheder i Home Assistant.

```yaml
# station01_lan.yaml (LAN-BRO FOR STATION 1)

esphome:
  name: station01-lan
  friendly_name: "Station 01"

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf

ethernet:
  type: W5500
  mdc_pin: GPIO23
  mdio_pin: GPIO18
  clk_mode: GPIO0_IN

api:
ota:
logger:

canbus:
  - id: lokal_can
    platform: esp32_can
    rx_pin: GPIO4
    tx_pin: GPIO5
    bit_rate: 125kbps
    on_frame:
      # Modtag position fra Node 1
      - can_id: 0x012
        then:
          - lambda: |-
              if (x.size() > 0) {
                if (x[0] == 0x01) id(n1_pos).publish_state("Normal");
                else if (x[0] == 0x02) id(n1_pos).publish_state("Afvigende");
              }
      # Modtag sporbesat fra Node 1
      - can_id: 0x013
        then:
          - lambda: |-
              if (x.size() > 0) id(n1_bod).publish_state(x[0] == 0x01);

# --- KOMPONENTER SYNLIGE I HOME ASSISTANT FOR NODE 1 ---
# (Gentages for Node 2, 3 og 4 med deres respektive 0x02x, 0x03x og 0x04x ID'er)

switch:
  - platform: template
    name: "Node 1 Sporskifte"
    icon: "mdi:vector-line"
    turn_on_action:
      - canbus.send: { canbus_id: lokal_can, can_id: 0x010, data: [0x01] }
    turn_off_action:
      - canbus.send: { canbus_id: lokal_can, can_id: 0x010, data: [0x00] }

select:
  - platform: template
    name: "Node 1 Signal"
    icon: "mdi:traffic-light"
    options:
      - "Rød (Stop)"
      - "Grøn (Kør)"
      - "Gul (Kør forsigtigt)"
    optimistic: true
    set_action:
      - lambda: |-
          if (x == "Rød (Stop)") {
            id(lokal_can).send_data(0x011, false, {0x00});
          } else if (x == "Grøn (Kør)") {
            id(lokal_can).send_data(0x011, false, {0x01});
          } else if (x == "Gul (Kør forsigtigt)") {
            id(lokal_can).send_data(0x011, false, {0x02});
          }

text_sensor:
  - platform: template
    name: "Node 1 Position"
    id: n1_pos

binary_sensor:
  - platform: template
    name: "Node 1 Sporbesat"
    device_class: occupancy
    id: n1_bod
```

### 📋 Oversigt over det faste ID-skema (Ens på alle 10 stationer)

|Node på stationen|Sporskifte (Kommando)|Signal (Kommando)|Position (Feedback)|Sporbesat (Feedback)|
|:---|:---:|:---:|:---:|:---:|
|Node 1|0x010|0x011|0x012|0x013|
|Node 2|0x020|0x021|0x022|0x023|
|Node 3|0x030|0x031|0x032|0x033|
|Node 4|0x040|0x041|0x042|0x043|

Fordelen ved select-komponenten i LAN-gatewayen er, at du i Home Assistant får en pæn rullemenu for hvert signal ("Rød", "Grøn", "Gul"), som du nemt kan styre fra dine automatiseringer til både hovedsignaler, oversigtstavler og stationspulter.
Hvis du vil gå videre med opsætningen, kan jeg hjælpe med:
* At udvide station_sub_node.yaml til at styre alle 4 sporskifter på én gang (koden ovenfor viser opsætningen for det første sporskifte på MCP23017).
* At vise Home Assistant-automatiseringen, der synkroniserer signalbilledet ud til flere stationers LAN-gateways samtidig.
AI-svar kan indeholde fejl. Få flere oplysninger

skriv bare koden for 4 sporskifter

Her er den komplette og færdige kode til station_sub_node.yaml, som udnytter alle 16 ben på din MCP23017 til at styre 4 uafhængige sporskifter og læse dobbelt mekanisk feedback for hvert eneste af dem.
Udgangene (0–7) går direkte til din ULN2803A, og indgangene (8–15) læser fra dine optokoblere. Til sporbesat-melderne bruger koden ledige GPIO-pinde på din ESP32-S3, da MCP23017 nu er fuldt udnyttet.

Dataprotokol for CAN-bussen (4 sporskifter på én node)

For at holde CAN-trafikken enkel sender og modtager denne node 2 bytes data pr. pakke:
* Byte 0 (x[0]): Sporskiftets nummer (0x01, 0x02, 0x03 eller 0x04).
* Byte 1 (x[1]): Position/Status (0x01 = Normal, 0x00 / 0x02 = Afvigende).

📦 Kode til station_sub_node.yaml (Alle 4 sporskifter)

```yaml
# station_sub_node.yaml (KOMPLET 4-SPORSKIFTE VERSION)

esphome:
  comment: "ESP32-S3 CAN-node med 4 sporskifter og dobbelt feedback"

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf

substitutions:
  can_id_sporskifte: "0x010" # Modtag kommando: [Sporskifte 1-4, Retning 0-1]
  can_id_signal:     "0x011" # Modtag signalbillede: [0=Rød, 1=Grøn, 2=Gul]
  can_id_fb_spor:    "0x012" # Send feedback: [Sporskifte 1-4, Position 1-2]
  can_id_fb_bod:     "0x013" # Send sporbesat: [Sektion 1-2, Besat 0-1]

canbus:
  - id: lokal_can
    platform: esp32_can
    rx_pin: GPIO4
    tx_pin: GPIO5
    bit_rate: 125kbps
    on_frame:
      # 1. Modtag kommando til et af de 4 sporskifter
      - can_id: ${can_id_sporskifte}
        then:
          - lambda: |-
              if (x.size() >= 2) {
                uint8_t sporskifte = x[0];
                bool retning = (x[1] == 0x01);
                
                if (sporskifte == 1) {
                  if (retning) id(spors_1).turn_on(); else id(spors_1).turn_off();
                } else if (sporskifte == 2) {
                  if (retning) id(spors_2).turn_on(); else id(spors_2).turn_off();
                } else if (sporskifte == 3) {
                  if (retning) id(spors_3).turn_on(); else id(spors_3).turn_off();
                } else if (sporskifte == 4) {
                  if (retning) id(spors_4).turn_on(); else id(spors_4).turn_off();
                }
              }

      # 2. Modtag kommando til signalbillede
      - can_id: ${can_id_signal}
        then:
          - lambda: |-
              if (x.size() > 0) {
                id(signal_roed).turn_off();
                id(signal_groen).turn_off();
                id(signal_gul).turn_off();
                
                if (x[0] == 0x00) id(signal_roed).turn_on();
                else if (x[0] == 0x01) id(signal_groen).turn_on();
                else if (x[0] == 0x02) id(signal_gul).turn_on();
              }

i2c:
  sda: GPIO1
  scl: GPIO2

mcp23017:
  - id: mcp_hub
    address: 0x20

# ==============================================================================
# UDGANGE: MCP23017 Ben 0-7 til ULN2803A (4 sporskifter x 2 udgange)
# ==============================================================================
switch:
  # --- Sporskifte 1 (MCP ben 0 og 1) ---
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 0, mode: OUTPUT }
    id: uln_1a
    internal: true
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 1, mode: OUTPUT }
    id: uln_1b
    internal: true
  - platform: template
    id: spors_1
    turn_on_action:  [switch.turn_off: uln_1b, switch.turn_on: uln_1a]
    turn_off_action: [switch.turn_off: uln_1a, switch.turn_on: uln_1b]

  # --- Sporskifte 2 (MCP ben 2 og 3) ---
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 2, mode: OUTPUT }
    id: uln_2a
    internal: true
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 3, mode: OUTPUT }
    id: uln_2b
    internal: true
  - platform: template
    id: spors_2
    turn_on_action:  [switch.turn_off: uln_2b, switch.turn_on: uln_2a]
    turn_off_action: [switch.turn_off: uln_2a, switch.turn_on: uln_2b]

  # --- Sporskifte 3 (MCP ben 4 og 5) ---
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 4, mode: OUTPUT }
    id: uln_3a
    internal: true
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 5, mode: OUTPUT }
    id: uln_3b
    internal: true
  - platform: template
    id: spors_3
    turn_on_action:  [switch.turn_off: uln_3b, switch.turn_on: uln_3a]
    turn_off_action: [switch.turn_off: uln_3a, switch.turn_on: uln_3b]

  # --- Sporskifte 4 (MCP ben 6 og 7) ---
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 6, mode: OUTPUT }
    id: uln_4a
    internal: true
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 7, mode: OUTPUT }
    id: uln_4b
    internal: true
  - platform: template
    id: spors_4
    turn_on_action:  [switch.turn_off: uln_4b, switch.turn_on: uln_4a]
    turn_off_action: [switch.turn_off: uln_4a, switch.turn_on: uln_4b]

# ==============================================================================
# SIGNALER: ESP32-S3 GPIO 6, 7 og 15
# ==============================================================================
light:
  - platform: binary
    id: signal_roed
    output: out_r
  - platform: binary
    id: signal_groen
    output: out_g
  - platform: binary
    id: signal_gul
    output: out_y

output:
  - platform: gpio
    pin: GPIO6
    id: out_r
  - platform: gpio
    pin: GPIO7
    id: out_g
  - platform: gpio
    pin: GPIO15
    id: out_y

# ==============================================================================
# INDGANGE: MCP23017 Ben 8-15 (Optokoblere) + ESP32 GPIO til DCC Sporbesat
# ==============================================================================
binary_sensor:
  # --- Feedback Sporskifte 1 (MCP ben 8 og 9) ---
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 8, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x01, 0x01] }
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 9, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x01, 0x02] }

  # --- Feedback Sporskifte 2 (MCP ben 10 og 11) ---
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 10, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x02, 0x01] }
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 11, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x02, 0x02] }

  # --- Feedback Sporskifte 3 (MCP ben 12 og 13) ---
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 12, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x03, 0x01] }
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 13, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x03, 0x02] }

  # --- Feedback Sporskifte 4 (MCP ben 14 og 15) ---
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 14, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x04, 0x01] }
  - platform: gpio
    pin: { mcp23017: mcp_hub, number: 15, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_spor}, data: [0x04, 0x02] }

  # --- DCC Sporbesatmeldere (Flyttet til ledige ESP32-S3 GPIO 8 og 9) ---
  - platform: gpio
    pin: { number: GPIO8, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s, delayed_off: 2.5s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_bod}, data: [0x01, 0x01] }
    on_release:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_bod}, data: [0x01, 0x00] }

  - platform: gpio
    pin: { number: GPIO9, mode: INPUT_PULLUP, inverted: true }
    filters: [delayed_on: 0.1s, delayed_off: 2.5s]
    on_press:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_bod}, data: [0x02, 0x01] }
    on_release:
      - canbus.send: { canbus_id: lokal_can, can_id: ${can_id_fb_bod}, data: [0x02, 0x00] }
```

Vil du også have den tilsvarende kode til LAN-gatewayen, der pakker disse 2-byte beskeder ud og viser de 4 sporskifter i Home Assistant?
AI-svar kan indeholde fejl. Få flere oplysninger
