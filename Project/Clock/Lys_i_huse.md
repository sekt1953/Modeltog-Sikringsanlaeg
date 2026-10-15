# Lys i Huse

## lys i vores huse som begynder at tænde fra kl 17 til 19 , jeg tæmker at bruge programbare led ws typen

* Det er en fantastisk idé! Ved at bruge adresserbare WS2812B (eller lignende WS-type) NeoPixel LED-strips kan du styre lyset i hvert enkelt hus uafhængigt. Du kan lade lyset tænde i et tilfældigt mønster mellem kl. 17:00 og 19:00, så det ser realistisk ud – præcis som om folk kommer hjem og tænder lyset i forskellige rum på forskellige tidspunkter.
* Vi kan bygge dette som en valgfri house_lighting.yaml pakke, som du kan tilføje til de ESP32-stationer, der skal styre lys i huse på modelbanen.
* Her er koden til den nye belysningspakke. Den bruger det accelererede ur til at tænde op til 20 huse/vinduer (du kan nemt justere antallet) med en realistisk, varm glødepære-farve.

### 1. Opret lys-pakken: house_lighting.yaml

```yaml
# house_lighting.yaml

# Konfiguration af WS2812B LED-strippen
light:
  - platform: neopixelbus
    type: GRB
    variant: WS2812X
    pin: GPIO25            # <--- Forbind DATA-ledningen fra din LED-strip til denne pin
    num_leds: 20          # <--- Antal huse / vinduer (1 LED pr. hus)
    name: "Modelby Belysning"
    id: city_lights
    restore_mode: ALWAYS_OFF

interval:
  # Tjekker hvert virtuelle minut, om det er tid til at tænde eller slukke lys
  - interval: 1s
    then:
      - lambda: |-
          // Vi tjekker kun lyset, hvis uret rent faktisk kører
          if (id(clock_running).state) {
            time_t raw_time = id(simulated_timestamp);
            struct tm *time_info = gmtime(&raw_time);
            int current_hour = time_info->tm_hour;
            int current_minute = time_info->tm_min;
            
            // Konverter den nuværende virtuelle tid til minutter på dagen (f.eks. kl. 17:00 = 1020 minutter)
            int current_minutes_of_day = (current_hour * 60) + current_minute;
            
            auto call = id(city_lights).turn_on();
            call.set_transition_length(2000); // 2 sekunders blød opvågning af lyset (glødepære-effekt)
            
            // Gå igennem hver enkelt LED (hus)
            for (int i = 0; i < 20; i++) {
              
              // Generer et unikt, men fast tændingstidspunkt for hvert hus mellem kl. 17:00 og 19:00
              // Vi bruger en simpel matematisk formel (hash) baseret på husets nummer (i), så det er det samme hver dag
              int start_delay = (i * 7) % 120; // Giver spredning over 120 minutter (2 timer)
              int tænd_tid = 1020 + start_delay; // 1020 er kl. 17:00 i minutter
              
              // Lad husene slukke igen om natten, f.eks. tilfældigt mellem kl. 23:00 og 01:00 (1380 minutter)
              int stop_delay = (i * 13) % 120;
              int sluk_tid = 1380 + stop_delay;
              
              // Styring af det enkelte hus (LED)
              if (current_minutes_of_day >= tænd_tid && current_minutes_of_day < sluk_tid) {
                // Tænd huset med en flot, varm hvid farve (Rød=255, Grøn=160, Blå=40)
                call.set_led(i, Color(255, 160, 40));
              } else {
                // Sluk lyset i huset
                call.set_led(i, Color(0, 0, 0));
              }
            }
            call.perform();
          }
```

### 2. Sådan tilføjer du lys-pakken til en station

Du kan nu blot koble denne pakke på de stationer, hvor du har monteret en LED-strip under husene. Det gøres super nemt i enhedens hoved-YAML (f.eks. station_nord.yaml):

```yaml
esphome:
  name: station-nord

esp32:
  board: esp32dev

substitutions:
  node_friendly_name: "Station Nord"

# TRÆK AL LOGIK OG BELYSNING IND HER:
packages:
  slave_core: !include fast_clock_slave_core.yaml
  slave_display: !include display_tft_18.yaml
  house_lighting: !include house_lighting.yaml # <--- Tænder for lysstyringen!

# wifi, api, logger, osv...
```

(Bemærk: Hvis du bruger pakken på din Master-enhed i stedet for en slave, skal du blot ændre id(clock_running).state i koden til at matche masterens switch-id, hvis det er anderledes).

💡 Forbindelsesdiagram for WS2812B:

