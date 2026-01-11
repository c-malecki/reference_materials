https://www.makerfabs.com/dvr8833-2-channel-dc-motor-driver-module.html?srsltid=AfmBOoqLgUmnR39OBx6OafKk9gxyiSVOT5hQjS42Is4mCadIEyW9JnRH

https://www.amazon.com/gp/customer-reviews/R141LAF6JR4F56/ref=cm_cr_dp_d_rvw_ttl?ie=UTF8

https://www.instructables.com/ESP32-and-DC-Motors-Tutorial-Part-1/

 the inputs on the driver board will accept either 5v or 3.3v logic. But, that PWM signal is required. Each board can control 2 motors.
 
 Pins AN1 and AN2 control driving 1 motor connected to AO1 and AO2.
 Pins BN1 and BN2 control the motor connected to BO1 and BO2.
 
 The ground from the board providing the PWM signal must be connected to the ground pin of the driver board. To drive the motor forward, set AN1 low and provide a PWM signal on AN2. To drive it backward, set AN2 low and provide the provide the PWM signal on AN1. Stop motor by driving AN1 and AN2 low. The same applies to the motor connected to the B outputs and signals provided on the B inputs. Note that on the Arduino, only certain pins support PWM. You'll want to check the docs for your board to make sure to use the right pins for your model. On mine, I used:

#define AN1 3
#define AN2 9
#define BN1 10
#define BN2 11
#define STDBY 2
Initialized in setup() with:
pinMode(AN1, OUTPUT);
pinMode(AN2, OUTPUT);
pinMode(BN1, OUTPUT);
pinMode(BN2, OUTPUT);
pinMode(STDBY, OUTPUT);
Enable the board with:
digitalWrite(STDBY, HIGH);
Control the motors like this (for example):
//Accelerate motor A forward
digitalWrite(AN1, LOW);
for(int i=0;i<255;i++) {
analogWrite(AN2, i);
delay(20);
}
Provide the Motor Voltage to the VM pin on the controller board, do not try to use the 5v from the Arduino.
My breadboard looks a mess, but I'm including a picture at the beginning of my video.

Pin description:
ANI1: logic input control port of AO1, the level is 0-5V.
AIN2: logic input control port of AO2, the level is 0-5V.
BNI1: logic input control port of BO1, the level is 0-5V.
BIN2: logic input control port of BO2, the level is 0-5V.
AO1 and AO2 are 1 H-bridge output ports, which are connected to two pins of a DC motor.
BO1 and BO2 are two H-bridge output ports, connected to the two pins of another external direct motor.
GND: Ground.
VM: chip and motor power supply pin, voltage range 2.7 V – 10.8 V.
STBY: Ground or floating chip does not work, no output, connect to 5V to work; level 0-5V.
NC: Empty feet
Basically compatible with TB6612 module pins.

DRV8833 usage:
DRV8833 is dual drive, which can drive two motors
The following are the IO ports that control the two motors.
The STBY port is connected to the IO port of the microcontroller to clear the motor.
VM connected to power within 12V

Drive 1 way
Truth Table:
AIN1 0 0 1
AIN2 0 1 0
Stop Reverse Forward
A01, AO2 is connected to the two pins of motor 1.

Drive 2 way
PWMB is connected to the PWM port of the microcontroller
Truth Table: BIN1 0 0 1
BIN2 0 1 0
Stop Reverse Forward
B01, BO2 is connected to the two feet of motor 2.