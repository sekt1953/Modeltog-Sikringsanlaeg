# AI Clock

## AI-oversigt

* To make a clock in ESPHome run 6 to 8 times faster than real time (often used for miniature models, fast-forwarding testing, or scale-model day/night cycles), you cannot use the native time component directly because it locks onto real-world NTP or Home Assistant servers.
* Instead, you can simulate an accelerated clock using template sensors and a high-frequency interval component.
* Here is how to build a flexible configuration that lets you adjust the speed multiplier (e.g., set it to 6.0 or 8.0) using a Home Assistant helper or a local slider.

### The ESPHome Configuration

```yaml
# Add a variable to track the simulated timestamp locally
globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800' # Default start date (e.g., Jan 1, 2026)

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
      struct tm *time_info = localtime(&raw_time);
      
      // Formats as HH:MM:SS
      strftime(formatted_time, sizeof(formatted_time), "%H:%M:%S", time_info);
      return std::string(formatted_time);

  - platform: template
    name: "Accelerated Date"
    id: accelerated_date
    icon: "mdi:calendar-clock"
    update_interval: 5s
    lambda: |-
      char formatted_date[30];
      time_t raw_time = id(simulated_timestamp);
      struct tm *time_info = localtime(&raw_time);
      
      // Formats as YYYY-MM-DD
      strftime(formatted_date, sizeof(formatted_date), "%Y-%m-%d", time_info);
      return std::string(formatted_date);
```

### How to use this time for displays (e.g., OLED or LED Matrix)

If you are passing this time into a physical display component, do not use it.strftime mapped to a real-time component. Instead, map it directly to your simulated_timestamp global inside your display component's lambda loop:

```yaml
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    lambda: |-
      char str[20];
      time_t raw_time = id(simulated_timestamp);
      struct tm *time_info = localtime(&raw_time);
      
      // Print the fast-forwarded time onto the screen
      strftime(str, sizeof(str), "%H:%M", time_info);
      it.print(0, 0, id(your_font), str);
```

### Why this approach works best:

1. Dynamic Scaling: You can alter the slider in Home Assistant to change the speed seamlessly from 6x to 8x without rebuilding or flashing the ESP module.
2. Standard Formatting: By using native C++ time_t and struct tm, standard flags like %H, %M, %d will flawlessly handle overflow (e.g., rolling minutes over to hours, and hours over to new days) automatically.
Would you like help adapting this to a specific display type (like an addressable LED strip or an LCD display), or do you need help triggering automations based on this accelerated time?

## I want 2 gpio sending signal to H-bridge l238n to change pålaritet on the signal to old clocks

* For at drive et ældre impuls-ur (som fasetårnure eller gamle DSB-stationure), skal du typisk vende polariteten (skifte mellem frem og tilbage) hver gang uret skal gå et minut frem.
* Da du vil køre tiden 6 til 8 gange hurtigere, kan vi integrere styringen af H-broen (L298N) direkte i den accelererede ur-kode. Hver gang det virtuelle ur skifter minut, sender ESPHome en kort puls og bytter polaritet på de to GPIO-pinde.
* Her er den komplette ESPHome-kode, der kombinerer den hurtige tid med H-bro-styringen:

## ESPHome Konfiguration

```yaml
globals:
  - id: simulated_timestamp
    type: int
    restore_value: no
    initial_value: '1792012800' # Starttidspunkt
  - id: last_simulated_minute
    type: int
    restore_value: no
    initial_value: '-1'
  - id: clock_polarity
    type: bool
    restore_value: no
    initial_value: 'false'

# Konfiguration af de 2 GPIO-pinde til din L298N H-bro
switch:
  - platform: gpio
    pin: GPIO12  # Forbind til IN1 på L298N
    id: h_bridge_in1
    name: "H-Bridge IN1"
  - platform: gpio
    pin: GPIO13  # Forbind til IN2 på L298N
    id: h_bridge_in2
    name: "H-Bridge IN2"

number:
  - platform: template
    name: "Time Speed Multiplier"
    id: speed_multiplier
    optimistic: true
    min_value: 1.0
    max_value: 20.0
    step: 0.5
    initial_value: 7.0 # Kører ca. 7 gange hurtigere som standard

interval:
  - interval: 1s
    then:
      - lambda: |-
          // 1. Fremskynd den virtuelle tid
          id(simulated_timestamp) += (int)id(speed_multiplier).state;

          // 2. Find ud af hvilket virtuelt minut vi er i nu
          time_t raw_time = id(simulated_timestamp);
          struct tm *time_info = localtime(&raw_time);
          int current_minute = time_info->tm_min;

          // 3. Tjek om det virtuelle ur har skiftet minut
          if (id(last_simulated_minute) == -1) {
              id(last_simulated_minute) = current_minute;
          } 
          else if (current_minute != id(last_simulated_minute)) {
              id(last_simulated_minute) = current_minute;
              
              // Skift polaritet (H-bro retning)
              id(clock_polarity) = !id(clock_polarity);
              
              // Udløs impulsen via H-broen baseret på polariteten
              if (id(clock_polarity)) {
                id(h_bridge_in1).turn_on();
                id(h_bridge_in2).turn_off();
              } else {
                id(h_bridge_in1).turn_off();
                id(h_bridge_in2).turn_on();
              }
              
              // Kør en timer der slukker for signalet igen efter en kort puls (f.eks. 400ms)
              // så spolen i uret ikke brænder af eller bruger strøm konstant
              arch_delay(400); 
              
              id(h_bridge_in1).turn_off();
              id(h_bridge_in2).turn_off();
          }

# Valgfri tekstsensor så du kan se tiden i Home Assistant
text_sensor:
  - platform: template
    name: "Accelerated Time"
    id: accelerated_time
    update_interval: 1s
    lambda: |-
      char formatted_time[9];
      time_t raw_time = id(simulated_timestamp);
      struct tm *time_info = localtime(&raw_time);
      strftime(formatted_time, sizeof(formatted_time), "%H:%M:%S", time_info);
      return std::string(formatted_time);
```

### Ledningsdiagram (ESP til L298N til Ur)

1. ESP GPIO12 → IN1 på L298N
2. ESP GPIO13 → IN2 på L298N
3. ESP GND → GND på L298N (Vigtigt: Fælles stel!)
4. Ekstern strømforsyning (f.eks. 12V/24V alt efter uret) → VCC og GND på L298N.
5. OUT1 og OUT2 på L298N → De to ledninger til dit gamle impuls-ur.
6. L298N Jumpere: Sørg for at ENA- og ENB-jumperne sidder på (så broen altid er aktiveret).

#### Hvordan det virker:

* Koden tjekker hvert sekund, om minuttallet på dit hurtige ur har ændret sig.
* Minut A: IN1 bliver HØJ, og IN2 bliver LAV i 400 millisekunder (strømmen løber én vej → uret tikker 1 minut frem). Derefter slukkes begge pinde for at skåne spolen.
* Minut B: Næste gang minuttet skifter, bliver IN2 HØJ, og IN1 bliver LAV i 400 millisekunder (strømmen vendes → uret tikker endnu 1 minut frem).

Hvis du oplever, at uret misser impulser eller tikker ustabilt ved den høje hastighed, kan det skyldes pulslængden. Skal vi justere pulslængden (de 400ms), eller har du brug for hjælp til at finde ud af, hvilken spænding dit specifikke ur kræver i L298N-modulet?


## 