1. 5V (VCC) på LED-strippen → Forbindes til en ekstern 5V strømforsyning (ikke direkte fra ESP32, da den ikke kan levere strøm nok til mange LED'er).
2. GND på LED-strippen → Forbindes til den eksterne strømforsynings GND OG til en GND-pin på din ESP32 (Vigtigt med fælles stel!).
3. DIN (Data In) på LED-strippen → Forbindes til GPIO25 på ESP32.

### Hvorfor denne løsning er smart:

* Realistisk tænding: Hus 1 tænder måske kl. 17:00, Hus 2 kl. 17:07, Hus 3 kl. 17:14 osv. Banen vågner langsomt og hyggeligt til liv i skumringen.
* Tidsstyret: Hvis du sætter uret på pause, fryser lysene i deres nuværende tilstand. Hvis du spoler frem til næste dag, ræser lysstyringen med frem, så alt passer sammen.

Vil du have, at vi justerer farven på lyset (f.eks. så nogle huse har et mere koldt/blåt tv-lys eller neonlys i butiksvinduer), eller passer den varme glødepære-farve perfekt til din modelby?

## jeg vil bruge en extra slave til at styre lyset

*Det giver rigtig god mening at dedikere en ekstra, separat ESP32-slave udelukkende til at styre husbelysningen. På den måde belaster du ikke skærm- eller ur-enhederne, og du kan placere denne ESP32 centralt under din modelby, hvor alle ledningerne fra husene mødes.
* Da denne enhed kun skal styre lyset, har den ikke brug for en skærm, men den skal stadig lytte til Master-uret via Home Assistant og have sin egen fysiske Start/Pause-knap, så du også kan styre køreplanen herfra, hvis du står tæt på byen.

Her er den færdige opsætning til din dedikerede Lys-Slave:

### 1. Opret den specifikke lys-pakke: fast_clock_lighting_slave.yaml

Opret denne nye fil i din ESPHome-mappe. Den kombinerer uret fra Home Assistant med WS2812B-styringen:

```yaml
# fast_clock_lighting_slave.yaml

# Konfiguration af WS2812B LED-strippen (Forbind DATA til GPIO25)
light:
  - platform: neopixelbus
    type: GRB
    variant: WS2812X
    pin: GPIO25            
    num_leds: 20          # Adjust to the number of houses/LEDs you have
    name: "Modelby Belysning"
    id: city_lights
    restore_mode: ALWAYS_OFF

# Hent tids- og statusdata fra Master-uret via Home Assistant
text_sensor:
  - platform: homeassistant
    entity_id: text_sensor.accelerated_time
    id: master_time
    on_value:
      then:
        - lambda: |-
            // Kør lysstyringen hver gang det digitale ur opdateres
            if (id(master_running_status).state) {
              int hr, mn, sc;
              if (sscanf(x.c_str(), "%d:%d:%d", &hr, &mn, &sc) == 3) {
                int current_minutes_of_day = (hr * 60) + mn;
                
                auto call = id(city_lights).turn_on();
                call.set_transition_length(2000); // 2 sekunders blød glødepære-effekt
                
                for (int i = 0; i < 20; i++) {
                  // Unikt tændingstidspunkt mellem kl. 17:00 (1020 min) og 19:00
                  int start_delay = (i * 7) % 120; 
                  int taend_tid = 1020 + start_delay;
                  
                  // Unikt slukketidspunkt mellem kl. 23:00 (1380 min) og 01:00 næste dag
                  int stop_delay = (i * 13) % 120;
                  int sluk_tid = 1380 + stop_delay;
                  
                  if (current_minutes_of_day >= taend_tid && current_minutes_of_day < sluk_tid) {
                    call.set_led(i, Color(255, 160, 40)); // Varm glødepære-farve
                  } else {
                    call.set_led(i, Color(0, 0, 0)); // Slukket
                  }
                }
                call.perform();
              }
            }

binary_sensor:
  # Fysisk knap på kontrolpulten ved byen (GPIO14)
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

  # Lytter til om uret kører eller er sat på pause
  - platform: homeassistant
    entity_id: switch.ur_status_start_pause
    id: master_running_status
```

### 2. Opret hovedfilen til din nye Lys-Slave (modelby_lys.yaml)

Nu opretter du selve enheds-filen til din nye ESP32. Koden bliver ekstremt kort, da alt ligger gemt i pakken:

```yaml
# modelby_lys.yaml
esphome:
  name: modelby-lys

esp32:
  board: esp32dev

substitutions:
  node_friendly_name: "Modelby Lysstyring" # Navnet der vises på alle skærme, hvis du trykker pause herfra!

# TRÆK LYS-PAKKEN IND HER:
packages:
  lighting_slave: !include fast_clock_lighting_slave.yaml

# Netværkskonfiguration
wifi:
  ssid: "DitWiFi"
  password: "DitPassword"

api:
logger:
```

### Fordele ved denne opsætning:

1. Ingen skærm påkrævet: Denne ESP32 passer sig selv i baggrunden under anlægget.
2. Global synkronisering: Lys-Slaven tænder ikke lyset ud fra sit eget interne ur, men lytter direkte efter Master-uret i Home Assistant. Hvis du sætter uret på pause fra en helt anden station, fryser lyset øjeblikkeligt i Modelbyen også.
3. Fuldt integreret: Knappen på denne enhed (GPIO14) fungerer nøjagtigt ligesom på de andre slaver. Hvis du trykker på den, pauser alt lys og alle ure på hele banen, og der vil stå || PAUSE: Modelby Lysstyring på samtlige stationsdisplays.

Nu har du et fuldstændig professionelt og opdelt netværk af ESP32-enheder til din bane! Er der andre by-effekter, du kunne tænke dig at koble på denne lys-slave, nu hvor den er dedikeret til byen (f.eks. svejselys på et værksted, et blinkende TV-lys i et vindue, eller gadebelysning, der tænder på et andet tidspunkt)?
