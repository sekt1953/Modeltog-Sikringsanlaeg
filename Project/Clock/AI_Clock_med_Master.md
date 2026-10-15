# AI clock med master

## nu mangler jeg bare at have 10 stk esp32 med kode som kan pause/forsætte tiden vise i alle display hvor der er trykket pause

* For at få 10 uafhængige ESP32-enheder til at tale sammen, synkronisere start/pause og vise præcis, hvilken enhed der har trykket på pause, skal vi bruge Home Assistant som den centrale "dirigent".
* I stedet for at hver ESP32 kører sit eget uafhængige ur, lader vi én primær ESP32 (Master) styre selve ur-logikken og H-broen. De 9 andre ESP32-enheder (Slaves) lytter blot på tiden og status fra Home Assistant. Hvis en person trykker på en fysisk knap på en vilkårlig af de 10 enheder, sendes beskeden til Home Assistant, som pauser uret globalt og sender navnet på enheden ud til alle displays.
* Her er opsætningen opdelt i to pakker, som du gemmer i din ESPHome-mappe.

1. Opret pakken til din MASTER (Hoved-uret med H-bro)

Gem denne som fast_clock_master.yaml. Denne ESP32 styrer tiden og dit gamle mekaniske ur.

```yaml
# fast_clock_master.yaml

i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

output:
  - platform: gpio
    pin: GPIO16
    id: h_bridge_in1
  - platform: gpio
    pin: GPIO17
    id: h_bridge_in2

# Modtag information fra Home Assistant om, hvem der pausede uret
text_sensor:
  - platform: homeassistant
    name: "Hvem pausede uret"
    entity_id: input_text.clock_paused_by
    id: paused_by_who

  - platform: template
    name: "Accelerated Time"
    id: accelerated_time
    icon: "mdi:clock-fast"
    update_interval: 1s
    lambda: |-
      char formatted_time[10];
      time_t raw_time = id(simulated_timestamp);
      struct tm *time_info = gmtime(&raw_time);
      strftime(formatted_time, sizeof(formatted_time), "%H:%M:%S", time_info);
      return std::string(formatted_time);
    on_value:
      then:
        - component.update: oled_display

  - platform: template
    name: "Accelerated Date"
    id: accelerated_date
    icon: "mdi:calendar-clock"
    update_interval: 1s
    lambda: |-
      char formatted_date[15];
      time_t raw_time = id(simulated_timestamp);
      struct tm *time_info = gmtime(&raw_time);
      strftime(formatted_date, sizeof(formatted_date), "%Y-%m-%d", time_info);
      return std::string(formatted_date);

# Fysisk knap på Masteren (Kalder Home Assistant i stedet for intern logik)
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO14
      mode: INPUT_PULLUP
    name: "Fysisk Betjeningsknap"
    on_click:
      - min_length: 50ms
        max_length: 1000ms
        then:
          - homeassistant.service:
              service: script.clock_toggle_global
              data:
                device_name: "Master Ur"
      - min_length: 3000ms
        max_length: 10000ms
        then:
          - button.press: reset_clock_button

switch:
  - platform: template
    name: "Ur Status (Start/Pause)"
    id: clock_running
    icon: "mdi:play-pause"
    optimistic: true
    assumed_state: false
    initial_state: true
    on_turn_on:
      - component.update: oled_display
    on_turn_off:
      - component.update: oled_display

button:
  - platform: template
    name: "Nulstil Ur"
    id: reset_clock_button
    icon: "mdi:restart"
    on_press:
      then:
        - lambda: |-
            id(simulated_timestamp) = 1792026900;
            id(last_triggered_minute) = 55;
            id(is_fast_forwarding) = false;
        - component.update: oled_display

  - platform: template
    name: "Spol frem til næste dag 03:55"
    id: fast_forward_button
    icon: "mdi:fast-forward"
    on_press:
      then:
        - lambda: |-
            if (!id(is_fast_forwarding)) {
              id(saved_multiplier) = id(speed_multiplier).state;
              id(speed_multiplier).make_variable_set(60.0);
              id(is_fast_forwarding) = true;
              id(clock_running).publish_state(true);
            }

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 120.0
    step: 0.5
    initial_value: 7.0

display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    lambda: |-
      it.line(0, 15, 127, 15);
      if (id(is_fast_forwarding)) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, ">> SPOLER FREM >>");
      } else if (id(clock_running).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        std::string pause_msg = "|| PAUSE: " + id(paused_by_who).state;
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, pause_msg.c_str());
      }
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 9
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

globals:
  - id: simulated_timestamp
    type: int
    initial_value: '1792026900'
  - id: last_triggered_minute
    type: int
    initial_value: '55'
  - id: polarity_flip
    type: bool
    initial_value: 'false'
  - id: is_fast_forwarding
    type: bool
    initial_value: 'false'
  - id: saved_multiplier
    type: float
    initial_value: '7.0'

time:
  - platform: homeassistant
    id: homeassistant_time

interval:
  - interval: 1s
    then:
      - lambda: |-
          if (id(clock_running).state) {
            id(simulated_timestamp) += (int)id(speed_multiplier).state;
            time_t raw_time = id(simulated_timestamp);
            struct tm *time_info = gmtime(&raw_time);
            int current_minute = time_info->tm_min;
            int current_hour = time_info->tm_hour;
            
            if (id(is_fast_forwarding) && current_hour == 3 && current_minute == 55) {
              id(is_fast_forwarding) = false;
              id(speed_multiplier).make_variable_set(id(saved_multiplier));
            }
            
            if (current_minute != id(last_triggered_minute)) {
              id(last_triggered_minute) = current_minute;
              id(polarity_flip) = !id(polarity_flip);
              if (id(polarity_flip)) {
                id(h_bridge_in1).turn_on();
                delay(300);
                id(h_bridge_in1).turn_off();
              } else {
                id(h_bridge_in2).turn_on();
                delay(300);
                id(h_bridge_in2).turn_off();
              }
            }
          }
```

