---
folder:     sysc2003
menu:
- header: Introduction
  url:    introduction
- header: Tips & Tricks
  url:    tips-youll-want-to-know
- header: Settings
  url:    common-problems-and-solutions
- header: miniIDE
  url:    miniide
- header: ICC12
  url:    icc12
- header: NoICE
  url:    noice
---

# SYSC 2003

## Introduction

I've noticed that the labs have been getting pretty difficult and most of it isn't from the content. A lot of the difficulty comes from understanding and getting the workflow set up. I've taken the time to figure out common problems with miniIDE, NoICE, and ICC12. These 3 programs are essential to getting working code on the Axiom board. So I've made a list of things you'll need to do in order to get everything working.

You can also checkout the [sample code](axiom-board) to use different parts of the Axiom board. Here's a [review sheet](https://dl.dropboxusercontent.com/u/45208358/review2003.zip) for the exam too.

## Tips You'll want to Know

I'll Update this as I learn more but for now here's a sample on different important topics we covered.

### PWM - Pulse Width Modulation

Essentially, PWM is a way to simulate an analog signal using just digital signals. Lets say Logic 0 is 0V and Logic 1 is 5V and you look a period of 1 second. If you're connected to a motor you can run it at 20% speed by sending it a 1V signal. However, to simulate it theoretically you have to pulse a digital 5V for 200 ms every second.

Register Name | Description
:------------:|:-----------------
PWMPOL        | Controls the "polarity" of the PWM pins, this value determines what logic value the PWM cycle will start at. Usually set to 1.
PWMPERx       | Sets the Period for an entire cycle in terms of the PWM clock cycles.
PWMDTYx       | Controls the "duty cycle" of the pin, this value ranges from 0 - 100 (off to completely on). When the duty cycle is reached the polarity is switched for the remaining time of the period.
PWMCAE        | Controls the Alignment, left aligned if 1, if 0 then it try's to center the section **determined by** the polarity for the **time controlled by** the duty cycle.
PWME          | This is used to enable the pins for use with the current settings in the control registers.

### PORTS and Their Associated Registers

Each Port (ex Port K) has multiple registers associated with it that's used to make them work. There is the **Data Direction** Register (ex DDRK) and the **Data** Register (PORTK).

* The Direction registers tell you whether it's configured as output or input.
* The Data registers tells you what bits are high/low at that port.
  * It it's configured as output this is where you'll set the output to high/low. If it's configured as input this is where you'll read the data from.


The lower 4 bits in PORTK are used for the LED1 - LED4 We want to set the Data Direction Register for port k (DDRK) to `DDRK | 0x0F` -> xxxx 1111 so it knows that the lower 4 bits are going to be written to for output. We also want to set PORTK to `PORTK & 0xF0` -> xxxx 0000 to make sure everything is off to start with.


{% highlight c %}

DDRK |= 0x0F;
PORTK &= 0xF0;

{% endhighlight %}

### How the Components Work

#### LCD

All LCDs have a similar mode of operation, they generally have a way to send **instructions** and a way to send **data**. If you send an instruction it tells the LCD to do something (e.g. clear screen, move cursor). If you send data, it's telling the LCD to display the corresponding ASCII character at the current cursor location. Using the LCD on the Axiom Board can be done using this code.

#### DC Motor / PWM

The DC motor on the Axiom Board is connected to the PWM (Pulse Width Modulation) pin 7.

##### The usual set up (using pin 7 as an example) is:

{% highlight c %}

PWMPOL |= 0x80;   //  sets the Polarity for PWM pin 7 to high.
PWMCAE |= 0x80;   //  sets the alignment to left aligned

PWMPER7 = 100;    //  period set to 100 ticks
PWMDTY7 = 20;     //  duty set to 20%

PWME |= 0x80;     //  enable pin 7

{% endhighlight %}

#### Stepper Motor

The stepper motor works using a bottom and left electromagnet. Essentially you turn these one and off to turn the the motor. I won't show how to derive these values, you'll have to do that yourself if you want to know how it works. However, I've provided the tables you should get.


**CW - Clockwise**

 PT6 | PT5
:---:|:---:
  0  |  0  
  1  |  0  
  1  |  1  
  0  |  1  


**CCW - Counter Clockwise**

 PT6 | PT5
:---:|:---:
  0  |  0  
  0  |  1  
  1  |  1  
  1  |  0  

To setup the stepper motor you have to set bits 5 & 6 high in DDRT so we can change change bits 5 & 6 in PTT which controls the bottom and left magnets in the stepper motor. We also need to set bit 6 hight in DDRP & PTP respectively to enable the stepper motor (stepper enable pin is bit 6 in PTP, DDRP lets us set it). In the following code you'll see this done in the main function and you'll see the tables implemented in the step function.

#### Keypad & 7-Segment Display

U7_EN allows the use of the controller that manipulates the keypad and the 7-segment display.

##### How it Works

![Keypad Port Registers](/assets/projects/{{ page.folder }}/keypad-port-functions.jpg)

If U7_EN is high PTH4 - PTH7 contains the columns of the buttons pressed on the keypad. However, you have to tell it what rows to look at. Set PTP0 - PTP3 high for the particular row you want to read.


