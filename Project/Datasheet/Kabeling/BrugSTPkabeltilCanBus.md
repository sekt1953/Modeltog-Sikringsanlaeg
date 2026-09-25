# STP Kabling for modelbane med CANBUS

## Hvis jeg stp kabel til canbus hvor skal skærmen forbindes ?

* Når du bruger et STP (Shielded Twisted Pair) kabel til CAN-bus, skal skærmen (shield) som hovedregel kun forbindes til stel (GND) i én enkelt ende af kabelstrækket.
* Hvis du forbinder skærmen i begge ender, risikerer du at skabe en "stelsløjfe" (ground loop). Det kan få strøm til at løbe igennem skærmen, hvilket skaber mere støj og forstyrre CAN-bus-signalet i stedet for at beskytte det.
* Her er de vigtigste retningslinjer for, hvor og hvordan skærmen skal monteres:
  1. **Forbind kun i én ende (Hovedreglen)**
     * ***Hvor:*** Vælg typisk den ende, hvor den primære styreenhed (f.eks. master-controlleren, ECU'en eller PLC'en) sidder.
     * ***Hvorfor:*** Det sikrer, at støj, som skærmen opfanger, bliver ledt direkte til jord/stel ét centralt sted, uden at der opstår vandrende strømme i kablet.
  2. **Lad den anden ende være "svævende"**
     * I den modsatte ende (eller ved de andre noder på bussen) skal skærmen ***klippes af og isoleres*** med f.eks. krympeflex eller isolerbånd, så den ikke rører metaldele eller andre stelledninger ved et uheld.
  3. **Forbind skærmen korrekt (Brug CAN_GND rigtigt)**
     * ***Skærmen er IKKE det samme som CAN_GND:*** Du må aldrig bruge skærmen som erstatning for den fælles reference-jord (CAN_GND). CAN-bus kræver tre ting: CAN_H, CAN_L og CAN_GND (reference).
     * **Korrekt opsætning med et STP-kabel:**
       * ***Trådpar 1:** Bruges til CAN_H og CAN_L.
       * ***Ekstra leder/par:*** Bruges til CAN_GND (forbinder alle noder).
       * **Yderste metalskærm:** Forbindes til chassis/stel i én ende.
     * **Undtagelse: Meget lange kabelstræk**
       * Hvis din CAN-bus strækker sig over meget lange afstande i industrielt miljø, og der er stor spændingsforskel på jorden i de to ender, kan man forbinde skærmen til stel i den ene ende direkte, og i den anden ende gennem en kondensator (f.eks. 10-100 nF, højspænding). Det afleder højfrekvent støj i begge ender uden at skabe en DC-stelsløjfe. I 95% af alle installationer er det dog bedst blot at lade den anden ende være helt afbrudt.
       * For at give dig den helt præcise vejledning, vil jeg gerne høre lidt mere om dit projekt. Fortæl mig gerne:
         * Hvilket miljø skal bussen ligge i? (F.eks. i en bil/båd, til 3D-printer/Klipper, eller i et industrielt anlæg?)
         * Hvor langt forventer du, at kablet bliver samlet set?

## Instalationen skal bruges til modelbane med DCC styring

