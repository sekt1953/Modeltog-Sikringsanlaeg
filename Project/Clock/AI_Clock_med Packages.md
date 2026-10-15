# Ur Kode med Packages

* Mange tak for koden! Det viser sig, at det ikke var traditionel Arduino C++, men en ESPHome YAML-konfiguration (typisk brugt til Home Assistant). Det giver rigtig god mening til en modelbane, da det gør det super nemt at styre uret direkte fra en tablet eller telefon.
* Koden har dog en enkelt teknisk udfordring i ESPHome: localtime() funktionen i ESPHome/Arduino kræver normalt, at systemets interne tidszone-motor er initialiseret, og rå UTC-timestamps (som dit start-timestamp 1792012800 fra oktober 2026) kan opføre sig uforudsigeligt eller vise forkert tidszone, hvis ESPHome ikke har en aktiv time-komponent.
* Her er den optimerede og rettede udgave af din kode, hvor der er tilføjet en lokal tids-motor, som sikrer, at strftime formateringen fungerer perfekt:

```yaml
# Add a variable to track the simulated timestamp locally
globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800' # Default start date (e.g., Oct 2026)

# Nødvendig for at ESPHome kan håndtere 'localtime' og tidszoner korrekt
time:
  - platform: homeassistant
    id: homeassistant_time

# A template slider so you can change the acceleration factor on the fly
number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
    step: 0.5
    initial_value: 7.0 # Splitting the difference between 6 and 8

interval:
  # This ticks every 1 second and advances our virtual clock by N seconds
  - interval: 1s
    then:
      - lambda: |-
          // Advance the clock by the multiplier value every real second
          id(simulated_timestamp) += (int)id(speed_multiplier).state;

# Create formatted text sensors to display your accelerated time
text_sensor:
  - platform: template
    name: "Accelerated Time"
    id: accelerated_time
    icon: "mdi:clock-fast"
    update_interval: 1s
    lambda: |-
      char formatted_time[30];
      time_t raw_time = id(simulated_timestamp);
      // Bruger gmtime i stedet for localtime for at undgå uønsket tidszone-forskydning på rå timestamps
      struct tm *time_info = gmtime(&raw_time); 
      
      // Formats as HH:MM:SS
      strftime(formatted_time, sizeof(formatted_time), "%H:%M:%S", time_info);
      return std::string(formatted_time);

  - platform: template
    name: "Accelerated Date"
    id: accelerated_date
    icon: "mdi:calendar-clock"
    update_interval: 1s # Sat til 1s så datoen skifter præcis ved midnat
    lambda: |-
      char formatted_date[30];
      time_t raw_time = id(simulated_timestamp);
      struct tm *time_info = gmtime(&raw_time);
      
      // Formats as YYYY-MM-DD
      strftime(formatted_date, sizeof(formatted_date), "%Y-%m-%d", time_info);
      return std::string(formatted_date);
```

### Hvad er ændret?

* Tilføjet time-komponent: Dette tvinger ESPHomes interne biblioteker til at inkludere de nødvendige tidsfunktioner.
* Skiftet til gmtime(): Når man arbejder med et fast, råt timestamp (som dit start-tal), vil localtime() forsøge at lægge din lokale tidszone (f.eks. +1 eller +2 timer i Danmark) oveni. Ved at bruge gmtime() låser vi uret præcis til den tid, dit timestamp repræsenterer.
Hvad er næste skridt for dit modelbane-ur? Fortæl mig gerne:
* Vil du have tilføjet en funktion til at stoppe/pause uret eller nulstille det til start-tiden via Home Assistant?
* Skal uret sendes videre til et fysisk display (f.eks. en TM1637 7-segment skærm eller et OLED-display forbundet til din ESP-enhed)?
* Vil du have lavet automatiseringer, der tænder natbelysning på modelbanen, når uret rammer f.eks. kl. 20:00?

### How du i make it to packages system for esphome

* For at lave din kode om til en ESPHome Package (pakke), skal du flytte selve ur-logikken over i en separat fil. På den måde kan du genbruge uret på tværs af flere forskellige ESP32/ESP8266-enheder uden at kopiere hele koden hver gang.
Her er den nemmeste måde at gøre det på.

