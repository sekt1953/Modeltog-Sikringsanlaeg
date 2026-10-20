# 4_CanBus_jeg vil gerne hav en hurtig introduktion til hvordan du udruller den fælles pakkefil i ESPHome

## jeg vil gerne hav en hurtig introduktion til hvordan du udruller den fælles pakkefil i ESPHome?

Her er en hurtig guide til, hvordan du placerer og udruller din fælles pakkefil (can_base.yaml), så dine tre ESP32-enheder kan finde den.

### Step 1: Placer den fælles fil det rigtige sted

ESPHome leder efter pakker lokalt i det bibliotek, hvor dine enhedskonfigurationer (.yaml-filer) ligger.

1. Åbn din fil-editor i Home Assistant (f.eks. File Editor- eller VS Code-add-on).
2. Gå til mappen config/esphome/.
3. Opret en ny fil og navngiv den nøjagtigt can_base.yaml.
4. Indsæt den fælles CAN-bus kode i filen og gem:

```yaml
# can_base.yaml
substitutions:
  can_tx_pin: "GPIO22" # Standard pinde, hvis andet ikke angives
  can_rx_pin: "GPIO23"
  can_speed: "250kbps"

canbus:
  - platform: esp32_can
    tx_pin: ${can_tx_pin}
    rx_pin: ${can_rx_pin}
    bit_rate: ${can_speed}
    id: can_bus
```

### Step 2: Forbind dine enheder til pakken

I dine tre andre enhedsfiler (f.eks. esp32_gateway.yaml, esp32_node2.yaml, esp32_node3.yaml) refererer du nu til pakken øverst i koden vha. !include:

```yaml
packages:
  can_network: !include can_base.yaml
```

Når ESPHome skal kompilere koden til en af dine ESP32'ere, tager den automatisk indholdet fra can_base.yaml og fletter det usynligt sammen med enhedens egen kode.

### Step 3: Kompiler og udrul (Flash)

Nu gør du fuldstændig, som du plejer i ESPHome-brugerfladen:

1. Åbn ESPHome-dashboardet i Home Assistant.
2. Find din enhed (f.eks. din Waveshare Gateway) og klik på de tre prikker og vælg Install (eller klik Install på det samlede overblik).
3. ESPHome vil nu hente den fælles pakke i baggrunden, tjekke for pin-konflikter, kompilere firmwaren og udrulle den til din ESP32 (enten via netværket/OTA eller USB).

💡 Det smarte i fremtiden (Når alt kører)

Hvis du en dag finder ud af, at 250kbps er for langsomt til din CAN-bus, eller du har brug for at ændre hastigheden til 500kbps, skal du kun ændre det ét sted:

1. Åbn can_base.yaml.
2. Ret can_speed: "250kbps" til can_speed: "500kbps".
3. Gem filen, og klik Install på dine 3 ESP32'ere i dashboardet. De henter alle tre den nye hastighed automatisk.

Er du klar til at oversætte og udrulle den første enhed, eller vil du have mig til at kigge på, hvordan du overvåger CAN-bus-trafikken i loggen for at sikre dig, at de tre ESP32'ere rent faktisk snakker sammen?