* Når du bruger CAN-bus til en modelbane med DCC-styring (f.eks. til Märklin Central Station, ESU EcoS eller Uhlenbrock), er der tale om et miljø med relativt korte afstande, men med en hel del højfrekvent støj fra skinnerne (hvor DCC-signalet og togmotorerne kører).
* Her er den helt præcise opsætning for din modelbane:
  1. **Forbind skærmen ved din Master (Centralstationen)**  
     * Skærmen på dit STP-kabel skal kun forbindes til stel ét enkelt sted, og det mest logiske sted på modelbanen er ved din hovedcentral (f.eks. Central Station eller boosteren, hvor bussen starter).
     * ***Ved centralen:*** Forbind skærmen til centralens jord/stel-terminal (hvis den findes) eller til den fælles DCC-retur/stel (brun/sort alt efter system).
     * ***Ved modulerne (skifter, tilbagemelding, controllere):*** Klip skærmen ren over, og isoler den med tape eller krympeflex. Den må ***ikke*** røre noget ved de moduler, der sidder rundt om på banen
  2. **Brug de rigtige ledere (Vigtigt for modelbaner)**
     * Da modelbaner ofte bruger standardiserede stik (f.eks. RJ45-netværkskabler til Märklin-bus/CAN), skal du være opmærksom på, at du ***aldrig må bruge skærmen som minus/returleder.***
     * Brug kablets indre ledere således:
       * **Snoet par 1:** CAN_H og CAN_L (De skal ligge i samme snoede par for at ophæve støj).
       * **En separat leder:** CAN_GND (0V / Reference). Den skal forbindes hele vejen igennem til alle moduler, så de har samme spændingsreference.
       * **Metalskærmen:** Holdes udenfor og jordes kun ved centralen som beskrevet ovenfor.
     * **Hvorfor er det vigtigt på en modelbane?**
       * DCC-signalet i skinnerne er reelt et kraftigt firkant-vekselspændingssignal, som udstråler radiostøj. Hvis du forbinder skærmen i begge ender under modelbanen, vil de magnetiske felter fra DCC-skinnerne induces direkte i skærmen og skabe en stelsløjfe. Det vil faktisk forvandle dit beskyttende STP-kabel til en antenne, der trækker støj ind i CAN-bussen, hvilket kan føre til mystiske fejl, hvor sporskifter skifter af sig selv, eller tilbagemeldinger svigter.
       * Ved kun at lade skærmen røre stel ved centralen, fungerer den som et effektivt skjold, der opfanger støjen og leder den væk, inden den når ind til de følsomme dataledninger.

## Skriv eksempler på ben nummer

* Her er et konkret eksempel på, hvordan du konfigurerer benforbindelserne (pinout), hvis du bruger et standard RJ45-stik (netværkskabel) til din CAN-bus på modelbanen.
* Inden for modelbane (især i Märklin-universet og systemer baseret på CiA 303-1 standarden), bruges der typisk to forskellige standarder for RJ45. Her er de to mest almindelige eksempler, så du kan vælge det, der passer til dit udstyr.

### **Eksempel 1: Märklin CAN-bus standard (Mest brugt til modelbaner)**

* Märklin bruger RJ45-stik til deres terminaler (f.eks. 60125/60145) og bokse. Bemærk, at de også fører banestrøm/forsyning med i kablet.
* Hvis du følger Märklins pinout, ser det således ud (farverne følger standard T568B netværkskabel):

|Ben (Pin)|Signal|Typisk farve (T568B)|Function|
|:---|:---|:---|:---|
|Ben 1|CAN_H|Hvid/Orange|Datasignal (Høj)|
|Ben 2|CAN_L|Orange|Datasignal (Lav)|
|Ben 3|CAN_GND|Hvid/Grøn|Reference-jord til data|
|Ben 4|+12V / Retur|Blå|Forsyning til moduler|
|Ben 5|Bane B|Hvid/Blå|DCC/Märklin Digital (Snoet med ben 4)|
|Ben 6|0V / Retur|Grøn|System-stel|
|Ben 7|Bane 0|Hvid/Brun|DCC/Märklin Digital (Snoet med ben 8)|
|Ben 8|+18V / VCC|Brun|Hovedforsyning|
|Metalkappe|Skærm (Shield)|(Kun forbundet i Centralen)|Støjbeskyttelse|

### Eksempel 2: International Industri-standard (CiA 303-1)

* Hvis dit udstyr følger den rene, officielle CAN-industristandard for RJ45, bruges der færre ben, og signalerne sidder anderledes:

|Ben (Pin)|Signal|Typisk farve (T568B)|Function|
|:---|:---|:---|:---|
|Ben 1|CAN_H|Hvid/Orange|Datasignal (Høj)|
|Ben 2|CAN_L|Orange|Datasignal (Lav)|
|Ben 3|CAN_GND|Hvid/Grøn|Reference-jord til data|
|Ben 7|(Valgfri)|Hvid/Brun|CAN_GND (Nogle systemer bruger ben 7 i stedet for 3)|
|Metalkappe|Skærm (Shield)|(Kun forbundet i Centralen)|Støjbeskyttelse|