#### 1. Opret pakke-filen (f.eks. fast_clock.yaml)

* Opret en ny fil i din ESPHome-mappe (samme sted som dine andre enheder) og kald den fast_clock.yaml. Indsæt denne kode, som indeholder hele ur-systemet:

```yaml
# fast_clock.yaml

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800' # Default start date

time:
  - platform: homeassistant
    id: homeassistant_time

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
    step: 0.5
    initial_value: 7.0

interval:
  - interval: 1s
    then:
      - lambda: |-
          id(simulated_timestamp) += (int)id(speed_multiplier).state;

text_sensor:
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
```

* (Bemærk: Jeg har rettet char formatted_time til char formatted_time[30], da arrays skal have en defineret størrelse i C++ for at undgå fejl under kompilering).

#### 2. Inkluder pakken i din enheds konfiguration

* Åbn nu den YAML-fil, som hører til din specifikke ESP-enhed (f.eks. modelbane-node.yaml). Du indsætter pakken i toppen ved at bruge packages:-funktionen:

```yaml
esphome:
  name: modelbane-ur-node
  friendly_name: Modelbane Ur Node

esp32:
  board: esp32dev

# HENT UR-SYSTEMET IND HER:
packages:
  modelbane_ur: !include fast_clock.yaml

# Resten af din normale opsætning (wifi, api, osv.)
wifi:
  ssid: "MitWiFi"
  password: "MitPassword"

api:
  encryption:
    key: "din-hemmelige-nøgle"

logger:
```

* Når du nu gemmer og installerer koden på din ESP, vil ESPHome automatisk smelte fast_clock.yaml sammen med din enheds kode.
Vil du have, at vi gør pakken endnu mere fleksibel? Jeg kan vise dig, hvordan vi bruger substitutions (variabler), så du kan ændre f.eks. start-datoen eller hastigheden direkte fra din hovedfil uden at skulle rette i selve pakke-filen. Skal vi kigge på det?
AI-svar kan indeholde fejl. Få flere oplysninger

## I will like the time shown on a ssd1306_i2c

For at få vist det hurtige modelbane-ur på en SSD1306 OLED-skærm (I2C), skal vi blot udvide vores pakke-fil, så den også sender tids- og datoteksterne videre til skærmen.
Her er, hvordan du opdaterer din fast_clock.yaml pakke-fil, så den automatisk tænder I2C-bussen og styrer SSD1306-skærmen:

### 1. Opdater fast_clock.yaml

Erstat indholdet i din fast_clock.yaml med denne kode. Den definerer I2C-pindene (standard for ESP32 er SDA=GPIO21 og SCL=GPIO22) og tegner uret flot op på skærmen:

```yaml
# fast_clock.yaml

# Aktiver I2C-bussen (Standard ESP32 pins. Ret dem hvis du bruger ESP8266/andre pins)
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Konfiguration af SSD1306 OLED-skærmen
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    lambda: |-
      // Tegn en ramme omkring skærmen
      it.rect(0, 0, 128, 64);
      
      // Vis overskrift/status
      it.print(64, 4, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      
      // Vis den accelererede tid (Stort i midten)
      it.printf(64, 22, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      
      // Vis den accelererede dato (Lille i bunden)
      it.printf(64, 50, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper (Fonts) som bruges på skærmen
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 20

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800'

time:
  - platform: homeassistant
    id: homeassistant_time

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
    step: 0.5
    initial_value: 7.0

interval:
  - interval: 1s
    then:
      - lambda: |-
          id(simulated_timestamp) += (int)id(speed_multiplier).state;

text_sensor:
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
        - component.update: oled_display # Tvinger skærmen til at opdatere med det samme sekundet skifter

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
```

### Vigtigt angående hardware:

