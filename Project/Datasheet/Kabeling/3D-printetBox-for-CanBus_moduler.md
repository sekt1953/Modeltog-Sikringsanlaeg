# 3D printet Box for CanBus moduler

## hvordan for jeg 3D printet box ledende så den kan beskytte em

* Du kan gøre en 3D-printet boks elektrisk ledende og opnå EMC-afskærmning (elektromagnetisk kompatibilitet) ved at påføre en ledende maling eller belægning på indersiden, eller ved at beklæde den med kobber- eller alufolie.
* Selve ledende filamenter (f.eks. med kulfiber eller grafen) er som regel kun gode nok til ESD (statiskafladning), men yder sjældent tilstrækkelig dæmpning mod reel højfrekvent EMC/EMI.

### Metoder til at gøre boksen ledende

* Ledende maling (EMC / EMI Shielding Paint):
  * Brug specialmaling som f.eks. nikkel- eller sølv/kobber-baseret maling (fx fra MG Chemicals).
  * Mal indersiden af dit 3D-print i et jævnt, dækkende lag. Det blokerer effektivt elektromagnetisk støj.
* Kobber- eller alufolie:
  * Beklæd indersiden med selvklæbende kobber- eller aluminiumstape.
  * Sørg for, at stykkerne overlapper hinanden, så der er elektrisk forbindelse over det hele. Det er en nem og billig løsning til prototyper.
* Jordforbindelse (Husk dette trin!):
  * For at afskærmningen virker mod EMC, skal det ledende lag forbindes til systemets jord (GND). Brug en skrue, en crimp-terminal eller ledende lim/tape til at fastgøre en jordledning direkte til det indvendige ledende lag.
* Hvis du vil vide mere, kan du fortælle mig:
  * Hvilket materiale du vil 3D-printe i (f.eks. PLA, PETG, ABS)
  * Hvilket frekvensområde elektronikken arbejder med eller hvilken type støj, du forsøger at afskærme
* Så kan jeg anbefale den bedste type maling eller fremgangsmåde til dit projekt.

### Jeg vil printe med pla

* Når du printer i PLA, er ledende maling eller kobbertape stadig de absolut mest effektive metoder, da PLA i sig selv er en elektrisk isolator. PLA har dog en lav smeltetemperatur (omkring 60°C), så du skal passe på med visse typer opløsningsmidler i malingen.

#### De bedste metoder til PLA-print

* Nikkel- eller kobberbaseret EMC-maling:
  * Spray eller mal indersiden af boksen med en dertil beregnet afskærmningsmaling. Produkter som akrylbaseret nikkel-spray fra MG Chemicals hæfter rigtig godt på PLA uden at opløse eller deformere plasten.
* Selvklæbende kobbertape:
  * Kobbertape med ledende lim er utrolig effektivt til PLA. Det er billigt, nemt at klippe til og klistrer perfekt på den glatte PLA-overflade. Husk at glatte tapen godt ud, så banerne overlapper og skaber en ubrudt "skal".

#### Vigtige forholdsregler ved PLA

* Undgå stærke opløsningsmidler: Nogle billige spraymalinger indeholder skrappe opløsningsmidler (som acetone), der kan angribe eller blødgøre PLA. Sørg for, at malingen er akrylbaseret eller godkendt til plast.
* Varme under lodning: Hvis du vil lodde en jordledning direkte på kobbertapen, skal du gøre det før du klistrer tapen fast i boksen. PLA smelter ved meget lave temperaturer, og loddekolben vil øjeblikkeligt smelte hul i din boks.
* Mekanisk tolerance: Husk at indbygge ca. 0,2–0,5 mm ekstra luft i dit design, hvis du bruger maling eller tape i samlingerne (f.eks. hvor låget møder boksen), så delene stadig kan lukkes tæt efter behandlingen.

## Hvad hvis jeg bruger PCTG ?

* Hvis du skifter til PCTG, får du en boks, der er langt mere robust, kemikalieresistent og temperaturtolerant end PLA. PCTG minder meget om PETG, men har endnu højere slagstyrke og klarhed, hvilket gør det til et fremragende valg til mekaniske kabinetter.
* I forhold til EMC-afskærmning ændrer PCTG et par vigtige ting til din fordel:

### Fordele ved PCTG til EMC-afskærmning

* Tåler skrappere maling (Opløsningsmidler): Hvor PLA risikerer at deformere eller blive svagt, hvis du bruger spraymaling med stærke opløsningsmidler, er PCTG ekstremt modstandsdygtigt over for kemikalier. Du kan stort set bruge alle typer ledende EMC-spraymaling (nikkel, kobber eller sølv) uden at bekymre dig om, at plasten tager skade.
* Højere temperaturmodstand: PCTG klarer varme op til omkring 75–80°C før det bliver blødt (hvor PLA giver op allerede ved 50–60°C). Det betyder, at boksen bedre tåler den varme, som elektronikken indeni genererer, og afskærmningen holder sig stabil over tid.
* Mekanisk bearbejdning: Hvis du skal montere stik, jordterminaler eller skruer direkte igennem kabinettet for at forbinde dit EMC-lag til jord, flækker PCTG ikke nær så let som PLA, når du borer i det eller spænder skruer til.

### Sådan gør du en PCTG-boks ledende

* Processen er stort set den samme som med PLA, men med færre bekymringer:
  1. Rengøring: PCTG har en meget glat overflade. Vask indersiden med en smule opvaskemiddel eller sprit (hvilket det sagtens tåler) for at fjerne fedtfingre, så malingen eller tapen hæfter optimalt.
  2. Afskærmning: Påfør enten en akryl- eller grafitbaseret ledende maling, eller beklæd indersiden med kobbertape. Takket være PCTG's kemiske resistens hæfter malingen typisk rigtig godt.
  3. Lodning (Stadig pas på): Selvom PCTG tåler mere varme end PLA, smelter det stadig ved mødet med en 350°C varm loddekolbe. Hvis du bruger kobbertape, skal du stadig lodde din jordledning på tapen før du klistrer den fast i boksen.
* For at sikre, at dit EMC-design bliver helt tæt, vil du så have forslag til, hvordan du laver en ledende samling mellem boks og låg, eller vil du have konkrete produktanbefalinger til den ledende maling?