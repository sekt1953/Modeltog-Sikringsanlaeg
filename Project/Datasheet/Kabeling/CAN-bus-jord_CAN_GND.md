# CAN-bus-jord (CAN GND)

## når man har en CANBUS med H, L, jord, forbinder jord til power stel med 1Mohm og en kondensator , hvor stor skal kondensatoren så være ?

* Når du forbinder en lokal CAN-bus-jord (CAN GND) til strømforsyningens stel/chassis (Power Ground) via en 1 MΩ modstand parallelt med en kondensator, laver du en AC-koblet stel-adskillelse (ofte brugt til at bryde DC-jordløkker og bortlede HF-støj/ESD).
* Til dette formål skal kondensatoren typisk være 4,7 nF til 100 nF.
* Her er detaljerne for valg af kondensator:

### 1. Typiske standardværdier

* 10 nF til 47 nF: Det mest almindelige kompromis. Det giver en god balance mellem hurtig afledning af transienter (ESD) og dæmpning af højfrekvent støj, uden at lade for meget AC-strøm løbe mellem stelkredsløbene ved lave frekvenser.
* 100 nF: Anvendes også ofte, hvis du har problemer med lavere frekvensstøj, men det øger AC-koblingen mellem de to stelsystemer.

###  2. Vigtige specifikationer (Udover kapaciteten)

* Når du vælger kondensatoren, er dens fysiske egenskaber vigtigere end den præcise nanofarad-værdi:
  * Spænding (Voltage rating): Den skal kunne tåle store spændingsforskelle mellem systemerne. Vælg mindst 100V, men i biler eller industrielle miljøer anbefales ofte 500V til 1kV for at kunne modstå ESD-stød og spændings-transienter uden at kortslutte.
  * Dielektrikum (Type): Brug en keramisk kondensator (f.eks. af typen X7R), da de har fremragende egenskaber ved høje frekvenser og lave interne tab (ESR).

### Hvorfor denne kombination?

* 1 MΩ modstanden sørger for at aflade statisk elektricitet (DC) langsomt, så der ikke opbygges en høj DC-spændingsforskel mellem de to stelsystemer.
* Kondensatoren fungerer som en genvej (lav impedans) for høje frekvenser, hvilket tillader RF-støj og ESD-pulser at løbe harmløst til stel i stedet for at forstyrre CAN-transceiveren.

Vil du vide mere om, hvordan du placerer denne kobling i forhold til eventuel skærmet CAN-kabel (shield), eller leder du efter værdien til en split-termination i stedet?

## jeg vil gerne vide hvordan jeg placerer i forhold skærmet CAN-kabel

* Når du bruger et skærmet CAN-bus-kabel (Shielded Twisted Pair - STP) sammen med dit stel-adskillelsesnetværk (1 MΩ + kondensator), er det afgørende, hvor og hvordan skærmen forbindes for at undgå jordsløjfer (ground loops), der fungerer som antenner for støj.
Den mest effektive og standardiserede metode til industrielle miljøer og automotive applikationer er enkeltpunkts-jording af skærmen til DC, og AC-kobling i de resterende punkter.
* Her er præcis, hvordan du placerer komponenterne i systemet:

### 1. Hovedreglen for skærmen (Shield)

* Direkte jord i én ende (Master / Strømforsyningspunkt): Forbind kabelskærmen direkte til systemets centrale power-stel (chassis/jord) i kun én ende af bus-segmentet. Dette forhindrer, at der løber DC-strømme igennem skærmen på grund af spændingsforskelle mellem forskellige knudepunkter (nodes).
* AC-kobling i de andre ender (Slaves / Nodes): Ved alle andre knudepunkter på bussen må skærmen ikke forbindes direkte til lokal jord. Det er her, du indsætter dit netværk med 1 MΩ modstanden og kondensatoren.

### 2. Diagram for placering ved et knudepunkt (Node)

Inde i dine "slave"-knudepunkter skal forbindelsen struktureres således:

```text
[ CAN-KABEL ] ─────────────────────────────────────────► Til næste node
  ├── CAN H  ────────────────► [ CAN Transceiver H ]
  ├── CAN L  ────────────────► [ CAN Transceiver L ]
  │
  ├── CAN GND ──────┬────────► [ CAN Transceiver GND ]
  │                 │
  │               [ 1 MΩ ] || [ Kondensator (f.eks. 10 nF / 500V) ]  <-- Parallelkobling
  │                 │
  └─ SHIELD ────────┴────────► Tilsluttes lokal POWER STEL / CHASSIS
```

### 3. Sådan forbindes det i praksis

1. CAN H og CAN L går direkte til transceiverens databen.
2. Kablets skærm (Shield) føres frem til knudepunktets printplade (PCB).
3. Det lokale CAN-signal-jord (CAN GND) forbindes til skærmen/lokal power stel gennem parallelkoblingen af 1 MΩ modstanden og kondensatoren.
4. Hvorfor fungerer det? RF-støj, der opfanges af skærmen, bliver ledt direkte til apparatets chassis/power-stel gennem kondensatoren (lav impedans for høj frekvens), mens modstanden forhindrer statisk elektricitet i at bygge sig op. Fordi der ikke er en direkte DC-forbindelse i begge ender, kan der ikke opstå destruktive stenkredsløb (ground loops).

### Vigtigt for layoutet på printet (PCB)

* Hvis du designer dit eget print, skal du placere 1 MΩ modstanden og kondensatoren så tæt på kabeltilslutningen (stikket) som overhovedet muligt. Støjen skal ledes til kabinettet/stel inden den når at stråle ind i resten af dit kredsløb eller transceiver-chippen.
* Er dit CAN-bus-system installeret i et køretøj (hvor stel er bilens karosseri), eller er det til en industriel maskine / automation, hvor der trækkes separate jordledninger?