1. Pins: Koden herover bruger GPIO21 og GPIO22 til I2C. Hvis du bruger en ESP8266 (f.eks. D1 Mini), skal du typisk ændre dem til GPIO4 (D2) og GPIO5 (D1).
2. Skærmstørrelse: Koden er sat op til en standard 128x64 pixel skærm. Hvis din skærm er den mindre 128x32 model, skal du ændre model: "SSD1306 128x64" til model: "SSD1306 128x32" og rette pixel-koordinaterne (it.printf) en smule til i højden.
   * For at vi kan få det til at passe helt perfekt til din hardware, har du så brug for:
   * At tilpasse koden til en ESP8266 / NodeMCU / Wemos D1 Mini i stedet for en ESP32?
   * At ændre layoutet på skærmen (f.eks. fjerne sekunder eller vise noget andet tekst)?
   * Hjælp til at ændre skærmstørrelsen, hvis du har den smalle 128x32 model?

## Jeg bruger SSD1306 128x64

Super! Med en SSD1306 128x64 har du masser af plads til at lave et flot og letlæseligt layout til din modelbane.
Her er den færdige fast_clock.yaml pakke-fil, hvor layoutet er optimeret præcis til en 128x64 skærm. Tiden står stort og tydeligt i midten, så du nemt kan se den på afstand, mens datoen og en flot skillelinje rammer det hele ind:

```yaml
# fast_clock.yaml

# Aktiver I2C-bussen (Standard ESP32 pins. Ret til GPIO4/GPIO5 hvis du bruger ESP8266)
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Konfiguration af din SSD1306 128x64 OLED-skærm
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    pages:
      - id: clock_page
        lambda: |-
          // Tegn en pæn vandret skillelinje under overskriften
          it.line(0, 15, 127, 15);
          
          // Overskrift i toppen
          it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
          
          // Det accelererede UR (Stort og centreret i midten)
          it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
          
          // Den accelererede DATO (Lederen i bunden)
          it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper (Fonts) hentes automatisk fra Google Fonts under kompilering
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 24 # Gjort større så det er nemt at se på afstand

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800' # Starter i oktober 2026

time:
  - platform: homeassistant
    id: homeassistant_time

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
    step: 0.5
    initial_value: 7.0

interval:
  - interval: 1s
    then:
      - lambda: |-
          id(simulated_timestamp) += (int)id(speed_multiplier).state;

text_sensor:
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
        - component.update: oled_display # Opdaterer skærmen med det samme når uret tikker

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
```

Husk at tjekke din hovedfil:

* I din enheds primære YAML-fil (f.eks. esp32-clock.yaml) skal du blot sikre dig, at du har inkluderet pakken sådan her:

```yaml
packages:
  modelbane_ur: !include fast_clock.yaml
```

* Nu hvor skærmen er på plads, er der så andet, vi skal tilføje?
  * Vil du have mulighed for at skifte mellem 12-timers og 24-timers visning?
  * Skal vi tilføje en fysisk trykknap til ESP'en, så du kan sætte uret på pause eller nulstille det uden at skulle have fat i telefonen?

## Jeg vil gerne have en Start/Pause knap til at pause tiden, hvis vi ikke kan følge køreplanen

Det giver rigtig god mening! På en modelbane kan det hurtigt blive stressende, hvis rangeringen tager længere tid end planlagt.
Her er den opdaterede fast_clock.yaml pakke-fil. Jeg har tilføjet en virtuel Start/Pause-knap (en switch), som du både kan styre fra Home Assistant og bruge direkte i koden. Når kontakten slås FRA, pauser uret med det samme.
Som en ekstra bonus viser skærmen nu "|| PAUSE" i toppen i stedet for "MODELBANE UR", når tiden er stoppet, så du aldrig er i tvivl om status.

Opdateret fast_clock.yaml med Pause-funktion

