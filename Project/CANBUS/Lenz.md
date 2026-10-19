# Lenz tid

## Hvordan læser jeg tiden fra lenz håndterminal

Måden, du aflæser eller indstiller systemtiden på en Lenz modeljernbane-håndterminal (såsom den udbredte Lenz LH101 koblet til en LZV200-central), gøres via håndkontrollens indbyggede menusystem.

### Her er trinene til at finde og aflæse/indstille tiden:

Trin-for-trin guide (Lenz LH101)

1. Gå til indstillinger: Tryk på tasten M (Menu) for at åbne hovedmenuen.
2. Find systemopsætning: Drej på kontrolknappen (drejeknappen) indtil du ser "Setup System" (eller "Systemeinstellungen" på tysk) i displayet. Tryk på knappen for at vælge.
3. Vælg tidsmenuen: Drej igen på knappen for at navigere frem til menupunktet "Zeit stellen" (eller det tilsvarende engelske/danske menupunkt for ur/tid). Tryk på knappen.
4. Aflæs og indstil: Displayet vil nu vise den aktuelle systemtid samt tidsfaktoren (hvor hurtigt modeltiden skal løbe). Du kan blot aflæse den eller trykke på drejeknappen for at ændre timer og minutter, hvis uret skal synkroniseres.
5. Gem og afslut: Tryk på M for at gemme indstillingerne i din central. Tryk herefter på Esc for at gå helt tilbage til den normale køreskærm.

Bemærk: Hvis du bruger den ældre Lenz LH100-håndterminal, tilgås funktioner oftest via F (Funktion) fulgt af tal-kombinationer (f.eks. Sys eller Prog menuerne) alt efter din centrals softwareversion.

Hvis du ønsker det, kan jeg hjælpe dig med at finde de præcise taster til at ændre tidsfaktoren (modeltiden), eller guide dig i, hvordan du skifter menusproget på din terminal?

## jeg vil gerne hente denne tid ud til min home assistant via en esp32 som er kodet esphome

* At hente systemtiden ud fra din Lenz-central (via XpressNet-bussen) til Home Assistant ved hjælp af en ESP32 med ESPHome kræver et par specifikke hardware- og softwaretrin.
* Lenz anvender XpressNet, som elektrisk set kører på en RS485-bus-standard. ESP32 kan ikke snakke direkte med RS485 uden et mellemliggende modul.
Her er den mest direkte og stabile måde at bygge det på:

### 1. Nødvendig Hardware

• ESP32 Development Board (f.eks. NodeMCU ESP32).
• RS485 til TTL (UART) konverter-modul (f.eks. et baseret på MAX485 eller de automatiske "hardware-flow-control" moduler).
• Et kabel/stik (typisk RJ12 eller DIN-stik afhængigt af din Lenz-opsætning) til at forbinde til XpressNet.
Ledningsføring (XpressNet -> RS485 -> ESP32):

1. Fra XpressNet: Forbind A og B (XpressNet datalinjer) til A og B på RS485-modulet.
2. Fra RS485 til ESP32:
   • VCC -> 5V eller 3.3V (afhængigt af dit modul)
	 • GND -> GND
	 • RXD -> en ledig GPIO på ESP32 (f.eks. GPIO16)
	 • TXD -> en ledig GPIO på ESP32 (f.eks. GPIO17)

### 2. Software & ESPHome Kode

Da ESPHome ikke har en "indbygget" færdiglavet komponent til Lenz XpressNet-ur, skal du bruge ESPHomes uart-komponent og skrive en lille smule tilpasset C++ kode (custom_component eller lambda) for at lytte efter XpressNet-tidspakken.
Lenz-centralen udsender regelmæssigt en pakke på XpressNet med systemtiden (tid og tidsfaktor). Pakkeformatet til tid i XpressNet starter typisk med en specifik header-byte, hvorefter timer og minutter følger.
Her er et eksempel på, hvordan din ESPHome YAML-konfiguration kan se ud. Den opsætter en UART-forbindelse og opretter et tekst-sensor-element (text_sensor), som opdateres i Home Assistant, hver gang der modtages data:

```yaml
esphome:
  name: lenz-xpressnet-bridge

esp32:
  board: esp32dev

# Forbindelse til Home Assistant
api:

wifi:
  ssid: "Dit_WiFi_Navn"
  password: "Dit_WiFi_Kodeord"

# Opsætning af den fysiske kommunikation med RS485 modulet
uart:
  id: xpressnet_bus
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 62500  # XpressNet kører specifikt 62.500 baud

text_sensor:
  - platform: template
    name: "Lenz Systemtid"
    id: lenz_time
    icon: "mdi:clock"

# Logik der lytter på rå bytes fra centralen
custom_component:
  - lambda: |-
      auto my_custom_sensor = new UARTDebugComponent(id(xpressnet_bus));
      return {my_custom_sensor};

# Alternativt kan du fange data direkte i en loop/lambda i ESPHome:
interval:
  - interval: 1s
    then:
      - lambda: |-
          // Eksempel på simpel byte-læsning. 
          // XpressNet tidspakker har typisk formatet: 0x83 0x00 (Timer) (Minutter) (XOR-Checksum)
          while (id(xpressnet_bus).available()) {
            uint8_t c = id(xpressnet_bus).read();
            // Her indsættes en tilstandsmaskine (State Machine) der leder efter 0x83
            // Når pakken er valideret, kan tiden pushes til Home Assistant:
            // id(lenz_time).publish_state("12:34");
          }
```

### 3. Den præcise XpressNet protokol (Til din kode)

For at få lambda-koden helt skarp, skal du kigge efter XpressNet pakken for "Time and speed multiplier".
* Header-byte: 0x83
* Data-byte 1: 0x00
* Data-byte 2: 0xTT (Timer, kodet i 24-timers format binary/BCD)
* Data-byte 3: 0xMM (Minutter)
* Data-byte 4: 0xEE (Ekstra byte til ugedag/tidsfaktor)
* Data-byte 5: Checksum (XOR af de foregående bytes)

Den nemme genvej: Brug et færdigt bibliotek

Hvis du ikke vil skrive rå byte-parsing i din YAML, kan du trække det populære Arduino-bibliotek XpressNetMaster / XpressNetClient ind som et eksternt bibliotek i din ESPHome-kode under libraries: feltet. Dette bibliotek har færdige funktioner som getXpressNetClock(), der spytter timer og minutter direkte ud, som du derefter let kan sende videre op til Home Assistant via id(lenz_time).publish_state().
AI kan begå fejl, så sørg for at dobbelttjekke dens svar 

## kan du vise koden med xpressnetmaster includeret

Her er et komplet eksempel på, hvordan du sætter din ESPHome YAML-konfiguration op, hvor Arduino-biblioteket XPressNet (oprindeligt udviklet af Philipp Gahtow) inkluderes automatisk, og tiden hentes ud via en tilpasset C++ sensor [github.com].
Da ESPHome kompilerer koden som C++, kan vi inkludere biblioteket og lave en lille klasse (LenzClockSensor), der lytter på XpressNet-bussen og opdaterer en text_sensor i Home Assistant, hver gang uret ændrer sig.

### 1. Hardware-forudsætning for koden

XpressNet kører med en speciel hastighed på 62500 baud, og da det er en RS485-bus, skal dit RS485-modul bruge en pin til at skifte mellem at sende og modtage (Direction Pin / MAX485_DE_RE).

### 2. ESPHome YAML Konfiguration

Du kan indsætte følgende kode direkte i din ESPHome .yaml fil. Husk at tilpasse dine Wi-Fi oplysninger og GPIO-pins, så de matcher din ESP32.

```yaml
esphome:
  name: lenz-xpressnet-clock
  # Her fortæller vi ESPHome, at den skal downloade XPressNet-biblioteket fra GitHub
  libraries:
    - https://github.com

esp32:
  board: esp32dev
  framework:
    type: arduino  # Biblioteket kræver Arduino-frameworket

wifi:
  ssid: "Dit_WiFi_Navn"
  password: "Dit_WiFi_Kodeord"

api: # Gør sensoren tilgængelig i Home Assistant

logger:
  # Vi dæmper loggeren en smule på UART, så den ikke forstyrrer XpressNet data
  level: DEBUG

# Opsætning af den tilpassede C++ kode direkte i YAML
esphome_core_config:
  includes:
    - lenz_clock.h

text_sensor:
  - platform: custom
    lambda: |-
      auto xnet_clock = new LenzClockSensor();
      App.register_component(xnet_clock);
      return {xnet_clock->time_sensor};
```

