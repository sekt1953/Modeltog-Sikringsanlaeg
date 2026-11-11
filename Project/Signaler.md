# Signal styring

At bruge PCA9685PW til jeres gamle signaler er et helt fantastisk valg. Med 12-bit PWM (4096 trin) og ESPHomes indbyggede overgange (transition_length) kan du simulere den bløde, glødende optænding og nedtoning fra gamle glødepærer i stedet for det hårde, digitale LED-blink.
Når du kobler gamle signaler til en PCA9685PW (der kører på 3,3V), skal du være opmærksom på strømbegrænsningen og spændingen på signalerne:

## 1. Hardware: Pas på strømmen på PCA9685-benene

PCA9685PW er lavet til små lysdioder, ikke direkte til gamle glødepærer eller kraftige signaler.
* Strømgrænse pr. ben: Maks 25 mA (sink/træk mod GND) og kun 10 mA (source/udgang ved 3,3V). Samlet maks 400 mA for hele chippen.
* Løsning til gamle glødepærer (eller 12V LED-signaler): Sæt en ULN2803A direkte på udgangene af din PCA9685PW!

* PCA9685 styrer indgangene på ULN2803A med 3,3V PWM.
* ULN2803A trækker derefter signalerne mod GND fra jeres 12V-forsyning (eller en passende spænding til pærerne) med op til 500 mA pr. kanal.
* Bonus: Da de fleste gamle modelbanesignaler har fælles plus (Common Anode), passer en ULN2803A som fod i hose, fordi den netop trækker de enkelte farver mod minus (GND).

## 2. ESPHome-konfiguration med blød glødepære-effekt

I ESPHome understøttes PCA9685 direkte over samme I2C-bus (SDA: GPIO1, SCL: GPIO2) som din MCP23017. Ved at bruge monochromatic lys i stedet for binary, får du fuld adgang til bløde overgange.
Her er koden, du kan tilføje til din station_sub_node.yaml:

```yaml
# Tilføj PCA9685 på den eksisterende I2C-bus (Standard adresse 0x40, eller 0x41 hvis A0=3.3V)
pca9685:
  - id: pca_signaler
    address: 0x40
    frequency: 1000 # 1000 Hz fjerner al flimmer

# Output-kanaler på PCA9685 (0 til 15)
output:
  - platform: pca9685
    pca9685_id: pca_signaler
    channel: 0
    id: pca_sig1_roed
  - platform: pca9685
    pca9685_id: pca_signaler
    channel: 1
    id: pca_sig1_groen
  - platform: pca9685
    pca9685_id: pca_signaler
    channel: 2
    id: pca_sig1_gul

# Monochromatic lys giver blød glødepære-effekt via default_transition_length
light:
  - platform: monochromatic
    id: signal_roed
    output: pca_sig1_roed
    default_transition_length: 0.4s # Pæren gløder blødt op og ned på 0,4 sekunder!
  - platform: monochromatic
    id: signal_groen
    output: pca_sig1_groen
    default_transition_length: 0.4s
  - platform: monochromatic
    id: signal_gul
    output: pca_sig1_gul
    default_transition_length: 0.4s
```

Bemærk: Hvis du bruger en ULN2803A mellem PCA9685 og signalet, virker logikken som standard (høj PWM på PCA = ULN tænder = pæren lyser). Du behøver derfor ikke invertere udgangene.

## 💡 Ekstra bonus ved PCA9685 på noden

Med 16 kanaler på PCA9685 kan én enkelt node nu nemt trække 4 til 5 komplette flerspektrede signaler (f.eks. både indkørselssignal, udkørselssignal og dværgsignaler på stationen), samtidig med at MCP23017 styrer de 4 sporskifter.
Er jeres gamle signaler med rigtige små glødepærer (og i så fald hvilken spænding er de bygget til, f.eks. 12V eller 16V), eller er de ombygget til LED med fælles plus?
AI-svar kan indeholde fejl. Få flere oplysninger

vi kommmer til at tale om nogle andre signalbilleder end rød grøn og gul i vores verden snakker vi om stop, stop og ryg frem, kør igennem mm.