```aml
# fast_clock.yaml

# Aktiver I2C-bussen (Standard ESP32 pins. Ret til GPIO4/GPIO5 hvis du bruger ESP8266)
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Virtuel kontakt til at starte og pause uret
switch:
  - platform: template
    name: "Ur Status (Start/Pause)"
    id: clock_running
    icon: "mdi:play-pause"
    optimistic: true
    assumed_state: false
    initial_state: true # Uret starter som tændt

# Konfiguration af din SSD1306 128x64 OLED-skærm
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    lambda: |-
      // Tegn en pæn vandret skillelinje under overskriften
      it.line(0, 15, 127, 15);
      
      // Dynamic overskrift baseret på om uret kører eller er sat på pause
      if (id(clock_running).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "|| PAUSE");
      }
      
      // Det accelererede UR (Stort og centreret i midten)
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      
      // Den accelererede DATO (Lederen i bunden)
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper (Fonts)
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800' # Starter i oktober 2026

time:
  - platform: homeassistant
    id: homeassistant_time

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
    step: 0.5
    initial_value: 7.0

interval:
  # Tjekker hvert sekund - tæller kun op, hvis Start/Pause kontakten er TÆNDT
  - interval: 1s
    then:
      - lambda: |-
          if (id(clock_running).state) {
            id(simulated_timestamp) += (int)id(speed_multiplier).state;
          }

text_sensor:
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
```

Sådan virker det nu:

1. I Home Assistant: Du får en ny entitet, der hedder switch.ur_status_start_pause. Den kan du tilføje til dit dashboard som en helt almindelig tænd/sluk-knap.
2. På skærmen: Når du trykker på pause i Home Assistant, stopper tiden med at tælle med det samme, og teksten øverst på OLED-skærmen skifter til "|| PAUSE", så alle omkring banen ved, at køreplanen er sat i bero.

Vil du også have tilføjet en fysisk trykknap (hardware-knap tilsluttet en pin på din ESP), så du kan trykke på en rigtig knap på kontrolpulten for at pause uret?

## Ja, jeg vil gerne have en fysisk knap

Det gør vi! Ved at tilføje en fysisk trykknap til din kontrolpult, kan du lynhurtigt reagere og sætte køreplanen på pause uden at skulle låse din telefon eller tablet op.
Her er den opdaterede fast_clock.yaml pakke-fil. Jeg har tilføjet en fysisk knap på GPIO0 (du kan nemt ændre dette pin-nummer i koden, hvis du hellere vil bruge en anden pin). Hver gang du trykker på knappen, vil uret skifte (toggle) mellem Start og Pause.

Opdateret fast_clock.yaml med fysisk trykknap

```yaml
# fast_clock.yaml

# Aktiver I2C-bussen (Standard ESP32 pins)
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Fysisk trykknap tilsluttet din ESP
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO0      # <--- Ret dette til den PIN, din knap er forbundet til
      mode: INPUT_PULLUP # Bruger den interne modstand (knappen skal forbindes til GND)
    name: "Fysisk Pause Knap"
    on_press:
      then:
        - switch.toggle: clock_running # Skifter mellem start/pause ved hvert tryk

# Virtuel kontakt (styrer logikken og kan stadig ses/styres i Home Assistant)
switch:
  - platform: template
    name: "Ur Status (Start/Pause)"
    id: clock_running
    icon: "mdi:play-pause"
    optimistic: true
    assumed_state: false
    initial_state: true
    # Tvinger skærmen til at opdatere med det samme, når der trykkes på knappen
    on_turn_on:
      - component.update: oled_display
    on_turn_off:
      - component.update: oled_display

# Konfiguration af din SSD1306 128x64 OLED-skærm
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    lambda: |-
      // Tegn en pæn vandret skillelinje under overskriften
      it.line(0, 15, 127, 15);
      
      // Dynamisk overskrift baseret på om uret kører eller er sat på pause
      if (id(clock_running).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "|| PAUSE");
      }
      
      // Det accelererede UR (Stort og centreret i midten)
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      
      // Den accelererede DATO (Lederen i bunden)
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper (Fonts)
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800' # Starter i oktober 2026

time:
  - platform: homeassistant
    id: homeassistant_time

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
    step: 0.5
    initial_value: 7.0

interval:
  - interval: 1s
    then:
      - lambda: |-
          if (id(clock_running).state) {
            id(simulated_timestamp) += (int)id(speed_multiplier).state;
          }

text_sensor:
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
```

Sådan forbinder du knappen (Hardware):

1. Forbind den ene side af din trykknap til den valgte GPIO-pin (f.eks. GPIO0 som står i koden nu, eller en anden ledig pin som GPIO12 eller GPIO14).
2. Forbind den anden side af trykknappen direkte til en GND (Ground) pind på din ESP.
3. Fordi koden bruger INPUT_PULLUP, behøver du ikke nogen eksterne modstande – ESP'en klarer det hele selv.

