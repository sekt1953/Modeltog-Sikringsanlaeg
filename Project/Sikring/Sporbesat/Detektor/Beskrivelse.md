# Beskrivelse af functionen af Sporbesat detektor

 ![Sporbesat-Detektor.png](./Images/Sporbesat-Detektor.png)

## Diagram beskrivelse

* **J1:** Skrueterminal hvor DCC Power tilsluttes
* **J2:** Skrueterminal hvor Skinne tilsluttes DCC via sensor
* **D1 & D2:** To 5A Dioder i antiparallel, herigennem leveres hovedparten af strømmen til skinne afsnittet.
* **Q1** Emitter/Basis og R1 bruges til at måle om der vogne på skinnederne:
  * **R1** skal begrænse strømmen i Basis som på ingen måde må over stige 5mA.
  * Er der belastning på skinnederne Leder Q1 Strøm fra Emiter til Collector, i ca 50-100 µSec. det er den tid DCC signalet er Høj, og videre til Optokobleren, som så for NPN Transistoren i optokobleren til at gå Lav (On).
  * Derefter går der ca. 50-100 µSec. med afbrudt forbindelse mellem Emiter og Collector, og derved slukker NPN transistoren i Optokobleren signalet (Off).
  * Denne pulsering vil fortsætte så længe der er vogne på skinnederne, det er ikke helt det vi ønsker, vil gerne have et stabilt signal på udgangen (TP3), der for infører vi en kondensator:
  * **C1** skal holde TP2 (Høj) i de ca. 20 µSec. DCC signalet er (Lav) tilstand, men ikke i for lang tid, da vi gerne skal kunne frasorterer støjpulser.
  * **TP3:** her forbindes sensor til CANBUS Transmitter.
* **CANBUS Transmitter** skal programeres til kun at reagerer på signaler længere end 50 mSec. for at skifte til at vise at afsnittet er besat, på samme måde skal den ikke skifte til at vise afsnittet frit før der manglet signal i mere end 1-2 Sec.
* ***NB!***
  * jeg har ikke helt lagt mig fast på værdierne af R1, R2 & C1, det kommer and på test der vil blive udført på køredage.

## Hvordan anbringes sensoren på anlæget

Sensoren på diagrammet er en af fire på samme print, sensor printet anbringes så tæt på skinneafsnittet som muligt, sammen med et transmisions print som sender data til den centrale enhed via *CANBUS Transmitter* er er en meget støjemun dataforbindelse, det som bruges i moderne biler og industrien.

## DCC

* Basic signal uden belastning
  * ![DCC_Signal_Basic.bmp](./Images/DCC_Signal_Basic.bmp)
  * Vi 2V per division og propen er i x10 så det giver 20V per division.
    * så vi ser her et signal som ca. svinger mellem +20V til -20V
  * Vi ser 50µSec. per division 
    * Så vi ser et signal såm svinger melllem 100µSec til 200µSec per periode.
