# Projekter til modelbanen

* Sikringsanlæg
  * [Sporbesat](./Sikring/Sporbesat/README.md)
  * CANBUS
    * [CanBus og HomeAssistant](./Project/CANBUS/1_CanBus_og_HomeAssistant.md)
    * [CanBus med Output og svar](./Project/CANBUS/2_CanBus_med%20_Output_og_svar.md)

## Beskrivelse af koden til canbus noder

* [Beskrivelse her](./beskrivelse.md)
* [Til afkobling (decoupling) på printkort](./Afkobling.md)
* [signal styring med canbus](./Signaler.md)

## Komponenter til CANBUS noder

* ESP32-S3-WROOM-1-N16R8
  * Mouser No: 356-ESP32S3WRM1N16R8
  * [ESP32-S3-WROOM-1-N16R8 Datasheet (PDF)](https://www.mouser.dk/datasheet/3/1574/1/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf)
* MCP23017x-x-SO
  * Mouser No: 579-MCP23017T-E/SO
  * [MCP23017T-E/SO Datasheet (PDF)](https://www.mouser.dk/datasheet/3/282/1/MCP23017_Data_Sheet_DS20001952.pdf)
* PCA9685PW
  * Mouser No: 771-PCA9685PWQ900118
  * [PCA9685PW/Q900,118 Datasheet (PDF)](https://www.mouser.dk/datasheet/3/118/1/PCA9685.pdf)
* TBD62083AFNG,EL
  * Mouser No: 757-TBD62083AFWGEL
  * [TBD62083AFWG,EL Datasheet (PDF)](https://www.mouser.dk/datasheet/3/105/1/A548DCD4721AF66713DD4BA46BD13A4F84ED8A6CA2AD31CAA63E6B05F71AA6C7.pdf)
* SN65HVD230
  * Mouser No: 595-SN65HVD230MDREP
  * [https://www.ti.com/lit/gpn/sn65hvd230m-ep](https://www.ti.com/lit/gpn/sn65hvd230m-ep)
* NUP2105L
  * Mouser No: 637-NUP2105L
  * [NUP2105L Datasheet (PDF)](https://diotec.com/request/datasheet/nup2105l.pdf)
* DRV8231ADDAR
  * Mouser No: 595-DRV8231ADDAR
  * [DRV8231ADDAR Datasheet](https://www.ti.com/lit/gpn/drv8231a)
  * Description: Motor/Motion/Ignition Controllers & Drivers 35-V 3.7-A H-bridge motor driver with i
* 08055C104KAT4A (100 nF, 50V, X7R, 10%, 0805)
  * Mouser No: 581-08055C104KAT4A
  * [08055C104KAT4A Datasheet (PDF)](https://www.mouser.dk/datasheet/3/5926/1/KGM_X7R.pdf)
  * Description: Multilayer Ceramic Capacitors MLCC - SMD/SMT KGM21NR71H104KM NEW GLOBAL PN 50V 0.1uF A
* 08055C223KAZ4A (22 nF, 50V, X7R, 10%, 0805, DCC-føler (Peak-detektor))
  * Mouser No: 581-08055C223KAZ4A
  * [08055C223KAZ4A Datasheet (PDF)](https://www.mouser.dk/datasheet/3/5926/1/X7RDielectric.pdf)
  * Description: Multilayer Ceramic Capacitors MLCC - SMD/SMT KGF21BR71H223KM NW G LOB PN 50V .022uF X7 A
* GRM21BR61E106KA73L (10 µF, 25V, X5R, 10%, 0805, Strømforsyning (Bulk-afkobling))
  * Mouser No: 81-GRM21BR61E106KA3L
  * [GRM21BR61E106KA73L Datasheet (PDF)](https://www.mouser.dk/datasheet/3/76/1/GRM21BR61E106KA73-01A.pdf)
  * Description: Multilayer Ceramic Capacitors MLCC - SMD/SMT 10 uF 25 VDC 10% 0805 X5R

### Sporbesat sensor pcb

* Kabel
  * Varenummer:	339005025
  * Varenavn:	LIYCY 5x2,5 mm gray
  * Link: https://jmvcables.dk/shop/pvc-styre-og-signalkabler/liycy/liycy-5x25/
  * Anvendelse: LIYCY er et skærmet signalkabel, der anvendes hvor der kræves en fleksibel kabelforbindelse til fx kontroludstyr.
  * Kontakt
    * JMV Cables A/S
    * Skalstrupgaard 1, 4621 Gadstrup
    * Telefon: +45 46 76 14 14
    * E-mail: info@jmvcables.dk
* S5AC-13-F Description: Rectifiers 50V 5A
  * Mouser No: 621-S5AC-F
  * [S5AC-13-F Datasheet (PDF)](https://www.mouser.dk/datasheet/3/175/1/ds16007.pdf)
* SK13
  * Mouser No: 637-SK13
  * [SK13 Datasheet (PDF)](https://diotec.com/request/datasheet/sk12.pdf)
  * Description: Schottky Diodes & Rectifiers Schottky, SMA, 30V, 1A
* BC857
  * Mouser No: 583-C857
  * Datasheet:  [BC857 Datasheet (PDF)](https://www.mouser.dk/datasheet/3/287/1/bc856-bc858.pdf)
  * Description: Bipolar Transistors - BJT SOT23,PNP,0.1A,45V,L owNoise
* BC847B
  * Mouser No: 637-BC847B
  * Datasheet:  [BC847B Datasheet (PDF)](https://diotec.com/request/datasheet/bc846.pdf)
  * Description: Bipolar Transistors - BJT BJT, SOT-23, 45V, 100mA, NPN
* ILQ621
  * Mouser No: 782-ILQ621
  * Datasheet: [ILQ621 Datasheet](https://www.vishay.com/doc?83654)
  * Description: Transistor Output Optocouplers Phototransistor Out Quad CTR >50%
* C1206C223KMRECTU
  * Mouser No: 80-C1206C223KMRECTU
  * Datasheet: [C1206C223KMRECTU Datasheet (PDF)](https://www.mouser.dk/datasheet/3/72/1/KEM_C1090_X7R_ESD.pdf)
  * Description: Multilayer Ceramic Capacitors MLCC - SMD/SMT 63V 0.022uF X7R 1206 10%

## Modeltid ur er flyttet til her

* [Modeltidsur](https://github.com/sekt1953/Modeltog-Ursystem)