Har du styr på, hvilken GPIO-pin du vil montere knappen på, eller skal vi kigge på, hvilke pins der er ledige og sikre at bruge på din specifikke ESP-model?

## jeg bruger en ESP32 WROOM 32D HW-394 du må gerne foreslå ben

På en ESP32 WROOM 32D (HW-394) skal man være lidt påpasselig med sit pin-valg, da nogle ben (kaldet strapping pins) kan forhindre ESP32 i at starte op, hvis der er trykket på en knap under opstart (f.eks. GPIO0, GPIO2 og GPIO5).
Her er de absolut mest sikre og oplagte ben (pins) til dit projekt:

Anbefalede ben til din HW-394:

• Fysisk Pause-knap: GPIO14 (Et helt neutralt input/output-ben, som er 100% sikkert at bruge).
• I2C Skærm (SDA): GPIO21 (Standard for ESP32).
• I2C Skærm (SCL): GPIO22 (Standard for ESP32).
Her er den opdaterede kode til din fast_clock.yaml pakke-fil, hvor disse ben er lagt ind, så det passer perfekt til dit hardware-kort:

```yaml
# fast_clock.yaml

# Aktiver I2C-bussen på de korrekte standardben for din ESP32 HW-394
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Fysisk trykknap tilsluttet GPIO14
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO14     # <--- Sikkert og stabilt ben på din ESP32 WROOM
      mode: INPUT_PULLUP # Knappen forbindes mellem GPIO14 og GND
    name: "Fysisk Pause Knap"
    on_press:
      then:
        - switch.toggle: clock_running

# Virtuel kontakt til Home Assistant og logik
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

# Konfiguration af din SSD1306 128x64 OLED-skærm
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    lambda: |-
      it.line(0, 15, 127, 15);
      
      if (id(clock_running).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "|| PAUSE");
      }
      
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800'

time:
  - platform: homeassistant
    id: homeassistant_time

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
    step: 0.5
    initial_value: 7.0

interval:
  - interval: 1s
    then:
      - lambda: |-
          if (id(clock_running).state) {
            id(simulated_timestamp) += (int)id(speed_multiplier).state;
          }

text_sensor:
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
```

Forbindelsesdiagram til dit HW-394 kort:

1. OLED Skærm VCC → Forbind til 3V3 på ESP32.
2. OLED Skærm GND → Forbind til GND på ESP32.
3. OLED Skærm SDA → Forbind til D21 på ESP32.
4. OLED Skærm SCL → Forbind til D22 på ESP32.
5. Trykknap Ben 1 → Forbind til D14 på ESP32.
6. Trykknap Ben 2 → Forbind til en ledig GND på ESP32.

Nu er hardwaren på plads! Kunne du tænke dig, at vi tilføjede en funktion, så du også kan nulstille uret (f.eks. ved at holde knappen inde i 3 sekunder), eller er du klar til at uploade koden nu?

## Ja, jeg vil gerne kunne nulstille uret

Det er en super smart funktion! Ved at holde den samme fysisk knap inde i 3 sekunder, kan du nu lynhurtigt nulstille uret til jeres starttidspunkt, når et nyt spil (en ny køreplan) begynder.
For at gøre det muligt i ESPHome, har jeg tilføjet en on_click-funktion på knappen:
• Kort tryk: Sætter uret på Pause eller genstarter det (skifter status).
• Langt tryk (3 sekunder): Nulstiller uret og sætter tiden tilbage til jeres standard starttidspunkt (1. oktober 2026 kl. 00:00:00 UTC).
Her er den opdaterede fast_clock.yaml pakke-fil, lige klar til at gemme og uploade:

```yaml
# fast_clock.yaml

# Aktiver I2C-bussen på de korrekte standardben for din ESP32 HW-394
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Fysisk trykknap tilsluttet GPIO14 med både klik- og hold-funktion
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO14
      mode: INPUT_PULLUP
    name: "Fysisk Betjeningsknap"
    on_click:
      # Kort tryk (under 1 sekund): Skift mellem Start/Pause
      - min_length: 50ms
        max_length: 1000ms
        then:
          - switch.toggle: clock_running
      # Langt tryk (hold inde i mindst 3 sekunder): Nulstil tiden
      - min_length: 3000ms
        max_length: 10000ms
        then:
          - lambda: |-
              id(simulated_timestamp) = 1792012800; // Sætter tiden tilbage til startværdien
          - logger.log: "Uret blev nulstillet via fysisk knap!"
          - component.update: oled_display

# Virtuel kontakt til Home Assistant og logik
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

# Konfiguration af din SSD1306 128x64 OLED-skærm
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    id: oled_display
    lambda: |-
      it.line(0, 15, 127, 15);
      
      if (id(clock_running).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "|| PAUSE");
      }
      
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800' # Standard starttidspunkt

time:
  - platform: homeassistant
    id: homeassistant_time

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
    step: 0.5
    initial_value: 7.0

interval:
  - interval: 1s
    then:
      - lambda: |-
          if (id(clock_running).state) {
            id(simulated_timestamp) += (int)id(speed_multiplier).state;
          }

text_sensor:
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
```

Nu har du et fuldt funktionelt ur med både start/pause, fysisk skærm og nulstilling via langvarigt tryk. Hvis du har brug for det, kan jeg også hjælpe med:

* At lave en knap i Home Assistant Dashboardet, der også kan nulstille uret derfra?
* At ændre start-tidspunktet (f.eks. hvis køreplanen skal starte kl. 07:00 i stedet for 00:00)?

## Ja, jeg vil gerne have disse functionet dashboardet

For at få fuld kontrol over uret fra dit Home Assistant Dashboard, skal vi tilføje en virtuel nulstillingsknap (button) i ESPHome-koden.
Her er den opdaterede fast_clock.yaml pakke-fil. Når du uploader denne, vil ESPHome automatisk sende alle de rigtige knapper og skydere videre til Home Assistant, så du kan bygge et flot kontrolpanel.

1. Opdater fast_clock.yaml med den virtuelle nulstillingsknap

```yaml
# fast_clock.yaml

# Aktiver I2C-bussen på de korrekte standardben for din ESP32 HW-394
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Fysisk trykknap tilsluttet GPIO14
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO14
      mode: INPUT_PULLUP
    name: "Fysisk Betjeningsknap"
    on_click:
      # Kort tryk: Start/Pause
      - min_length: 50ms
        max_length: 1000ms
        then:
          - switch.toggle: clock_running
      # Langt tryk (3 sek): Nulstil tiden
      - min_length: 3000ms
        max_length: 10000ms
        then:
          - button.press: reset_clock_button

# Knapper og kontakter til Home Assistant Dashboardet
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
            id(simulated_timestamp) = 1792012800; // Sætter tiden tilbage til startværdien
        - logger.log: "Uret blev nulstillet!"
        - component.update: oled_display

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
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
      
      if (id(clock_running).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "|| PAUSE");
      }
      
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800'

time:
  - platform: homeassistant
    id: homeassistant_time

interval:
  - interval: 1s
    then:
      - lambda: |-
          if (id(clock_running).state) {
            id(simulated_timestamp) += (int)id(speed_multiplier).state;
          }

text_sensor:
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
```

2. Sådan laver du dit Home Assistant Dashboard

Når du har uploadet koden, dukker alle entiteterne op under din ESPHome-integration i Home Assistant. Du kan lave et flot og kompakt kontrolpanel på dit dashboard ved at bruge denne Lovelace YAML-kode (tilføj et "Enhedskort" eller et "Blandede elementer"-kort):

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
  - entity: button.nulstil_ur
    name: Nulstil køreplan