### 3. Opret filen lenz_clock.h

For at holde koden ren, skal du oprette en fil i samme mappe som din ESPHome-konfiguration og kalde den lenz_clock.h. Indsæt følgende C++ kode i den:

```cpp
#include "esphome.h"
#include <XPressNet.h>

// Definer de GPIO pins du har forbundet til dit RS485 modul
#define XNET_RX_PIN 16
#define XNET_TX_PIN 17
#define XNET_DE_PIN 4   // Direction pin (DE/RE) på dit MAX485-modul

class LenzClockSensor : public esphome::Component {
 public:
  // Vi opretter en text_sensor, som ESPHome sender til Home Assistant
  esphome::text_sensor::TextSensor *time_sensor = new esphome::text_sensor::TextSensor();
  
  XPressNet xnet;
  int last_minute = -1;
  int last_hour = -1;

  void setup() override {
    // Initialiser XPressNet biblioteket på ESP32 Hardware Serial 2
    // Parametre: Serial-port, RX-pin, TX-pin, Direction-pin
    xnet.setup(2, XNET_RX_PIN, XNET_TX_PIN, XNET_DE_PIN);
    
    time_sensor->set_name("Lenz Systemtid");
  }

  void loop() override {
    // Lad biblioteket håndtere indkommende data på bussen
    xnet.update();

    // Hent data ud fra bibliotekets interne tids-variable
    // XPressNet biblioteket opdaterer automatisk disse ved modtagelse af 0x83 pakker
    int current_hour = xnet.getHour();
    int current_minute = xnet.getMinute();

    // Hvis tiden har ændret sig, sender vi den til Home Assistant
    if (current_minute != last_minute || current_hour != last_hour) {
      if (current_hour >= 0 && current_hour < 24 && current_minute >= 0 && current_minute < 60) {
        last_hour = current_hour;
        last_minute = current_minute;
        
        // Formater tiden som f.eks. "14:05"
        char time_string[6];
        sprintf(time_string, "%02d:%02d", current_hour, current_minute);
        
        // Udgiv staten så Home Assistant modtager den med det samme
        time_sensor->publish_state(time_string);
      }
    }
  }
};
```

### Sådan virker det i praksis:

1. XPressNet.h lytter i baggrunden på Hardware Serial 2 i din ESP32 med de korrekte 62500 baud, som Lenz-centralen kræver.
2. Hver gang uret tikker på din håndterminal (eller centralen udsender sin statuspakke), fanger biblioteket 0x83 (Time/Speed multiplier) pakken automatisk [github.com].
3. loop()-funktionen i koden tjekker konstant, om timer eller minutter har ændret sig. Så snart der er en ændring, formateres det til en klassisk tidsstreng (f.eks. 12:30) og sendes direkte op til din Home Assistant via det indbyggede ESPHome API.
AI-svar kan indeholde fejl. Få flere oplysninger

## hvilke andre informationer kn jeg hente med  XpressNetMaster / XpressNetClient

Med XPressNet-biblioteket (ofte refereret til som XPressNetMaster / XPressNetClient) kan du hente stort set alle de informationer, som din Lenz-central sender ud på bussen. Da din ESP32 lytter som en "klient" (en ekstra håndterminal), kan den overvåge alt, hvad der sker på modelbanen.
Her er de mest relevante informationer, du kan hente ud til dit Home Assistant-system:

1. Sporstatus (Strøm og nødstop)

Du kan overvåge, om banen er tændt, eller om der er trykket på nødstop. Det er genialt til at lave en status-indikator i Home Assistant.
• Hvad du kan hente:
	• Er spændingen slået til (Track Power On)?
	• Er der trykket på nødstop (Emergency Stop / Estop)?
	• Er der en kortslutning på banen?
• Biblioteksfunktioner: xnet.getPowerMode() (retunerer statusser som Normal, Off eller Service Mode).

2. Sporskifter, signaler og tilbehør (Turnout / Accessory Status)