### 2. Opret pakken til de 9 SLAVES (Ekstra displays rundt på banen)

Gem denne som fast_clock_slave.yaml. Denne kode skal ligge på de 9 andre ESP32'ere. De importerer tidsdata og pause-status direkte fra Masteren via Home Assistant.

```yaml
# fast_clock_slave.yaml

i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Hent data fra Master-uret via Home Assistant entiteter
text_sensor:
  - platform: homeassistant
    name: "Master Time"
    entity_id: text_sensor.accelerated_time
    id: master_time
    on_value:
      then:
        - component.update: oled_display

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
        # Sender slavenavnet (defineret i enhedens hovedfil via substitutions) til scriptet i Home Assistant
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
        - component.update: oled_display

display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    lambda: |-
      it.line(0, 15, 127, 15);
      
      if (id(master_running_status).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        std::string pause_msg = "|| PAUSE: " + id(paused_by_who).state;
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, pause_msg.c_str());
      }
      
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(master_time).state.c_str());
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(master_date).state.c_str());

font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 9
  - file: "gfonts://Roboto"
    id: font_large
    size: 24
```

Når du opretter en slave-fil (f.eks. station_nord.yaml), inkluderer du den blot med et unikt navn via substitutions:

```yaml
substitutions:
  node_friendly_name: "Station Nord" # Navnet der vises på alle skærme ved pause!

packages:
  slave_clock: !include fast_clock_slave.yaml
```

### 3. Opsætning i Home Assistant (Hjernen der forbinder dem)

For at enhederne kan dele teksten om, hvem der har trykket på pause, skal du oprette en virtuel tekst-hjælper og et script i Home Assistant.
Tilføj dette til din configuration.yaml i Home Assistant (eller opret det direkte via den visuelle brugerflade under Hjælpere/Automatiseringer):

```yaml
# configuration.yaml

# 1. Opret tekstfeltet til at huske navnet
input_text:
  clock_paused_by:
    name: "Uret er sat på pause af"
    initial: "Ingen"

# 2. Opret scriptet der modtager trykket, ændrer teksten og pauser Master-uret
script:
  clock_toggle_global:
    argument:
      device_name: string
    sequence:
      - choose:
          # Hvis uret kører -> Sæt på pause og gem navnet på trykkeren
          - conditions:
              - condition: state
                entity_id: switch.ur_status_start_pause
                state: "on"
            sequence:
              - service: input_text.set_value
                target:
                  entity_id: input_text.clock_paused_by
                data:
                  value: "{{ device_name }}"
              - service: switch.turn_off
                target:
                  entity_id: switch.ur_status_start_pause
          # Hvis uret i forvejen er sat på pause -> Start uret igen
          - conditions:
              - condition: state
                entity_id: switch.ur_status_start_pause
                state: "off"
            sequence:
              - service: switch.turn_on
                target:
                  entity_id: switch.ur_status_start_pause
```

