# Split i flere ESPHome Package

## Er det en ide at have display delen i egen package ?

Ja, det er en rigtig god idé at splitte koden op, så display-delen ligger i sin helt egen pakke [INDEX]. Det giver dig tre store fordele på din modelbane:

1. Nem fejlfinding: Hvis en skærm går i stykker, eller du vil skifte til en anden skærmtype på en enkelt station, skal du kun rette i display-filen.
2. Renere kode: Dine slave-filer (f.eks. station_nord.yaml) bliver ekstremt korte og nemme at overskue.
3. Fleksibilitet: Du kan have 5 stationer med TFT-skærme og 4 stationer med OLED-skærme, men de kan stadig genbruge den samme grundlæggende slave-logik for knapper og Home Assistant-forbindelse.
For at gøre dette, deler vi slaven op i to separate pakker: Logik-pakken og Display-pakken.

### 1. Grundpakken: fast_clock_slave_core.yaml

Opret denne fil. Den indeholder udelukkende forbindelsen til Home Assistant, sensorerne og den fysiske knap.

```yaml
# fast_clock_slave_core.yaml

# Hent data fra Master-uret via Home Assistant
text_sensor:
  - platform: homeassistant
    name: "Master Time"
    entity_id: text_sensor.accelerated_time
    id: master_time

  - platform: homeassistant
    name: "Master Date"
    entity_id: text_sensor.accelerated_date
    id: master_date

  - platform: homeassistant
    name: "Hvem pausede uret"
    entity_id: input_text.clock_paused_by
    id: paused_by_who

binary_sensor:
  # Fysisk knap forbundet til GPIO14 på denne slave-enhed
  - platform: gpio
    pin:
      number: GPIO14
      mode: INPUT_PULLUP
    name: "Fysisk Betjeningsknap"
    on_click:
      then:
        - homeassistant.service:
            service: script.clock_toggle_global
            data:
              device_name: ${node_friendly_name}

  - platform: homeassistant
    name: "Master Running Status"
    entity_id: switch.ur_status_start_pause
    id: master_running_status
```

### 2. Displaypakken: display_tft_18.yaml

Opret denne fil. Den indeholder kun SPI-opsætningen, skrifttyperne og selve tegningen af det analoge ur. Den "kigger" automatisk over i sensorerne fra core-pakken.

```yaml
# display_tft_18.yaml

spi:
  clk_pin: GPIO18
  mosi_pin: GPIO23

display:
  - platform: st7735
    model: "INITR_BLACKTAB"
    cs_pin: GPIO5
    dc_pin: GPIO19
    reset_pin: GPIO4
    rotation: 0 
    id: tft_display
    lambda: |-
      Color color_white = Color(255, 255, 255);
      Color color_black = Color(0, 0, 0);
      Color color_red   = Color(255, 0, 0);
      Color color_gray  = Color(100, 100, 100);
      
      // ANALOGT UR I TOPPEN
      int cx = 64; int cy = 50; int r = 40;
      it.circle(cx, cy, r, color_white);
      it.circle(cx, cy, r-1, color_white);
      
      it.line(cx, cy - r, cx, cy - r + 5, color_white);
      it.line(cx, cy + r, cx, cy + r - 5, color_white);
      it.line(cx - r, cy, cx - r + 5, cy, color_white);
      it.line(cx + r, cy, cx + r - 5, cy, color_white);

      if (id(master_time).has_state()) {
        int hr, mn, sc;
        if (sscanf(id(master_time).state.c_str(), "%d:%d:%d", &hr, &mn, &sc) == 3) {
          float angle_s = (sc * 6) * M_PI / 180.0;
          it.line(cx, cy, cx + (r - 4) * sin(angle_s), cy - (r - 4) * cos(angle_s), color_red);
          
          float angle_m = (mn * 6 + sc * 0.1) * M_PI / 180.0;
          it.line(cx, cy, cx + (r - 8) * sin(angle_m), cy - (r - 8) * cos(angle_m), color_white);
          
          float angle_h = ((hr % 12) * 30 + mn * 0.5) * M_PI / 180.0;
          it.line(cx, cy, cx + (r - 16) * sin(angle_h), cy - (r - 16) * cos(angle_h), color_white);
          
          it.filled_circle(cx, cy, 2, color_red);
        }
      }

      // TEKST I BUNDEN
      it.line(0, 102, 127, 102, color_gray);
      
      if (id(master_running_status).state) {
        it.print(64, 106, id(font_small), color_white, TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        std::string pause_text = "PAUSE: " + id(paused_by_who).state;
        it.print(64, 106, id(font_small), color_red, TextAlign::TOP_CENTER, pause_text.c_str());
      }
      
      it.printf(64, 120, id(font_medium), color_white, TextAlign::TOP_CENTER, "%s", id(master_time).state.c_str());
      it.printf(64, 144, id(font_small), color_gray, TextAlign::TOP_CENTER, "%s", id(master_date).state.c_str());

font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_medium
    size: 16

# Tving skærmen til at opdatere, når tiden ændrer sig
text_sensor:
  - id: master_time
    on_value:
      then:
        - component.update: tft_display
```