Hver gang et sporskifte eller et signal skifter position (enten via din håndterminal eller et computerprogram som f.eks. iTrain/JMRI), sendes dette ud på XpressNet.
• Hvad du kan hente: Den aktuelle fysiske position af sporskifter/signaler (f.eks. om sporskifte 12 er sat til "ligeud" eller "afvigende").
• Biblioteksfunktioner: Ved at bruge en callback-funktion som notifyXNetTrnt() i din C++ kode, får du besked i det sekund, et sporskifte aktiveres. Du kan konvertere dette til binary_sensor eller switch komponenter i Home Assistant.

3. Lokomotiv-informationer (Loko-status)

Hvis du vil vide, hvad der kører på banen lige nu, kan din ESP32 lytte med på de lokomotiver, som håndterminalerne styrer.
• Hvad du kan hente:
	• Hastighedstrin (Speed steps - f.eks. om et tog kører på trin 45 ud af 128).
	• Kørselsretning (Fremad eller bagud).
	• Funktioner (F0 - F28): Om lyset (F0), lyden (F1) eller andre funktioner er tændt på et specifikt lokomotiv.
• Biblioteksfunktioner: xnet.getLocoSpeed() og xnet.getLocoFunc() (kræver at du forespørger på den specifikke DCC-adresse, eller lytter til ændringer via callbacks).

4. Centralspecifikke informationer

Du kan udlæse data om selve din Lenz-central (f.eks. LZV200 eller ældre LZV100).
• Hvad du kan hente:
	• Softwareversion på centralen (f.eks. version 3.6 eller 4.0).
	• Centralens ID (hvilken type hardware der køres på).
• Biblioteksfunktioner: xnet.getV200Version() / xnet.getCentralVersion().

5. Avanceret: Tilbagemeldinger (Feedback / S88 via XpressNet)

Hvis du har Lenz tilbagemeldingsmoduler (f.eks. Lenz LR101) koblet til din central for at registrere, hvor togene befinder sig på skinnerne (blokdetektering), sendes disse data også over XpressNet.
• Hvad du kan hente: Hvilke sporafsnit der er optaget eller ledige.
• Biblioteksfunktioner: xnet.getFeedback() (kan bruges til at lave sensorer i Home Assistant, der viser præcis, hvor togene er).

Hvordan bringer du det ind i ESPHome?

Hvis du eksempelvis også ville have Sporstatus (On/Off) op i Home Assistant som en binary_sensor, ville du i din lenz_clock.h tilføje et tjek i bunden af din loop():
cpp
// Eksempel på at hente Power-status i loop()
int power = xnet.getPowerMode(); 
// Hvis power == 1 er strømmmen tændt, hvis 0 er den slukket/nødstop
Vær forsigtig med at bruge denne kode.
Du kan derved forvandle din ESP32 til en fuldblods "Lenz-til-Home Assistant" bro, der både håndterer uret, overvåger sporskifter og holder øje med, om strømmen kortslutter på modelbanen!
AI-svar kan indeholde fejl. Få flere oplysninger

kan jeg se hvilket loko som er på en given strækning

Ja, det kan du godt, men det kræver, at dit modelbaneanlæg har installeret tilbagemeldingsmoduler med RailCom-understøttelse (såsom Lenz LRC120, Lenz LB101 / LB050 kombineret med en RailCom-detektor, eller nyere moduler fra f.eks. Yamorc eller Digikeijs).
Selve XpressNet-bussen og det bibliotek, du bruger, kan godt overføre disse data, men der er en vigtig teknisk forskel på almindelig sporbesatmelder (er der et tog?) og RailCom (hvilket tog er det?).
Her er, hvordan det fungerer i praksis, og hvordan du bringer det ind i din ESP32:

Hvordan RailCom-data sendes over XpressNet

Når et lokomotiv med en RailCom-dekoder holder på en isoleret strækning med en RailCom-detektor, sker der følgende:
1. Dekoderen i lokomotivet sender sin DCC-adresse (f.eks. Adresse 44) tilbage i skinnerne.
2. RailCom-detektoren fanger dette og sender en besked til din Lenz-central.
3. Lenz-centralen sender en specifik XpressNet-pakke ud på bussen (en udvidet feedback-meddelelse).