##### Example

![Keypad Port Registers](/assets/projects/{{ page.folder }}/keypad-example.jpg)

Here `PTP = 0bxxxx0100` this tells the hardware to read Row 3 on the keypad and store the inputs in the 4 most significant bits in PTH (PTH4 - PTH7). In our example it will read 0b1100xxxx because buttons 3 & 4 are pressed.

##### Note on 7-Segment Display

The hardware is configured so that if U7_EN is enabled you can write a binary coded decimal (BCD) number to the lower bits in PTT to display a number.

#### Output Compare

The core thing to realize here is that there is 16 bit registers for each timer pin (TC0 - TC7) and there is a sole 16 bit timer count register (TCNT) that are compared every clock tick. The hardware changes the logic on the pins with a match occurs depending on the setup of the control registers. You can use optional software interrupts (TFLG1) or use hardware configured interrupts (TIE). Both types of interrupts will occur at the same time, depending of TCNT and TCx. The hardware interrupt only does the action you specify in TCTL1 or TCTL2. You write the ISR for the software interrupt.

Register Name | Description
:------------:|:-----------------
TCTL1 & TCTL2 | There are 2 bits per output compare line. So they are split among to 8 bit registers. Each pair of bits for the ports specify the action to perform when a comparison is found.
TSCR2         | The main function of this is to scale the clock.
TIE           | Hardware Interrupt Enable.
TFLG1         | Software Interrupt Flags (Clear to enable) (clear by setting (don't ask me why) ).
TSCR1         | Bit 7 is timer enable, 4, 5, 6 and other cool things you can checkout.
TCNT          | Number of clock ticks
TCx           | Number of ticks that will trigger interrupt(s) for line x.


## Common Problems and Solutions

### Settings

#### NoICE Settings

![NoICE settings in Target Communications](/assets/projects/{{ page.folder }}/NoICETargetCommunication.jpg)

The most important settings come from NoICE, notice the COM Port is set to COM1 and the Baud Rate is 19200. This is what is required to interface with the HC12 chip.

#### MiniIDE Settings

![MiniIDE Settings](/assets/projects/{{ page.folder }}/miniIDEsetting.jpg)

In MiniIDE make sure the Terminal settings are set to COM1 and 19200 to match  what NoICE is expecting, that way files you build in MiniIDE will work on the board.

#### ICC Settings

![ICC Compiler Settings](/assets/projects/{{ page.folder }}/ICCsettings.jpg)

The ICC Compiler also needs special settings (under the project menu), you want custom device settings, set the PC to **0x4000**, the Data Memory to **0x1000**, and the stack pointer to **0x3DFF**. Make sure all other settings are the same too. Under the compiler tab make sure the output is **Motorola S19** (with source debugging for better debugging info).

### MiniIDE

![Assembly Files](/assets/projects/{{ page.folder }}/asmFiles.jpg)

For the assembly files you write in MiniIDE, in order to include a file you'll need to put it at the end. The reason is you'll need the code *after* the `ORG $4000`.

{% highlight ca65 %}

{% include code_snippets/{{ page.folder }}/labPrototype.asm %}

{% endhighlight %}

After compiling a S19 file with miniIDE you can open it in a texteditor and change the last line from `S9030000FC` to `S9034000BC` and that will automatically set the PC to 4000 for you.

### ICC12

![Make sure to addFiles](/assets/projects/{{ page.folder }}/addFiles.jpg)

For the ICC12 Compiler you'll need to create a new project where the entire path (`M:\SYSC2003\Labs\LabxProj.prj`) has no spaces **(NONE, 0, NO)**. THEN, you'll need to add ALL files you'll need to the project. Including new ones you create as you go.

NOTE: Forgetting to add newly created files is a common mistake I make often.

You'll also need to write code in an old style. Specifically you'll need to declare variables before using them in the functions. Notice the LCD_displayStr function below with `int i;` declared at the top before use in the loop.

{% highlight c %}

{% include code_snippets/{{ page.folder }}/labDemo.c %}

{% endhighlight %}

#### Important note about your .s Files

To declare variables in .s files like .db in .asm files, you have to reserve space in memory and initialize them afterwards

*  .blkb <amount_of_bytes>
*  .blkw <amount_of_words>
* .blkb <amount_of_long_words(4 bytes)
* .ascii "This is used to declare string variables"
* .asciz "This can be used to put escape character in a string \n."
* I believe using a '=' will work in some cases too.
  * SYMBOL=#VALUE


### NoICE

![NoICE Loading File](/assets/projects/{{ page.folder }}/NoICEload.jpg)

Make sure you're loading HEX files otherwise the .S19 files won't appear.

![Setting PC in NoICE](/assets/projects/{{ page.folder }}/NoICEPC.jpg)

Make sure to set the PC to 4000. Also if the program hangs, it's important to close NoICE and then flick the board off/on again before opening NoICE and reloading files to make sure it works properly.

NOTE: If it's a compiled C program one source of error may be not including `asm("swi");` at the end before the return statment. The assembly equivalent is writing `swi` instead of `bgnd` or `end`.