Det giver rigtig god mening – når I kører efter rigtige DSB-signalreglementer (som I-signaler, U-signaler og dværgsignaler), handler det ikke bare om farver, men om signalbegreber som Stop, Stop og ryk frem (rangering/forsigtig fremrykning) og Kør igennem (eller Kør med nedsat hastighed).
Med PCA9685PW og ESPHome er det super nemt at mappe disse præcise danske signalbegreber til de rigtige lamper (og eventuelle blink- eller vinkelfunktioner), så hele jeres system – fra Home Assistant og CAN-bussen ned til signalerne, oversigtstavlerne og stationspulterne – taler det helt rigtige jernbanesprog.
Her er, hvordan vi bygger det ind i jeres CAN-protokol og ESPHome-kode:

## 1. Definer jeres CAN-talkoder efter DSB-signalbegreber

I stedet for at tænke i farver, tildeler I hvert signalbegreb en fast talkode (1 byte) på CAN-bussen. Her er et eksempel på en standardtabel, som I kan udvide efter behov:

|CAN-kode|Signalbegreb|Typisk lampebillede (DSB-eksempel)|
|:---|:---|:---|
|0x00|Stop|1 rød lampe lyser konstant|
|0x01|Kør igennem|1 grøn lampe (evt. + grøn pil/streg afhængig af signaltype)|
|0x02|Kør (med nedsat hastighed)|1 grøn + 1 gul lampe (ved afvigende sporskifte)|
|0x03|Stop og ryk frem|1 rød + 1 gul lampe (eller 2 skråtstillede hvide i dværgsignal)|
|0x04|Kør forsigtigt|1 gul lampe (f.eks. indkørsel til besat spor / blindspor)|
|0x05|Forbikørsel tilladt (Ranger)|2 hvide lamper (dværgsignal)|


## 2. ESPHome-kode til station_sub_node.yaml (Signalbegreber på PCA9685)

På ESP32-S3 noden opretter I de individuelle lamper fra jeres PCA9685 (f.eks. rød, grøn, gul, hvid 1, hvid 2). Derefter laver vi en on_frame funktion, der slukker alle lamper blødt og tænder præcis den kombination, som det modtagne signalbegreb kræver.

```yaml
# PCA9685 udgange til et komplet dansk hoved-/dværgsignal
output:
  - { platform: pca9685, pca9685_id: pca_signaler, channel: 0, id: out_roed }
  - { platform: pca9685, pca9685_id: pca_signaler, channel: 1, id: out_groen }
  - { platform: pca9685, pca9685_id: pca_signaler, channel: 2, id: out_gul }
  - { platform: pca9685, pca9685_id: pca_signaler, channel: 3, id: out_hvid_1 }
  - { platform: pca9685, pca9685_id: pca_signaler, channel: 4, id: out_hvid_2 }

# De enkelte lamper med blød glødepære-effekt (0.4 sekunder)
light:
  - { platform: monochromatic, id: lampe_roed,   output: out_roed,   default_transition_length: 0.4s }
  - { platform: monochromatic, id: lampe_groen,  output: out_groen,  default_transition_length: 0.4s }
  - { platform: monochromatic, id: lampe_gul,    output: out_gul,    default_transition_length: 0.4s }
  - { platform: monochromatic, id: lampe_hvid_1, output: out_hvid_1, default_transition_length: 0.4s }
  - { platform: monochromatic, id: lampe_hvid_2, output: out_hvid_2, default_transition_length: 0.4s }

# CAN-bus modtager, der oversætter signalbegreber til lampekombinationer
canbus:
  - id: lokal_can
    platform: esp32_can
    rx_pin: GPIO4
    tx_pin: GPIO5
    bit_rate: 125kbps
    on_frame:
      - can_id: ${can_id_signal}
        then:
          - lambda: |-
              if (x.size() > 0) {
                uint8_t begreb = x[0];
                
                // Sluk først alle lamper blødt
                id(lampe_roed).turn_off();
                id(lampe_groen).turn_off();
                id(lampe_gul).turn_off();
                id(lampe_hvid_1).turn_off();
                id(lampe_hvid_2).turn_off();

                // Tænd derefter det valgte signalbegreb
                if (begreb == 0x00) {        // STOP
                  id(lampe_roed).turn_on();
                } 
                else if (begreb == 0x01) {   // KØR IGENNEM
                  id(lampe_groen).turn_on();
                } 
                else if (begreb == 0x02) {   // KØR (Nedsat hastighed)
                  id(lampe_groen).turn_on();
                  id(lampe_gul).turn_on();
                } 
                else if (begreb == 0x03) {   // STOP OG RYK FREM
                  id(lampe_roed).turn_on();
                  id(lampe_gul).turn_on();
                } 
                else if (begreb == 0x04) {   // KØR FORSIGTIGT
                  id(lampe_gul).turn_on();
                }
                else if (begreb == 0x05) {   // RANGER / FORBIKØRSEL TILLADT
                  id(lampe_hvid_1).turn_on();
                  id(lampe_hvid_2).turn_on();
                }
              }
```