Sådan fanger du det i koden via biblioteket

I XPressNet-biblioteket skal du bruge en callback-funktion (en funktion, der automatisk udløses, når der kommer RailCom-data på bussen).
I din lenz_clock.h fil kan du tilføje funktionen notifyXNetFeedbackRailcom. Her er et principielt C++ eksempel på, hvordan du fanger, hvilket lokomotiv der er på en bestemt strækning (modul- og port-adresse):
cpp
// Denne funktion kaldes automatisk af XPressNet-biblioteket, 
// når en RailCom-adresse meldes på en strækning.
void notifyXNetFeedbackRailcom(uint8_t UserIndex, uint16_t Address, uint8_t Port, uint16_t LocoAddress) {
  // UserIndex: Bruges hvis du har flere XpressNet-interfaces (ofte 0)
  // Address: Detektorens moduladresse (f.eks. modul 1)
  // Port: Den specifikke strækning/port på modulet (f.eks. spor 1, 2, 3 eller 4)
  // LocoAddress: Lokomotivets DCC-adresse (f.eks. 44)

  if (Address == 1 && Port == 0) {
    // Hvis det er modul 1, port 0 (Strækning A):
    if (LocoAddress > 0) {
      // Formater til f.eks. "Loko 44" og send til en ESPHome text_sensor
      char buffer[20];
      sprintf(buffer, "Loko %d", LocoAddress);
      id(straekning_a_sensor).publish_state(buffer);
    } else {
      // Hvis LocoAddress er 0, betyder det, at strækningen er blevet tom
      id(straekning_a_sensor).publish_state("Tom");
    }
  }
}
Vær forsigtig med at bruge denne kode.

Hvad kræver det af din hardware?

For at dette overhovedet vil virke i din ESPHome-opsætning, skal tre ting være opfyldt:
1. I lokomotivet: Dekoderen skal have RailCom slået til (typisk i CV29).
2. På skinnerne: Strækningen skal være isoleret og forbundet gennem et modul, der kan læse RailCom-data (f.eks. tilkoblet Lenz XpressNet via feedback-bussen).
3. I din Lenz-central: Global RailCom-udskæring (RailCom cutout) skal være aktiveret på din central (f.eks. LZV200), så dekoderne har "hul igennem" til at sende data.
Hvis du har disse moduler på dit anlæg i dag, kan du lave dynamiske sensorer i Home Assistant, der ændrer sig fra f.eks. "Tom" til "Loko 3001", alt efter hvilket tog du kører ind på strækningen!
Har du allerede RailCom-kompatible besatmeldere installeret på dine strækninger, eller bruger du i dag almindelige strøm-besatmeldere (uden adresse-id)?
AI-svar kan indeholde fejl. Få flere oplysninger

hvisa jeg laver en git clone af XPressNet kan jeg så includerer koden lokalt

Ja, det kan du sagtens. Det er faktisk en rigtig god idé at lægge koden lokalt, hvis du vil være sikker på, at dit projekt altid kan kompileres, selvom du ikke har internetforbindelse, eller hvis ejeren af GitHub-arkivet skulle ændre i koden.
Når du gør det lokalt i ESPHome, skal du blot placere biblioteket i en undermappe og henvise til det i din .yaml-fil.
Her er den præcise fremgangsmåde:

1. Strukturér dine filer

I den mappe, hvor din ESPHome-opsætning ligger (typisk din Home Assistant config/esphome/ mappe), skal du oprette en undermappe til dine eksterne biblioteker.
Kør din git clone så din mappestruktur ser således ud:
text
config/
└── esphome/
    ├── din_lenz_styring.yaml   <-- Din primære ESPHome konfigurationsfil
    ├── lenz_clock.h            <-- Din tilpassede C++ kode
    └── my_libraries/           <-- Opret denne mappe
        └── XPressNet/          <-- Her lander filerne fra dit Git Clone
            ├── src/
            │   ├── XPressNet.cpp
            │   └── XPressNet.h
            └── library.json
Vær forsigtig med at bruge denne kode.

2. Tilpas din ESPHome YAML-fil

