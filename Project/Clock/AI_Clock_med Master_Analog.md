# AI Clock med Master og Analog Ur

## I will like the slave to use a 1.8 tft module 128x160 pixel display med spi interface, jeg ønsker det i protrætmode og et analog ur i toppen og tekst under

* For at bruge en 1.8" TFT-skærm (128x160) med ST7735-driver og SPI-interface på dine slave-enheder, skal vi opdatere fast_clock_slave.yaml.
* Da skærmen skal være i portræt-tilstand (portrait), roterer vi billedet 90 eller 270 grader, så vi får 128 pixels i bredden og 160 pixels i højden. På en farveskærm kan vi tegne et flot analogt ur i toppen (med urskive, time-, minut- og sekundviser) og placere den digitale tid og dato som tekst i bunden.

* Her er de anbefalede ben (pins) til SPI-skærmen på en ESP32 HW-394:
  * CLK (SPI Clock): GPIO18
  * DATA / MOSI: GPIO23
  * CS (Chip Select): GPIO5
  * DC (Data/Command): GPIO19
  * RESET: GPIO4
  * LED / BL (Baggrundslys): Forbindes til 3.3V (eller en GPIO, hvis du vil kunne slukke det).

### Opdateret fast_clock_slave.yaml til TFT SPI-skærm

```yaml
# fast_clock_slave.yaml

# Aktiver SPI-bussen på standard hardware-pins for ESP32
spi:
  clk_pin: GPIO18
  mosi_pin: GPIO23

# Konfiguration af ST7735 1.8" TFT-skærmen
display:
  - platform: st7735
    model: "INITR_BLACKTAB" # Prøv GREENTAB eller REDTAB hvis farverne er byttet om
    cs_pin: GPIO5
    dc_pin: GPIO19
    reset_pin: GPIO4
    rotation: 0 # Sætter skærmen i portrætmode (128x160)
    id: tft_display
    lambda: |-
      // Farve-definitioner (RGB565)
      Color color_white = Color(255, 255, 255);
      Color color_black = Color(0, 0, 0);
      Color color_red   = Color(255, 0, 0);
      Color color_gray  = Color(100, 100, 100);
      Color color_bg    = Color(20, 20, 30); // Mørkeblålig baggrund
      
      // 1. ANALOGT UR I TOPPEN
      int cx = 64;  // Center X for uret
      int cy = 50;  // Center Y for uret
      int r = 40;   // Radius på uret
      
      // Tegn urskiven (Cirkel)
      it.circle(cx, cy, r, color_white);
      it.circle(cx, cy, r-1, color_white);
      
      // Tegn små markeringer for klokken 12, 3, 6, 9
      it.line(cx, cy - r, cx, cy - r + 5, color_white);
      it.line(cx, cy + r, cx, cy + r - 5, color_white);
      it.line(cx - r, cy, cx - r + 5, cy, color_white);
      it.line(cx + r, cy, cx + r - 5, cy, color_white);

      // Beregn visere ud fra Master-tiden (hvis vi har modtaget data)
      if (id(master_time).has_state()) {
        int hr, mn, sc;
        // Udtræk timer, minutter og sekunder fra den modtagne tekststreng (HH:MM:SS)
        if (sscanf(id(master_time).state.c_str(), "%d:%d:%d", &hr, &mn, &sc) == 3) {
          
          // Sekundviser (Rød og tynd)
          float angle_s = (sc * 6) * M_PI / 180.0;
          int sx = cx + (r - 4) * sin(angle_s);
          int sy = cy - (r - 4) * cos(angle_s);
          it.line(cx, cy, sx, sy, color_red);
          
          // Minutviser (Hvid)
          float angle_m = (mn * 6 + sc * 0.1) * M_PI / 180.0;
          int mx = cx + (r - 8) * sin(angle_m);
          int my = cy - (r - 8) * cos(angle_m);
          it.line(cx, cy, mx, my, color_white);
          
          // Timeviser (Tynd/Kort eller Grå)
          float angle_h = ((hr % 12) * 30 + mn * 0.5) * M_PI / 180.0;
          int hx = cx + (r - 16) * sin(angle_h);
          int hy = cy - (r - 16) * cos(angle_h);
          it.line(cx, cy, hx, hy, color_white);
          
          // Center-prik på uret
          it.filled_circle(cx, cy, 2, color_red);
        }
      }

      // 2. TEKST OG STATUS I BUNDEN
      it.line(0, 102, 127, 102, color_gray); // Skillelinje
      
      // Vis status (MODELBANE UR eller PAUSE)
      if (id(master_running_status).state) {
        it.print(64, 106, id(font_small), color_white, TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        std::string pause_text = "PAUSE: " + id(paused_by_who).state;
        it.print(64, 106, id(font_small), color_red, TextAlign::TOP_CENTER, pause_text.c_str());
      }
      
      // Digital tidsvisning under status
      it.printf(64, 120, id(font_medium), color_white, TextAlign::TOP_CENTER, "%s", id(master_time).state.c_str());
      
      // Dato helt i bunden
      it.printf(64, 144, id(font_small), color_gray, TextAlign::TOP_CENTER, "%s", id(master_date).state.c_str());

# Skrifttyper
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_medium
    size: 16

# Hent data fra Master-uret via Home Assistant
text_sensor:
  - platform: homeassistant
    name: "Master Time"
    entity_id: text_sensor.accelerated_time
    id: master_time
    on_value:
      then:
        - component.update: tft_display

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
    on_state:
      then:
        - component.update: tft_display
```

### Ledningsdiagram for 1.8" TFT til din ESP32 Slave:

1. VCC → 3V3 eller 5V (afhængigt af dit skærmmodul)
2. GND → GND
3. CS → GPIO5
4. RESET → GPIO4
5. A0 / DC → GPIO19
6. SDA / mosi → GPIO23
7. SCK / clk → GPIO18
8. LED / BL → 3V3 (for fuld lysstyrke på skærmen)

#### Hvad koden gør på skærmen:

* I området fra Y=0 til Y=100 tegner ESP32 et rundt analogt ur med matematisk beregnede visere, der flytter sig flydende og synkront i takt med, at det hurtige modelbane-ur tikker afsted.
* Når der trykkes på pause, lyser navnet på den pågældende station op med rød tekst lige under det analoge ur, så det fanger opmærksomheden med det samme.

Når du uploader koden til ST7735-skærme, kan farverne (f.eks. rød og blå) nogle gange være byttet om på grund af forskellige skærm-revisioner på markedet. Hvis det sker for dig, skal vi blot ændre model: "INITR_BLACKTAB" til en af de andre farve-tabs. Skal vi beholde det mørke farvetema, eller vil du have tilpasset farverne på urskiven?
