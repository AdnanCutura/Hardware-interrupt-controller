

# Hardware interrupt controller

Triggering hardware interrupts with `rotary encoder` and creating `Interrupt Service Routines` for handling these interrupts which reflects in rotating the `DC Motor` in certain direction.

## Components

### Microcontroller board STM32F103C8T6

**Microcontroller board** used in this project is `STM32F103C8T6` made by [ST-Electronics](https://www.st.com/content/st_com/en.html) also known as Blue Pill. The microcontroller unit, found in the board is `ARM Cortext-M3` 32 bit RISC core. The MCU operates at 72 MHz frequency. On 53mm long and 23mm wide board, 48 pins are placed. They are grouped into header 1 and header 2 (each with 20 pins), SWD header, and boot pins (each with 4 pins).



<img src="https://i.imgur.com/P0BVDq1.png" alt="image-20200507221845059" style="zoom:50%;" />

### Rotary encoder KY-040

The **rotary encoder** is used to create the actual hardware interrupt. In this project the chosen rotary encoder is `KY-040`. The primary function is to convert the angular position (rotation) of a knob into an output signal. It is the incremental type encoder, which reads changes in angular displacements using a slotted disk.

The chosen encoder has a ground pin (`GND`), pin for power supply (`VCC`), and three output pins. Output pins are:

- `CLK ` (clock) – the primary output pin
- `DT ` (deadtime) - the secondary output pin
- `SW`(switch) – the push button

With those output pins its possible to create two hardware interrupts. 



![image-20200507222249723](https://i.imgur.com/HonzZRz.png)

### DC Motor

The actuator, used in this project is the **DC Motor**. It converts the electrical energy into rotary electromechanical energy with the interaction of magnetic fields. 

![image-20200507222548752](https://i.imgur.com/7CRSTrC.png)

### Motor Driver L298N

DC Motor is directly connected to the **Motor driver**. The Driver is used to controlling the speed (using PWM) and the rotation direction (using H-bridge). We used the `L298N` Motor driver in this project. This driver is dual-channel, so it’s capable of controlling two DC motors simultaneously and individually.

![image-20200507222411732](https://i.imgur.com/UIUvFIZ.png)

Motor speed is controlled with `ENA` for first and `ENB` pin for second DC motor. If those pins are pulled HIGH the motor will rotate, LOW means the motor is idle. The actual speed could be controlled by sending PWM signals to those pins.

`IN1 `and `IN2 ` (plus `IN3` and `IN4` for second motor) are control pins, used for controlling the DC motor's direction. The motor will rotate according to the H-bridge truth table.

**Table 1** H-bridge truth table

| **IN1** | **IN2** | **Spinning  direction** |
| ------- | ------- | ----------------------- |
| Low     | High    | Backward                |
| High    | Low     | Forward                 |
| Low     | Low     | Off                     |
| High    | High    | Not possible            |

**Connection to motors** is possible with 4 output pins (`OUT1`, `OUT2`, `OUT3`, and `OUT4`). The rest of the pins are used for power. They are `VCC`, which supplies the motor with **power**, could be between 5 and 35V, `5V VS `which supplies switching logic circuitry with power and the **ground** pin (`GND`).

----

## Connection scheme

Rotary encoder and motor driver are directly connected to the microcontroller board. Rotary encoder `KY-040` main pins are connected to the board GPIO pins according to **Table 2** Encoder’s ground and voltage pin are also connected to the compatible 3.3V and ground pins on the board.

**Table 2** Rotary encoder and Blue Pill pin connections

| **KY-040 pin** | **STM32F103C8T6  pin** |
| -------------- | ---------------------- |
| CLK            | PA5                    |
| DT             | PA6                    |
| SW             | PA7                    |

 

`L298N ` pins are connected to the `STM32F103C8 `GPIO pins according to **Table 3**, OUT1 and OUT2 pins are connected to the DC Motor and two pins for power supply VCC and GND are connected to the 12V adapter.

**Table 3** Motor driver and Blue Pill pin connections

| **L298N pin** | **STM32F103C8T6  pin** |
| ------------- | ---------------------- |
| ENA           | PA3                    |
| IN1           | PA2                    |
| IN2           | PA1                    |



![image-20200507224840829](https://i.imgur.com/vnrChhC.png)

## Hardware interrupts

**The primary interrupt** is initiated with the CLK pin and its activated when the knob is rotated. The secondary output pin „DT“ is used to determine the rotating direction (clockwise or counterclockwise).

The direction is determined simply by checking the output signal B if it's equal to 1 (logical HIGH) in the moment of the interrupt (when output signal A is starting to send the logical HIGH signal), the position is counterclockwise, otherwise, if B is equal to 0 (logical LOW) it is counterclockwise direction.  

![Arduino UNO Tutorial 6 - Rotary Encoder](https://i.imgur.com/cUMDOA6.jpg)

**The secondary hardware interrupt** is initiated by pushing the knob like a button. If the button is pushed the logical HIGH is sent via output pin SW, and certain interrupt service routine (ISR) is triggered. In the idle period, the SW pin is outputting the logical LOW


## Implementation

This project is developed via software toolchain, which consists of [STM32 Cube MX](https://www.st.com/en/development-tools/stm32cubemx.html) and [Keil uVision5](https://www.keil.com/demo/eval/arm.htm)

The **STM32 Cube MX** is used to create boilerplate for further programming in IDE **Keil uVision5** using the programming language **C**.

Creating boilerplate considering setting up some core functionalities. That includes clock speed, timers, and GPIO pins configuration, setting up middleware, and so on. In our project, we configured mentioned GPIO pins to be input, output, or external interrupt. Clock speed is set to **8MHz** using `HSI RC`.

<img src="https://i.imgur.com/HEvs5Kc.png" alt="image-20200507230300989" style="zoom:67%;" />

The external interrupt (EXTI) pins are encoder’s CLK and SW pin. CLK is in the mode with Rising edge trigger detection and SW is in the mode with Rising/Falling edge detection. The input pin is also encoder’s DT pin and the output pins are pins that are connected to the Motor driver, IN1, IN2, and ENA.

Once we generated code from STM32 Cube MX, we can open it via Keil uVision5. 

The additional code we need is to program what should happen when the interrupt is triggered. For that needs, we have integrated function `void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)` which is *_weak* type and we can override it. The function parameter is the pin where the interrupt is triggered, and it’s called every time it happens. We can also track the motor position with a volatile integer variable.