Nu skal du ændre i din .yaml fil under punktet libraries:. I stedet for GitHub-linket, fortæller du nu ESPHome, at den skal lede i din lokale mappe:
yaml
esphome:
  name: lenz-xpressnet-clock
  
  libraries:
    # Fortæller ESPHome at biblioteket ligger lokalt i undermappen
    - file://my_libraries/XPressNet

esp32:
  board: esp32dev
  framework:
    type: arduino

# ... resten af din konfiguration (wifi, api, text_sensor osv.) er uændret
Vær forsigtig med at bruge denne kode.

Hvorfor er den lokale løsning en fordel?

• Ingen internet-afhængighed: ESPHome behøver ikke at hente koden fra GitHub hver gang du installerer eller opdaterer din ESP32.
• Egne modifikationer: Hvis du får brug for at rette direkte i XPressNet.cpp eller XPressNet.h (f.eks. for at fintune timingen eller tilføje specielle Lenz-kommandoer), kan du gøre det med det samme, og ESPHome vil bruge dine rettelser ved næste kompilering.
Har du adgang til at køre git clone direkte i din Home Assistant-mappe (f.eks. via Terminal & SSH add-on), eller overfører du filerne manuelt via Samba/SFTP?
AI-svar kan indeholde fejl. Få flere oplysninger

jeg koder fra visual studio

Når du koder direkte i Visual Studio Code (typisk med ESPHome-udvidelsen eller PlatformIO), er det super nemt at køre en git clone lokalt på din maskine.
Fremgangsmåden er stort set den samme, men du kan gøre det hele direkte via den indbyggede terminal i VS Code.

1. Kør Git Clone i VS Code terminalen