### Hvordan det virker under spillet:

1. Operatøren ved "Station Nord" (en slave) kan ikke følge med og trykker på sin fysiske knap (GPIO14).
2. Slaven sender øjeblikkeligt en besked til Home Assistant-scriptet: "Jeg er Station Nord, skift ur-status!".
3. Home Assistant ændrer input_text.clock_paused_by til "Station Nord" og slukker for switch.ur_status_start_pause på Masteren.
4. Master-uret fryser øjeblikkeligt tiden og stopper H-broen.
5. Inden for få millisekunder opdateres alle 10 OLED-displays på modelbanen, så der øverst står: || PAUSE: Station Nord.
6. Når problemet er løst, kan en vilkårlig operatør (eller Masteren) trykke på sin knap igen for at genoptage køreplanen for alle mand!

Har du brug for hjælp til at strukturere de enkelte 10 enheds-YAML-filer (hovedfilerne), så du er helt sikker på, hvordan du navngiver dem forskelligt ved hjælp af substitutions?

## jeg vil gerne at det kun er den der har stoppet uret og master som kan starte det igen

* For at sikre, at det kun er den station, der har trykket på pause (eller Masteren), som kan genstarte uret, skal vi flytte logikken ind i Home Assistant-scriptet.
* Scriptet skal tjekke: Hvis uret er sat på pause, matcher navnet på den, der trykker nu, så navnet på den, der trykkede på pause? (Eller er det Masteren?) Hvis ja, starter uret. Hvis nej, ignoreres trykket, så andre stationer ikke kan blande sig.

Her er de nødvendige opdateringer:

### 1. Opdater dit Home Assistant Script

Erstat dit tidligere script i din Home Assistant configuration.yaml (eller i din Script-editor i brugerfladen) med dette opdaterede script. Det indeholder nu den strikse adgangskontrol:

```yaml
script:
  clock_toggle_global:
    fields:
      device_name:
        description: "Navnet på den enhed der trykker"
        example: "Station Nord"
    sequence:
      - choose:
          # SITUATION A: Uret KØRER -> Alle må sætte det på pause
          - conditions:
              - condition: state
                entity_id: switch.ur_status_start_pause
                state: "on"
            sequence:
              - service: input_text.set_value
                target:
                  entity_id: input_text.clock_paused_by
                data:
                  value: "{{ device_name }}"
              - service: switch.turn_off
                target:
                  entity_id: switch.ur_status_start_pause
              - logger.log: "Uret blev sat på pause af {{ device_name }}"

          # SITUATION B: Uret er PAUSET -> KUN den der pausede (eller Master Ur) må starte det!
          - conditions:
              - condition: state
                entity_id: switch.ur_status_start_pause
                state: "off"
              - condition: template
                value_template: "{{ device_name == states('input_text.clock_paused_by') or device_name == 'Master Ur' }}"
            sequence:
              - service: switch.turn_on
                target:
                  entity_id: switch.ur_status_start_pause
              - service: input_text.set_value
                target:
                  entity_id: input_text.clock_paused_by
                data:
                  value: "Ingen"
```

### 2. Tilpasning i dine ESPHome-filer (Ingen kodeændring, kun vigtig info)

Du behøver ikke ændre strukturen i de pakke-filer (fast_clock_master.yaml og fast_clock_slave.yaml), vi lavede før. Du skal blot sikre dig, at navnene matcher præcis i dine enheders individuelle hoved-YAML-filer.
Når du laver dine slave-filer, skal du definere deres unikke navn i toppen via substitutions. Det er dette navn, Home Assistant låser uret med:
Eksempel på Slave 1 (station_nord.yaml):