## 3. Sådan ser det ud i Home Assistant (station01_lan.yaml)

På stationens LAN-gateway opretter du en select-menu med præcis de danske begreber, som jeres stationspersonalet og køreplan-automatiseringer bruger.

```yaml
select:
  - platform: template
    name: "I-Signal Nord"
    icon: "mdi:traffic-light"
    options:
      - "Stop"
      - "Stop og ryk frem"
      - "Kør forsigtigt"
      - "Kør (nedsat hastighed)"
      - "Kør igennem"
    optimistic: true
    set_action:
      - lambda: |-
          if (x == "Stop") id(lokal_can).send_data(0x011, false, {0x00});
          else if (x == "Kør igennem") id(lokal_can).send_data(0x011, false, {0x01});
          else if (x == "Kør (nedsat hastighed)") id(lokal_can).send_data(0x011, false, {0x02});
          else if (x == "Stop og ryk frem") id(lokal_can).send_data(0x011, false, {0x03});
          else if (x == "Kør forsigtigt") id(lokal_can).send_data(0x011, false, {0x04});
```

Fordelen på oversigtstavler og stationspulter

Fordi hele anlægget bruger de samme CAN-talkoder (0x00 til 0x05), kan en ESP32 på lokoførernes oversigtstavle eller stationspulten modtage nøjagtig den samme kode 0x03 (Stop og ryk frem). Hvis oversigtstavlen kun har en rød og en gul diode ud for signalet, tænder den bare dem begge – eller blinker med den gule, hvis I foretrækker det visuelt på tavlen.
Hvis I bruger blinkende signalbegreber (som f.eks. DSB-forsignaler med blinkende grønt eller gult lys, eller særlige rangerbegreber), vil du så have, at jeg viser, hvordan man laver et blødt glødepære-blink på PCA9685 direkte i ESPHome-koden?
AI-svar kan indeholde fejl. Få flere oplysninger

## vi har også hastigsheds viser på nogle stationer