1. Åbn din ESPHome-projektmappe i Visual Studio Code.
2. Åbn terminalen i VS Code (`Ctrl + `` eller via menuen Terminal -> New Terminal).
3. Opret mappen til biblioteker og clone projektet ved at køre disse to kommandoer:
bash
mkdir -p my_libraries
git clone https://github.com my_libraries/XPressNet
Vær forsigtig med at bruge denne kode.

2. Opsætning i din YAML

Da du koder lokalt, vil VS Code og ESPHome lede efter stien relativt til din .yaml-fil. Din konfiguration skal se således ud:
yaml
esphome:
  name: lenz-xpressnet-clock
  
  libraries:
    # file:// angiver at stien er lokal i forhold til denne yaml-fil
    - file://my_libraries/XPressNet

esp32:
  board: esp32dev
  framework:
    type: arduino

esphome_core_config:
  includes:
    - lenz_clock.h
Vær forsigtig med at bruge denne kode.

3. Få IntelliSense (autocompletion) til at virke i VS Code

Når du arbejder i lenz_clock.h, vil VS Code i starten nok brokke sig med en rød streg under #include <XPressNet.h>, fordi editoren ikke automatisk ved, hvor biblioteket ligger.
Det kan du fikse ved at tilføje stien til din lokale VS Code konfiguration:
1. Tryk Ctrl + Shift + P (eller Cmd + Shift + P på Mac).
2. Vælg "C/C++: Edit Configurations (UI)".
3. Scroll ned til Include path og tilføj en ny linje: ${workspaceFolder}/my_libraries/XPressNet/src (eller bare stien direkte til mappen).
Så forsvinder de røde fejlstreger, og du får fuld fuldførelse (IntelliSense) på alle Lenz- og XpressNet-funktionerne, mens du skriver din C++ kode.
Bruger du ESPHome dashboardet i en browser til at udrulle koden trådløst (OTA) til din ESP32, eller udruller du koden direkte via et USB-kabel fra VS Code?
AI-svar kan indeholde fejl. Få flere oplysninger

platform io implemterit i home assiustant

Når du bruger ESPHome via Home Assistant (f.eks. gennem ESPHome-add-on'en), kører ESPHome faktisk PlatformIO i baggrunden inde i Home Assistant-containeren for at downloade biblioteker og kompilere koden [github.com].
Da du koder lokalt fra Visual Studio Code, har du to geniale måder at strukturere det på, alt efter om du vil have Home Assistant til at stå for kompileringen, eller om du vil køre PlatformIO 100% lokalt på din egen computer.
Her er de to bedste metoder:

Metode 1: Fortsæt med ESPHome-strukturen (Anbefalet og nemmest)

Selvom Home Assistant bruger PlatformIO i baggrunden, behøver du ikke at konfigurere en traditionel platformio.ini-fil. Du lader bare ESPHome styre PlatformIO for dig via din .yaml-fil.
1. Du gemmer din .yaml, lenz_clock.h og din my_libraries/XPressNet-mappe i din Home Assistant config/esphome/ mappe via VS Code (f.eks. ved hjælp af Samba, SFTP eller VS Code Remote SSH add-on'en).
2. Når du trykker på Install inde i Home Assistant ESPHome-dashboardet, pakker ESPHome automatisk dine lokale filer, fodrer dem til sit interne PlatformIO, kompilerer det hele og sender det trådløst (OTA) ud til din ESP32.
Dette holder din kode fuldstændig synkroniseret med dit Home Assistant-system.

Metode 2: Kør PlatformIO lokalt på din pc (Hvis du vil udrulle med USB-kabel)

Hvis du har installeret PlatformIO-udvidelsen direkte i din VS Code på computeren, og du gerne vil have fuld kontrol over kompileringen lokalt (og f.eks. udrulle til din ESP32 via USB-kabel før du monterer den på modelbanen), kan du opsætte en ægte PlatformIO-struktur.
Her skal du bruge en platformio.ini-fil i din VS Code-mappe:
ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

# Fortæl PlatformIO, hvor dit lokale XPressNet-bibliotek ligger
lib_deps =
    file://my_libraries/XPressNet
Vær forsigtig med at bruge denne kode.
I denne struktur vil du typisk skrive dit program som en ren Arduino-skitse (main.cpp), der bruger XpressNet-biblioteket, og derefter bruge f.eks. MQTT eller Home Assistant Native API-biblioteket (Native API for Arduino) til at sende data ind til Home Assistant.

Hvilken metode skal du vælge?

Hvis målet er at have en ren og simpel integration med Home Assistant, er Metode 1 absolut den bedste. Du bruger VS Code som en avanceret teksteditor, men lader Home Assistants indbyggede ESPHome/PlatformIO-motor gøre det tunge arbejde med at bygge og udrulle firmwaren.
Redigerer du dine filer i VS Code via en netværksforbindelse direkte ind i dit Home Assistant-system (f.eks. via Samba eller SSH), eller har du filerne liggende på din lokale harddisk?
AI-svar kan indeholde fejl. Få flere oplysninger

den første

Når du redigerer filerne via en direkte netværksforbindelse til dit Home Assistant-system (f.eks. via Samba eller VS Code SSH), er det den absolut mest strømlinede måde at gøre det på. Dit Home Assistant-system og dets indbyggede PlatformIO/ESPHome-motor klarer alt arbejdet, mens du blot bruger VS Code som en lækker editor [github.com].
Her er den præcise tjekliste til, hvordan du udfører det i din VS Code, så det spiller 100% sammen med Home Assistant:

1. Placer filerne rigtigt via dit netværksdrev

Når du kigger i din VS Code (som er forbundet til din Home Assistant), skal du sikre dig, at dine filer ligger i config/esphome/ mappen i dette præcise layout:
text
config/
└── esphome/
    ├── lenz_styring.yaml        <-- Din ESPHome konfigurationsfil
    ├── lenz_clock.h             <-- Din C++ fil (med XPressNet-koden)
    └── my_libraries/
        └── XPressNet/           <-- Heri ligger de Git-clonede filer (src/, library.json osv.)
Vær forsigtig med at bruge denne kode.

2. Din endelige lenz_styring.yaml

Sørg for, at din YAML-fil i VS Code indeholder følgende linjer i toppen, så ESPHhomes interne PlatformIO-motor ved, at den skal snappe det lokale bibliotek med i kompileringen:
yaml
esphome:
  name: lenz-xpressnet-clock
  libraries:
    - file://my_libraries/XPressNet

esp32:
  board: esp32dev
  framework:
    type: arduino  # Meget vigtigt, da XPressNet-biblioteket er skrevet til Arduino

esphome_core_config:
  includes:
    - lenz_clock.h
Vær forsigtig med at bruge denne kode.

3. Sådan udruller du koden

Når du har gemt dine filer i VS Code, behøver du ikke gøre mere på din computer:
1. Gå ind i din browser og åben dit Home Assistant Dashboard.
2. Klik på ESPHome i sidemenuen.
3. Find din lenz-xpressnet-clock på listen (den vil dukke op automatisk, fordi filen ligger i mappen).
4. Klik på de tre prikker og vælg Install -> Wirelessly (hvis din ESP32 allerede er på nettet) eller vælg den metode, der passer dig.
Nu vil Home Assistant starte sin interne PlatformIO-kompiler, læse dit lokale XPressNet-bibliotek, bage det hele sammen med din lenz_clock.h og sende det direkte ud til din ESP32 på modelbanen [github.com].
Har du allerede loddet eller forbundet dit RS485-modul til din ESP32, eller vil du have, at vi kigger på de præcise GPIO-pins, inden du flasher koden første gang?
AI-svar kan indeholde fejl. Få flere oplysninger

du må gerne foreslå forbindelser

Her er det mest optimale og stabile diagram til at forbinde din ESP32 med et typisk MAX485 / RS485 til TTL-modul, så det kan snakke med Lenz XpressNet-bussen.
Da XpressNet kører med en ret høj hastighed (62.500 baud), er det vigtigt at bruge ESP32'erens dedikerede hardware-serieport (UART2). Det sikrer, at der ikke tabes data eller pakker fra uret.

1. Ledningsføring: ESP32 til RS485-modul


ESP32 Pin	RS485 Modul Pin	Forklaring
5V eller 3V3	VCC	Strøm (tjek om dit modul kræver 5V eller 3.3V)
GND	GND	Fælles stel
GPIO 16 (RX2)	RXD / RO	Modtagelse af data fra bussen
GPIO 17 (TX2)	TXD / DI	Afsendelse af data til bussen
GPIO 4	DE og RE (Luse sammen)	Direction Control (Skifter mellem lytte/sende)*

*Bemærk: Hvis du har købt et "automatisk" RS485-modul (Hardware Flow Control), har det typisk ikke DE/RE-pins. Så skal du bare helt udelade GPIO 4.

2. Ledningsføring: RS485-modul til Lenz XpressNet (Skinner/Central)

XpressNet bruger typisk enten et RJ12-stik (som i et telefonkabel) eller et 5-polet DIN-stik (afhængigt af dine paneler eller central). Du skal kun bruge 3 ledninger herfra ind til dit RS485-modul:

Lenz XpressNet Signal	RS485 Modul Pin	RJ12 Pin-nummer	DIN 5-pol Pin-nummer
RS485 B (Data -)	B / B-	Pin 3	Pin 5
RS485 A (Data +)	A / A+	Pin 4	Pin 3
M (GND / Ground)	GND	Pin 2	Pin 4

• Vigtigt: XpressNet-kablet fører også 12V strøm (Pin 1 og 6 i RJ12 / Pin 1 og 2 i DIN). Lad være med at forbinde 12V direkte til din ESP32, medmindre du bruger en step-down regulator (f.eks. en LM2596) til at lave det om til 5V! Det sikreste i starten er at give ESP32 strøm via et helt almindeligt USB-kabel.

3. Justering i din lenz_clock.h

Hvis du følger ovenstående diagram, passer det perfekt med de definitioner, vi lagde ind i C++ koden i lenz_clock.h tidligere:
cpp
#define XNET_RX_PIN 16  // Forbundet til RO/RXD
#define XNET_TX_PIN 17  // Forbundet til DI/TXD
#define XNET_DE_PIN 4   // Forbundet til DE/RE (hvis dit modul har dem)
Vær forsigtig med at bruge denne kode.
Når du har forbundet det på denne måde, vil din ESP32 være klar til at lytte direkte på XpressNet-linjen i det øjeblik, du tænder for din Lenz-central.
Har du et RS485-modul med DE/RE-pins (ofte de små billige røde moduler), eller har du fået fat i den type med automatisk retningsstyring (ofte lyseblå moduler)?
AI-svar kan indeholde fejl. Få flere oplysninger