```yaml
esphome:
  name: station-nord

substitutions:
  node_friendly_name: "Station Nord" # <--- Dette navn sendes til Home Assistant

packages:
  slave_clock: !include fast_clock_slave.yaml
```

Eksempel på Slave 2 (remisen.yaml):

```yaml
esphome:
  name: remisen

substitutions:
  node_friendly_name: "Remisen" # <--- Dette navn sendes til Home Assistant

packages:
  slave_clock: !include fast_clock_slave.yaml
```

(I master-pakken har vi allerede hårdkodet, at den sender navnet "Master Ur". Da scriptet tillader "Master Ur", vil master-knappen altid kunne overstyre og starte tiden igen, uanset hvem der pausede).

### Hvordan det virker nu under kørslen:

1. "Station Nord" trykker på sin fysiske knap. Uret pauser. Alle skærme viser: || PAUSE: Station Nord.
2. "Remisen" trykker på sin knap for at starte uret igen, fordi de er klar. Intet sker! Scriptet afviser trykket, da "Remisen" ikke ejer pausen.
3. Først når "Station Nord" trykker på sin egen knap (eller du trykker på Master Ur / kontrolpulten), accepterer Home Assistant signalet, starter uret igen, og fjerner pause-teksten på alle 10 skærme.

Nu er adgangsstyringen helt på plads! Skal vi også tilføje en lille visuel indikation på skærmen (f.eks. et blinkende symbol eller et udråbtegn), hvis man trykker på knappen og bliver afvist, eller er logikken klar til at blive testet af på dit anlæg?

## vi må ikke kunne nulstille uret, men kun kunne sætte det frem til næste dag kl 03:55, da det tager tid at kører urene frem

* Det giver rigtig god mening! Hvis man ved en fejl trykker på den hårde nulstillingsknap, vil det digitale ur springe øjeblikkeligt i tid, mens jeres gamle mekaniske ure vil tabe synkroniseringen fuldstændigt, fordi de fysisk skal nå at klikke hele vejen rundt.
* For at forhindre denne fejl har jeg fjernet den hårde nulstillingsfunktion fuldstændigt – både fra den fysiske betjeningsknap (det lange tryk) og fra Home Assistant. Nu er den eneste måde at nulstille eller gøre klar til næste dags køreplan på, at bruge "Spol frem" (Fast Forward)-funktionen, som klikker de mekaniske ure sikkert og hurtigt frem minut for minut.

Her er den opdaterede og sikrede fast_clock_master.yaml pakke-fil:

Opdateret fast_clock_master.yaml (Uden hård nulstilling)