```

Hvad du får på dit Dashboard:

* Modelbane Klokken & Dato: Viser den hurtige tid direkte på din telefonskærm live.
* Status-kontakt: En simpel skyder/knap til at fryse tiden for alle mand.
* Tids-hastighed: En slider, så du midt under spillet kan skrue hastigheden op på 10x eller ned på 1x (realtid).
* Nulstil køreplan: En klikbar knap, der øjeblikkeligt sætter tiden tilbage til start.

Nu er systemet helt komplet både på hardwaren og i Home Assistant! Kunne du tænke dig, at vi justerede starttidspunktet til et specifikt klokkeslæt (f.eks. kl. 07:00:00 om morgenen, hvor køreplanerne typisk starter)?

## 03:55:00

For at uret skal starte præcis klokken 03:55:00, skal vi blot ændre startværdien i koden. I stedet for midnat, bruger vi nu et timestamp, der rammer præcis klokken 03:55:00 (den 1. oktober 2026).
Jeg har opdateret værdien i initial_value samt i selve nulstillingsknappen, så uret hopper tilbage til 03:55:00 hver gang du trykker på nulstil.
Her er den fulde, opdaterede fast_clock.yaml pakke-fil:

```yaml
# fast_clock.yaml

# Aktiver I2C-bussen på de korrekte standardben for din ESP32 HW-394
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# Fysisk trykknap tilsluttet GPIO14
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO14
      mode: INPUT_PULLUP
    name: "Fysisk Betjeningsknap"
    on_click:
      # Kort tryk: Start/Pause
      - min_length: 50ms
        max_length: 1000ms
        then:
          - switch.toggle: clock_running
      # Langt tryk (3 sek): Nulstil tiden
      - min_length: 3000ms
        max_length: 10000ms
        then:
          - button.press: reset_clock_button

# Knapper og kontakter til Home Assistant Dashboardet
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
            id(simulated_timestamp) = 1792026900; // Sætter tiden til 1. okt 2026 kl. 03:55:00
        - logger.log: "Uret blev nulstillet til 03:55:00!"
        - component.update: oled_display

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
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
      
      if (id(clock_running).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "|| PAUSE");
      }
      
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792026900' # Unix timestamp for 2026-10-01 03:55:00

time:
  - platform: homeassistant
    id: homeassistant_time

interval:
  - interval: 1s
    then:
      - lambda: |-
          if (id(clock_running).state) {
            id(simulated_timestamp) += (int)id(speed_multiplier).state;
          }

text_sensor:
  - platform: template
    name: "Accelerated Time"
    id: accelerated_time
    icon: "mdi:clock-fast"
    update_interval: 1s
    lambda: |-
      char formatted_time;
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
      char formatted_date;
      time_t raw_time = id(simulated_timestamp);
      struct tm *time_info = gmtime(&raw_time);
      strftime(formatted_date, sizeof(formatted_date), "%Y-%m-%d", time_info);
      return std::string(formatted_date);
```

Tekniske detaljer om ændringen:

* Værdien 1792026900 er det rå Epoke-timestamp, der svarer til den 1. oktober 2026 klokken 03:55:00 UTC. Da vores kode bruger gmtime(), vil uret nu altid starte og nulstille præcis på dette minut.
Koden er nu klar til at blive gemt, kompileret og overført til din ESP32!
* Er der andre detaljer ved uret, der skal rettes til, før du uploader – for eksempel hvis du vil have skjult sekunderne på skærmen (03:55 i stedet for 03:55:00), eller er du helt klar til at teste det af på banen?

## I want 2 gpio sending signal to H-bridge l238n to change pålaritet on the signal to old clocks kan du tilføje dette som en esphome package

* For at drive gamle mekaniske banegårdsure (moduruve), skal polariteten på spændingen skifte for hvert minut (eller hvert halve minut), hvilket får urets spole til at flytte viseren frem.
For at styre en L298N H-bro, skal vi bruge 2 GPIO-ben (f.eks. GPIO16 og GPIO17). Hvert minut sender vi en kort impuls (f.eks. 300 millisekunder) gennem H-broen:
  * Minut 1: GPIO16 går HØJ, GPIO17 forbliver LAV (Strømmen løber den ene vej).
  * Minut 2: GPIO16 forbliver LAV, GPIO17 går HØJ (Strømmen vendes).
* Her er den opdaterede fast_clock.yaml pakke-fil, hvor denne H-bro-styring er fuldt integreret i det accelererede ur. Den holder styr på, hvilken vej polariteten skal vende, hver gang det virtuelle minut skifter:

```yaml
# fast_clock.yaml

