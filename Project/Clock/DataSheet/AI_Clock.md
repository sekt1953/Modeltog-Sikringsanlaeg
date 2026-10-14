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