### 3. Sådan ser din slave-fil ud nu (station_nord.yaml)

Når du nu skal oprette dine slavenoder, trækker du blot begge pakker ind i hovedfilen. Det bliver super kort og elegant:

```yaml
esphome:
  name: station-nord

esp32:
  board: esp32dev

substitutions:
  node_friendly_name: "Station Nord"

# HENT BEGGE PAKKER IND HER:
packages:
  slave_core: !include fast_clock_slave_core.yaml
  slave_display: !include display_tft_18.yaml

# Normale netværksindstillinger
wifi:
  ssid: "DitWiFi"
  password: "DitPassword"

api:
logger:
```

Nu er koden fuldstændig modulær! Hvis du på et tidspunkt vil lave en display_oled_128x64.yaml pakke til en af de andre stationer, kan du bare skifte den ud på den specifikke enhed uden at røre ved logikken. Er du klar til at gemme filerne og kaste dig over den første test-upload?

## Hvad med at til føje EN PAKKE MED display_oled_128x64.

* Det er en fremragende idé! Ved at lave en display_oled_128x64.yaml pakke, har du nu det perfekte mix-and-match system. Hvis en af dine stationer skal bruge en I2C OLED-skærm i stedet for en SPI TFT-farveskærm, ændrer du blot én enkelt linje i den stationens hovedfil.
* Her er koden til den nye display-pakke. Den passer direkte sammen med den samme fast_clock_slave_core.yaml pakke, du allerede har lavet:

### 1. Opret display-pakken: display_oled_128x64.yaml

Opret en ny fil i din ESPHome-mappe, kald den display_oled_128x64.yaml og indsæt følgende kode:

```yaml
# display_oled_128x64.yaml

# Aktiver I2C-bussen på de korrekte standardben for din ESP32 HW-394
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Konfiguration af SSD1306 128x64 OLED-skærmen
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    lambda: |-
      // Tegn en pæn vandret skillelinje under status/overskrift
      it.line(0, 15, 127, 15);
      
      // Vis status (MODELBANE UR eller PAUSE: Station)
      if (id(master_running_status).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        std::string pause_msg = "|| PAUSE: " + id(paused_by_who).state;
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, pause_msg.c_str());
      }
      
      // Vis den digitale tid fra Master-uret (Stort i midten)
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(master_time).state.c_str());
      
      // Vis datoen helt i bunden
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(master_date).state.c_str());

# Skrifttyper (Fonts)
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

# Tving OLED-skærmen til at opdatere med det samme sekunderne tikker ind fra Masteren
text_sensor:
  - id: master_time
    on_value:
      then:
        - component.update: oled_display
```

### 2. Sådan vælger du skærmtype på dine stationer

Nu bliver det utrolig nemt at styre dine 9 slaver. Du genbruger blot fast_clock_slave_core.yaml på dem alle, men trækker den display-pakke ind, som matcher hardwaren:
Eksempel A: En station MED den nye OLED-skærm (station_syd.yaml)

```yaml
esphome:
  name: station-syd

esp32:
  board: esp32dev

substitutions:
  node_friendly_name: "Station Syd"

packages:
  slave_core: !include fast_clock_slave_core.yaml
  slave_display: !include display_oled_128x64.yaml # <--- Trækker OLED-pakken ind!

# wifi, api, logger, osv...
Vær forsigtig med at bruge denne kode.
Eksempel B: En station MED TFT-farveskærmen (station_nord.yaml)
yaml
esphome:
  name: station-nord

esp32:
  board: esp32dev

substitutions:
  node_friendly_name: "Station Nord"

packages:
  slave_core: !include fast_clock_slave_core.yaml
  slave_display: !include display_tft_18.yaml # <--- Trækker TFT-pakken ind!

# wifi, api, logger, osv...
```

### Samlet overblik over dit nye ur-system:

* fast_clock_master.yaml (Kører ur-logikken, H-broen og sit eget OLED-display).
* fast_clock_slave_core.yaml (Håndterer netværk, knap-tryk og modtager tid).
* display_tft_18.yaml (Pakke til stationer med analogt ur på TFT-skærm).
* display_oled_128x64.yaml (Pakke til stationer med digitalt ur på OLED-skærm).

Nu er hele dit modelbane-ur-økosystem komplet og fuldstændig modulært. Er der andre displays, skærmlayouts eller måske nogle lyseffekter/relæer (f.eks. til at tænde stationsbelysning om natten), vi skal tilføje som en pakke, før du går i gang med at flashe dine ESP32'ere?