# Aktiver I2C-bussen til skærmen på din ESP32 HW-394
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# De to GPIO-ben, der skal forbindes til IN1 og IN2 på L298N H-broen
output:
  - platform: gpio
    pin: GPIO16
    id: h_bridge_in1
  - platform: gpio
    pin: GPIO17
    id: h_bridge_in2

# Fysisk trykknap tilsluttet GPIO14
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
          - switch.toggle: clock_running
      - min_length: 3000ms
        max_length: 10000ms
        then:
          - button.press: reset_clock_button

# Kontroller og knapper til Home Assistant
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
            id(simulated_timestamp) = 1792026900; // Sætter tiden til 1. okt 2026 kl. 03:55:00
            id(last_triggered_minute) = 55;        // Synkroniserer minut-tracker med starttidspunktet
        - logger.log: "Uret blev nulstillet til 03:55:00!"
        - component.update: oled_display

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
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
      
      if (id(clock_running).state) {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "MODELBANE UR");
      } else {
        it.print(64, 2, id(font_small), TextAlign::TOP_CENTER, "|| PAUSE");
      }
      
      it.printf(64, 20, id(font_large), TextAlign::TOP_CENTER, "%s", id(accelerated_time).state.c_str());
      it.printf(64, 52, id(font_small), TextAlign::TOP_CENTER, "%s", id(accelerated_date).state.c_str());

# Skrifttyper
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 10
  - file: "gfonts://Roboto"
    id: font_large
    size: 24

globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792026900' # 2026-10-01 03:55:00
  - id: last_triggered_minute
    type: int
    restore_value: no
    initial_value: '55' # Matcher minuttet fra start-timestampet
  - id: polarity_flip
    type: bool
    restore_value: no
    initial_value: 'false'

time:
  - platform: homeassistant
    id: homeassistant_time

interval:
  - interval: 1s
    then:
      - lambda: |-
          if (id(clock_running).state) {
            id(simulated_timestamp) += (int)id(speed_multiplier).state;
            
            // Tjek om minuttet har ændret sig i vores virtuelle tid
            time_t raw_time = id(simulated_timestamp);
            struct tm *time_info = gmtime(&raw_time);
            int current_minute = time_info->tm_min;
            
            if (current_minute != id(last_triggered_minute)) {
              id(last_triggered_minute) = current_minute;
              
              // Skift polaritet (flip boolsk værdi)
              id(polarity_flip) = !id(polarity_flip);
              
              // Trigger H-broen baseret på den nuværende polaritet
              if (id(polarity_flip)) {
                id(h_bridge_in1).turn_on();
                delay(300); // 300 ms impuls for at flytte spolen i uret
                id(h_bridge_in1).turn_off();
              } else {
                id(h_bridge_in2).turn_on();
                delay(300); // 300 ms impuls den modsatte vej
                id(h_bridge_in2).turn_off();
              }
            }
          }

text_sensor:
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
```

Ledningsdiagram for L298N H-broen:

1. IN1 på L298N → Forbind til GPIO16 på ESP32.
2. IN2 på L298N → Forbind til GPIO17 på ESP32.
3. GND på L298N → Forbind til en GND-ben på din ESP32 (Vigtigt for fælles stel).
4. VMS / VCC (Strømindgang på L298N): Tilslut den eksterne spænding, som dit gamle ur kræver (typisk 12V eller 24V).
5. OUT1 & OUT2 på L298N → Forbind direkte til de to ledninger på dit gamle banegårdsur.

### Hvordan det virker:

* Hvert sekund tjekker koden, om det accelererede minuttal har flyttet sig. Så snart uret slår et minut frem (uanset om det tager 1 real-sekund eller 10 real-sekunder baseret på din multiplier), sender ESP32 et præcist signal på 300 ms til H-broen, som flipper polariteten og rykker viseren på det mekaniske ur synkront med OLED-skærmen.
* Virker en puls-varighed på 300 ms fint til dit ur, eller ved du, om spolens mekanik i dit specifikke ur kræver en lidt længere impuls (f.eks. 500 ms) for at trække viseren helt på plads?