#### T568B Farve-identifikation (Når du kigger på stikket)

* Når du holder RJ45-stikket med guldlåsen (klipsen) vendt nedad og kablet ind mod dig selv, tælles benene fra venstre mod højre (Ben 1 til venstre, Ben 8 til højre).
* For at sikre, at vi rammer helt plet, kan du fortælle mig:
  * Hvilket specifikt mærke eller system (f.eks. Märklin, ESU, Uhlenbrock, DIY Arduino) bruger du?
  * Skal du selv klemme stikkene på kablet, eller bruger du færdige kabler, hvor du modificerer skærmen i den ene ende?

## Jeg har fået dette forslag for modbus

* Jeg har fået dette forslag for modbus
  * ben 1 & 2 VCC 12V
  * ben 3 not used
  * ben 4 data -
  * ben 5 data +
  * ben 6 gnd
  * ben 7 gnd
  * ben 8 gnd

* Dette forslag giver rigtig god mening elektronisk og følger et gennemtænkt princip. Læg mærke til, at du i dette layout faktisk ikke bruger ben 3 og 6 som et datapar. I stedet ligger dine data på ben 4 og ben 5.
* I et standard netværkskabel (T568B) er det det blå par, der sidder på netop ben 4 og 5. Da de to ledninger er snoet tæt om hinanden inde i kablet, udgør de det perfekte datapar til Modbus, som effektivt afviser højfrekvent støj fra dine DCC-skinner.
* Her er det komplette overblik over dit specifikke layout:

|Ben (Pin)|Signal|Typisk farve (T568B)|Function|
|:---|:---|:---|:---|
|Ben 1|VCC 12V|Hvid/Orange|Strømforsyning|
|Ben 2|VCC 12V|Orange|Strømforsyning (snoet med ben 1)|
|Ben 3|Not used|Hvid/Grøn|Bruges ikke|
|Ben 4|Data - (B)|Blå|Modbus data (Snoet par med ben 5)|
|Ben 5|Data + (A)|Hvid/Blå|Modbus data (Snoet par med ben 4)|
|Ben 6|GND|Grøn|Fælles reference-jord|
|Ben 7|GND|Hvid/Brun|Fælles reference-jord|
|Ben 8|GND|Brun|Fælles reference-jord|
|Metalkappe|Skærm (Shield)|(Ingen ben)|Forbindes KUN til stel i én enkelt ende!|

### Hvorfor er dette layout godt til din modelbane?

1. Kraftig jord (GND): Ved at bruge tre ben (6, 7 og 8) til GND sikrer du, at modulerne har en meget stabil og tyk stelforbindelse tilbage til strømforsyningen. Det minimerer spændingsfald under banen.
2. Perfekt snoet par: Data ligger på det blå par (4 og 5), hvilket er præcis, som kablet er designet til at blive brugt.

### Og hvad så med STP-skærmen?

* Selvom layoutet for benene nu er ændret, gælder den oprindelige hovedregel for dit STP-kabel stadigvæk i bedste velgående:
  * I "Master"-enden (ved din PC, interface eller master-styreboks): Forbind kablets ydre metalskærm/folie til systemets centrale jord eller stel.
  * Ude ved alle modulerne: Klip skærmen af, og isoler den, så den svæver frit.
* Hvis du ved en fejl forbinder metalskærmen til ben 6, 7 eller 8 (GND) ude ved de enkelte moduler, skaber du en stelsløjfe, fordi GND allerede føres med indeni kablet. Det vil fange støjen fra DCC-skinnerne og sende den direkte ind i datalinjerne.
* Er de moduler, du bygger eller installerer, færdigkøbte moduler (som f.eks. fra en bestemt modelbaneproducent), eller er det et gør-det-selv (DIY) projekt, hvor du selv lodder printpladerne?