Hastighedsvisere (som DSB's klassiske Z-signaler / hastighedsindikatorer) passer helt perfekt ind i jeres eksisterende PCA9685-opsætning.
Afhængigt af hvordan jeres fysiske hastighedsvisere er opbygget på banen, er der to måder at styre dem på fra den samme PCA9685-chip:

## 1. To typer hastighedsvisere på modelbanen

* Fast ciffer-tavle (Lampe-matrix eller enkelte tal): Hvis hastighedsviseren blot har 2–3 faste tal (f.eks. en lampe der oplyser et "4"-tal for 40 km/t og en anden for "6" for 60 km/t), bruger hvert tal bare én kanal på din PCA9685 (evt. via ULN2803A).
* 7-segment display (Til flere forskellige tal): Hvis I har bygget hastighedsvisere med små 7-segment lysdioder/pærer, der kan danne tallene 0–9 (f.eks. 4, 6, 8, 10), bruger de 7 kanaler på PCA9685.

## 2. Udvidelse af jeres CAN-protokol (Byte 2 til hastighed)

I stedet for at bruge et helt nyt CAN ID til hastighedsviseren, kan I blot udvide den eksisterende signal-pakke (can_id_signal) fra 1 byte til 2 bytes:

* Byte 0 (x): Signalbegreb (0x00 = Stop, 0x01 = Kør igennem, 0x02 = Kør med nedsat hastighed osv.).
* Byte 1 (x): Hastighedsviser i dekader (0x00 = Slukket, 0x04 = 40 km/t, 0x06 = 60 km/t, 0x08 = 80 km/t).

Fordel: Når hovedsignalet skifter til Stop (0x00), slukker hastighedsviseren automatisk i samme pakke, præcis som på den rigtige jernbane.

## 3. ESPHome-kode til station_sub_node.yaml (Med faste hastigheds-udgange)

Her er et eksempel på, hvordan du tilføjer hastighedsviseren (her for 40 km/t og 60 km/t) på de næste ledige kanaler på din PCA9685:

```yaml
# Ekstra PCA9685 kanaler til hastighedsviser (f.eks. kanal 5 og 6)
output:
  - { platform: pca9685, pca9685_id: pca_signaler, channel: 5, id: out_hast_40 }
  - { platform: pca9685, pca9685_id: pca_signaler, channel: 6, id: out_hast_60 }

light:
  - { platform: monochromatic, id: hast_40, output: out_hast_40, default_transition_length: 0.4s }
  - { platform: monochromatic, id: hast_60, output: out_hast_60, default_transition_length: 0.4s }

# Opdateret CAN-modtager der læser både signalbegreb (Byte 0) og hastighed (Byte 1)
canbus:
  - id: lokal_can
    platform: esp32_can
    rx_pin: GPIO4
    tx_pin: GPIO5
    bit_rate: 125kbps
    on_frame:
      - can_id: ${can_id_signal}
        then:
          - lambda: |-
              if (x.size() > 0) {
                uint8_t begreb = x;
                uint8_t hastighed = (x.size() >= 2) ? x : 0x00;

                // 1. Sluk alle hovedlamper og hastighedsvisere først
                id(lampe_roed).turn_off();
                id(lampe_groen).turn_off();
                id(lampe_gul).turn_off();
                id(lampe_hvid_1).turn_off();
                id(lampe_hvid_2).turn_off();
                id(hast_40).turn_off();
                id(hast_60).turn_off();

                // 2. Tænd hovedsignalets begreb
                if (begreb == 0x00) id(lampe_roed).turn_on();
                else if (begreb == 0x01) id(lampe_groen).turn_on();
                else if (begreb == 0x02) { id(lampe_groen).turn_on(); id(lampe_gul).turn_on(); }
                else if (begreb == 0x03) { id(lampe_roed).turn_on(); id(lampe_gul).turn_on(); }
                else if (begreb == 0x04) id(lampe_gul).turn_on();
                else if (begreb == 0x05) { id(lampe_hvid_1).turn_on(); id(lampe_hvid_2).turn_on(); }

                // 3. Tænd hastighedsviseren (kun hvis signalet ikke viser Stop)
                if (begreb != 0x00) {
                  if (hastighed == 0x04) id(hast_40).turn_on();
                  else if (hastighed == 0x06) id(hast_60).turn_on();
                }
              }
```

## 💡 Sådan styrer du det fra LAN-gatewayen / Home Assistant

I din station01_lan.yaml opretter du blot en ekstra select-menu for hastigheden (f.eks. "Slukket", "40 km/t", "60 km/t"), og lader LAN-gatewayen sende begge bytes sammen i én pakke (f.eks. {0x02, 0x04} for Kør med nedsat hastighed + 40 km/t).
Er jeres hastighedsvisere bygget med faste lamper pr. tal (som i koden ovenfor), eller er det 7-segment displays, hvor vi skal kode de enkelte segmenter (A–G) til at danne tallene?
AI-svar kan indeholde fejl. Få flere oplysninger
