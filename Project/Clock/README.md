# AI Ur til modelbanen

## AI's vejledning til kode for Ursystem

![2014-42-29.png](./Images/Skærmbillede%20fra%202026-10-15%2014-42-29.png)

1. [AI_Clock.md](./AI_Clock.md)
2. [AI_Clock_med Packages.md](./AI_Clock_med%20Packages.md)
3. [AI_Clock_med_Master.md](./AI_Clock_med_Master.md)
4. [AI Clock med Master og Analog Ur](./AI_Clock_med%20Master_Analog.md)
5. [split i flere ESPHome Package](./split%20i%20flere%20ESPHome%20Package.md)
6. [Lys i huse](./Lys_i_huse.md)

## L298N Dual full-bridge driver

* Kilder: 
  * [DS_l298.pdf](../Datasheet/Semiconductor/L298N/DS_l298.pdf)
  * [L298N Motor Driver.odt](../Datasheet/Semiconductor/L298N/ESP32%20with%20DC%20Motor%20and%20L298N%20Motor%20Driver.odt)

* L298N Motor Driver pinout
  * Let’s take a look at the L298N motor driver pinout and see how it works.

[![2019-46-55.png](./Images/Skærmbillede%20fra%202026-10-15%2019-46-55.png)](https://let-elektronik.dk/l298n-dual-h-bro-motordriver-2-dc-motorer-eller-1-bipolaer-stepper)

* The motor driver has a two-terminal block on each side for each motor. OUT1 and OUT2 at the left and OUT3 and OUT4 at the right.
  * OUT1: DC motor A + terminal
  * OUT2: DC motor A – terminal
  * OUT3: DC motor B + terminal
  * OUT4: DC motor B – terminal
* At the bottom, you have a three-terminal block with +12V, GND, and +5V. The +12V terminal block is used to power up the motors. The +5V terminal is used to power up the L298N chip. However, if the jumper is in place, the chip is powered using the motor’s power supply and you don’t need to supply 5V through the +5V terminal.
* Important: despite the +12V terminal name, you can supply any voltage between 5V and 35V (but 6V to 12V is the recommended range).
* Note: if you supply more than 12V, you need to remove the jumper and supply 5V to the +5V terminal.
* Jumper:
  *jumper in place – uses the motor power supply to power up the chip. Jumper removed: you need to prov

* Control terminals:
  * At the bottom right you have four input pins and two enable terminals. The input pins are used to control the direction of your DC motors, and the enable pins are used to control the speed of each motor.
    * IN1: Input 1 for Motor A
    * IN2: Input 2 for Motor A
    * IN3: Input 1 for Motor B
    * IN4: Input 2 for Motor B
    * EN1: Enable pin for Motor A
    * EN2: Enable pin for Motor B
* There are jumper caps on the enable pins by default. You need to remove those jumper caps to control the speed of your motors. Otherwise, they will either be stopped or spinning at the maximum speed.

### Control DC motors with the L298N Motor Driver

Now that you’re familiar with the L298N Motor Driver, let’s see how to use it to control your DC motors.

#### Enable pins

* The enable pins are like an ON and OFF switch for your motors. For example:
  * If you send a HIGH signal to the enable 1 pin, motor A is ready to be controlled and at the maximum speed;
  * If you send a LOW signal to the enable 1 pin, motor A turns off;
  * If you send a PWM signal, you can control the speed of the motor. The motor speed is proportional to the duty cycle. However, note that for small duty cycles, the motors might not spin, and make a continuous buzz sound.

|SIGNAL ON THE ENABLE PIN|MOTOR STATE|
|:---|:---|
|HIGH|Motor enabled|
|LOW|Motor not enabled|
|PWM|Motor enabled: speed proportional to the duty cycle|

#### Input pins

The input pins control the direction the motors are spinning. Input 1 and input 2 control motor A, and input 3 and 4 control motor B.

* If you apply LOW to input1 and HIGH to input 2, the motor will spin forward;
* If you apply power the other way around: HIGH to input 1 and LOW to input 2, the motor will rotate backwards. Motor B can be controlled using the same method but applying HIGH or LOW to input 3 and input 4.

#### For example, for motor A, this is the logic

|Direction|Input 1|Input 2|Enable 1|Ur|
|:---|:---|:---|:---|:---|
|Forward|0|1|1|Lige minut tal|
|Backwards|1|0|1|Ulige minut tal|
|Stop|0|0|0||

#### Enable pins

The enable pins are like an ON and OFF switch for your motors. For example:

* If you send a HIGH signal to the enable 1 pin, motor A is ready to be controlled and at the maximum speed;
* If you send a LOW signal to the enable 1 pin, motor A turns off;
* If you send a PWM signal, you can control the speed of the motor. The motor speed is proportional to the duty cycle. However, note that for small duty cycles, the motors might not spin, and make a continuous buzz sound.

#### Wiring ESP32, Power & L298N Ur Driver

|LN298N Motor Driver|Jumper|12V|GND|5V|Input 1|Input 2|Enable|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|Power|Fjernet|12/24V|GND||||
|ESP32|||GND|5V|GPIO16|GPIO17|5V|