```yaml
# fast_clock_master.yaml

# Aktiver I2C-bussen til skærmen på din ESP32 HW-394
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# De to GPIO-ben til IN1 og IN2 på L298N H-broen
output:
  - platform: gpio
    pin: GPIO16
    id: h_bridge_in1
  - platform: gpio
    pin: GPIO17
    id: h_bridge_in2

# Fysisk knap på Masteren (Kan KUN starte/pause, ikke nulstille)
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO14
      mode: INPUT_PULLUP
    name: "Fysisk Betjeningsknap"
    on_click:
      - min_length: 50ms
        max_length: 1000ms
        then:
          - homeassistant.service:
              service: script.clock_toggle_global
              data:
                device_name: "Master Ur"

# Modtag information fra Home Assistant om status
text_sensor:
  - platform: homeassistant
    name: "Hvem pausede uret"
    entity_id: input_text.clock_paused_by
    id: paused_by_who

  - platform: template
    name: "Accelerated Time"
    id: accelerated_time
    icon: "mdi:clock-fast"
    update_interval: 1s
    lambda: |-
      char formatted_time[30];
      time_t raw_time = id(simulated_timestamp);
      struct tm *time_info = gmtime(&raw_time);
      strftime(formatted_time, sizeof(formatted_time), "%H:%M:%S", time_info);
      return std::string(formatted_time);
    on_value:
      then:
        - component.update: oled_display

  - platform: template
    name: "Accelerated Date"
    id: accelerated_date
    icon: "mdi:calendar-clock"
    update_interval: 1s
    lambda: |-
      char formatted_date[30];
      time_t raw_time = id(simulated_timestamp);
      struct tm *time_info = gmtime(&raw_time);
      strftime(formatted_date, sizeof(formatted_date), "%Y-%m-%d", time_info);
      return std::string(formatted_date);

# Kontroller til Home Assistant (Hård nulstillings-button er FJERNET herfra)
switch:
  - platform: template
    name: "Ur Status (Start/Pause)"
    id: clock_running
    icon: "mdi:play-pause"
    optimistic: true
    assumed_state: false
    initial_state: true
    on_turn_on:
      - component.update: oled_display
    on_turn_off:
      - component.update: oled_display

button:
  - platform: template
    name: "Spol frem til næste dag 03:55"
    id: fast_forward_button
    icon: "mdi:fast-forward"
    on_press:
      then:
        - lambda: |-
            if (!id(is_fast_forwarding)) {
              // Gem den nuværende hastighed, så vi kan vende tilbage til den bagefter
              id(saved_multiplier) = id(speed_multiplier).state;
              // Sæt uret til at køre lynhurtigt fremad (f.eks. 60 sekunder pr. real-sekund)
              id(speed_multiplier).make_variable_set(60.0);
              id(is_fast_forwarding) = true;
              id(clock_running).publish_state(true); // Sørg for uret kører under spol
            }

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 120.0
    step: 0.5
    initial_value: 7.0

# Konfiguration af din SSD1306 128x64 OLED-skærm
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    lambda: |-
      it.line(0, 15, 127, 15);
      
      if (id(is_fast_forwarding)) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, ">> SPOLER FREM >>");
      } else if (id(clock_running).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        std::string pause_msg = "|| PAUSE: " + id(paused_by_who).state;
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, pause_msg.c_str());
      }
      
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 9
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

globals:
  - id: simulated_timestamp
    type: int
    initial_value: '1792026900' # Starter på 2026-10-01 03:55:00
  - id: last_triggered_minute
    type: int
    initial_value: '55'
  - id: polarity_flip
    type: bool
    initial_value: 'false'
  - id: is_fast_forwarding
    type: bool
    initial_value: 'false'
  - id: saved_multiplier
    type: float
    initial_value: '7.0'

time:
  - platform: homeassistant
    id: homeassistant_time

interval:
  - interval: 1s
    then:
      - lambda: |-
          if (id(clock_running).state) {
            id(simulated_timestamp) += (int)id(speed_multiplier).state;
            
            time_t raw_time = id(simulated_timestamp);
            struct tm *time_info = gmtime(&raw_time);
            int current_minute = time_info->tm_min;
            int current_hour = time_info->tm_hour;
            
            // Hvis vi spoler frem, og har ramt præcis 03:55 igen, så stop spoling
            if (id(is_fast_forwarding) && current_hour == 3 && current_minute == 55) {
              id(is_fast_forwarding) = false;
              id(speed_multiplier).make_variable_set(id(saved_multiplier));
            }
            
            if (current_minute != id(last_triggered_minute)) {
              id(last_triggered_minute) = current_minute;
              id(polarity_flip) = !id(polarity_flip);
              
              if (id(polarity_flip)) {
                id(h_bridge_in1).turn_on();
                delay(300);
                id(h_bridge_in1).turn_off();
              } else {
                id(h_bridge_in2).turn_on();
                delay(300);
                id(h_bridge_in2).turn_off();
              }
            }
          }
```

Det opdaterede Home Assistant Dashboard:

Du bør også fjerne den gamle nulstillingsknap fra din brugerflade i Home Assistant, så det nu ser således ud:

```yaml
type: entities
title: 🕒 Modelbane Fast Clock
entities:
  - entity: text_sensor.accelerated_time
    name: Modelbane Klokken
  - entity: text_sensor.accelerated_date
    name: Modelbane Dato
  - entity: switch.ur_status_start_pause
    name: Status (Tænd/Pause)
  - entity: number.time_speed_multiplier
    name: Tids-hastighed (Multiplier)
  - entity: button.spol_frem_til_naeste_dag_03_55
    name: ⏭️ Gør klar til næste dag (03:55)
```

Nu er systemet fuldstændig sikret mod utilsigtede tidsspring. Den eneste måde at flytte tiden på er den kontrollerede spoling, hvor de fysiske ure tikker synkront med!

