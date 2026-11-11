# Afkoblings kondensator

Til afkobling (decoupling) på dine printkort i SMD 0805 skal du bruge keramiske kondensatorer (MLCC) med et X7R dielektrikum (eller X5R) på 100 nF (0,1 µF) og en mærkespænding på 25V eller 50V.
Her er de præcise specifikationer og årsagerne til valget:

## 1. Dielektrikum: Vælg X7R (Undgå Y5V og Z5U)

* X7R: Dette er standarden til stabil afkobling. Den ændrer næsten ikke kapacitet ved temperaturudsving (den klarer fra -55°C til +125°C med maksimalt ±15% udsving).
* Undgå Y5V / Z5U: Disse billigere typer kan miste op til 80% af deres kapacitet, når der lægges spænding på dem (DC-bias) eller ved temperaturændringer.

## 2. Mærkespænding: Vælg 25V eller 50V

* Selvom du kun kører 3,3V og 12V på dit print, bør du vælge en 25V eller 50V kondensator.
* Keramiske kondensatorer mister effektiv kapacitet, jo tættere man kommer på deres maksimale spændingsgrænse. En 50V kondensator på en 12V-linje bevarer næsten sin fulde 100 nF kapacitet, og i 0805-størrelse koster 50V det samme som 16V.

## 3. Værdier og placering på dit printkort

* 100 nF (0,1 µF) – Hovedafkobling: Sæt én helt tæt på forsyningsbenene (VDD og GND) på hver eneste IC på printet (ESP32-S3, MCP23017, CAN-transceiver og ULN2803A).
* 10 µF (SMD 0805 eller 1206, X5R/X7R, 25V) – Bulk-afkobling: Sæt én eller to af disse ved strømindgangen på dit printkort (både på 12V-linjen og efter din 3,3V stepdown-konverter) for at udjævne større strømtræk, når Tortoise-motorerne skifter.
* 22 nF (SMD 0805, X7R eller C0G/NP0, 50V) – Til din DCC-føler: Til tidsbestemmelsen i din peak-detektor (sammen med 100 kΩ modstanden) er X7R også helt fint, men hvis du vil have ekstrem præcision, kan du vælge C0G (NP0), som slet ikke driver i værdi.

## Hurtig indkøbsliste (SMD 0805)

Når du bestiller komponenter (f.eks. hos Mouser, DigiKey, TME eller LCSC), skal du søge efter:

* 100nF 0805 X7R 50V (10% tolerance) – til al standard afkobling.
* 22nF 0805 X7R 50V (eller C0G) – til dine analoge DCC-følere.
* 10uF 0805 X5R 25V – til ekstra stabilisering af 3,3V- og 12V-forsyningen på hvert print.

Er du i gang med at tegne PCB-layoutet nu (f.eks. i KiCad eller EasyEDA), og vil du have et par hurtige tommelfingerregler til jordplan (ground plane) og linjebredder til dine 12V motorudgange og DCC-spor?
AI-svar kan indeholde fejl. Få flere oplysninger

## har du et mouser nr ?

Her er konkrete Mouser-varenumre på de tre SMD 0805-kondensatorer fra anerkendte producenter (KYOCERA AVX og Murata), som alle er standard lagervarer til billige stykpriser:

|Anvendelse|Værdi & Specifikation|Producent & Varenr.|Mouser Varenr.|
|:---|:---|:---|:---|
|Standard IC-afkobling|100 nF, 50V, X7R, 10%, 0805|KYOCERA AVX 08055C104KAT2A|581-08055C104K|
|DCC-føler (Peak-detektor)|22 nF, 50V, X7R, 10%, 0805|KYOCERA AVX 08055C223KAT2A|581-08055C223K|
|Strømforsyning (Bulk-afkobling)|10 µF, 25V, X5R, 10%, 0805|Murata GRM21BR61E106KA73L|81-GRM21BR61E106KA3L|

## Gode råd ved bestilling hos Mouser

* Køb på "Cut Tape": Når du indtaster Mouser-varenumrene i søgefeltet, skal du blot vælge varianten mærket med Cut Tape for at købe dem i stykvis (f.eks. 50 eller 100 stk. ad gangen for at opnå mængderabat).
* Nyt globalt varenummer: KYOCERA AVX er ved at skifte til et nyt globalt nummersystem (f.eks. KGM21NR71H104KT for 100 nF-udgaven), men begge numre fører dig direkte til den rigtige komponent hos Mouser.
Mangler du også Mouser-varenumre til nogle af de øvrige SMD-komponenter på dit print – f.eks. 330 Ω (1W) pull-up modstande, Schottky-dioderne til DCC-føleren eller CAN-bus transceiver-chippen (SN65HVD230)?
AI kan begå fejl, så sørg for at dobbelttjekke dens svar 
