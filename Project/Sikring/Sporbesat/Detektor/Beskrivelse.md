# Beskrivelse af functionen af Sporbesat detektor

## KiCad Sporbesat DesignBlock

![Sporbesat-Detektor.png](./Images/20261008/Skærmbillede%20fra%202026-10-08%2022-35-19.png)

## Test Board

* Fritzing
  * ![Stripboard_49x18_schem.png](../../../Fritzing/Stripboard_49x18_schem.png)
  * ![Stripboard_49x18_bb.png](../../../Fritzing/Stripboard_49x18_bb.png)
  * [Stripboard_49x18.fzz](../../../Fritzing/Stripboard_49x18.fzz)

## Diagram beskrivelse (rettelse kommer)

## Hvordan anbringes sensoren på anlæget

Sensoren på diagrammet er en af fire på samme print, sensor printet anbringes så tæt på skinneafsnittet som muligt, sammen med et transmisions print som sender data til den centrale enhed via *CANBUS Transmitter* er er en meget støjemun dataforbindelse, det som bruges i moderne biler og industrien.

## DCC

### DCC signal

* ![DCC_Signal_Basic.bmp](./Images/DCC_Signal_Basic.bmp)
* Vi 2V per division og propen er i x10 så det giver 20V per division.
  * så vi ser her et signal som ca. svinger mellem +20V til -20V
* Vi ser 50µSec. per division 
  * Så vi ser et signal såm svinger melllem 100µSec til 200µSec per periode.
