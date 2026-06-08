# NISH-Internship-Learning-Repository
Hands-on exploration of MSP430FR4133 with error tracking, fixes, and implementation notes.
A beginner-level documentation of everything I learned, every mistake I made, and every fix I found while programming the MSP430FR4133 microcontroller from absolute scratch.

Table of Contents

About This Repo
Hardware & Software
Exercises Completed
Errors & Solutions

1. CCS Installation & First Project
2. Basic C Syntax Errors
3. Watchdog Timer Confusion
4. Toggle LEDs with Switch
5. LCD Display Struggles
6. LCD Character Issues
7. Interrupt Register Order
8. Potentiometer + ADC Wrong Range
9. Ultrasonic Sensor on LCD

    About This Repo
This is my personal learning log for MSP430FR4133 embedded programming. I started with zero embedded experience and documented every real problem I encountered — not cleaned-up textbook errors, but the actual confusing situations I hit and had to figure out.
Each section describes:

What happened — the actual symptom I saw
What I tried — things that didn't work
What fixed it — the actual solution
What I learned — the concept behind it

## Hardware & Software

| Item | Detail |
|------|--------|
| **Board** | MSP-EXP430FR4133 LaunchPad |
| **MCU** | MSP430FR4133 — 16MHz, 16KB FRAM, built-in LCD |
| **IDE** | Code Composer Studio Theia |
| **Language** | C |
| **External sensors used** | HC-SR04 Ultrasonic, Potentiometer |



##  Exercises Completed

| # | Exercise | Concepts | Status |
|---|----------|----------|--------|
| 1 | Blink red LED | GPIO output, delay, while loop | ✅ Done |
| 2 | Alternate red & green LEDs | Multiple GPIO, toggle operator | ✅ Done |
| 3 | Button toggle between LEDs | GPIO input, pull-up, active-low | ✅ Done |
| 4 | LCD display on button press | hal_LCD driver, state tracking | ✅ Done |
| 5 | Potentiometer reading on LCD | ADC setup, 12-bit conversion, digit extraction | ✅ Done |
| 6 | Ultrasonic distance on LCD | Timer, clock config, external sensor | ✅ Done (minor display issue ongoing) |


##  Errors & Solutions

### Error 1: 🖥 CCS Installation & First Project

**What happened:**

Before writing a single line of code, getting CCS set up and creating the first project was confusing.

The biggest confusion — after selecting an empty project template there is a loading buffer screen. I kept waiting for it to finish, but the correct action is to click **Import immediately** without waiting.

Also, most tutorials online show **CCS Eclipse** (older version). I had **CCS Theia** (newer), which looks completely different. The correct menu path in Theia is:

```
File → Create New Project...
```

Not `File → New → CCS Project` as shown in most tutorials.

**What I learned:**
- Don't wait on the buffer screen — click Import
- CCS Theia UI is different from CCS Eclipse — online tutorials may not match your screen

 ## Error 2: Basic C Syntax Mistakes

When I started writing programs for the MSP430, most of my errors were not related to the hardware but to basic C syntax. Since I was new to embedded programming, I frequently encountered compilation errors caused by small mistakes.

One common mistake was forgetting to add the `#` symbol before the `include` statement.

```c
// Wrong
include <msp430.h>

// Correct
#include <msp430.h>
```

I also learned that C is case-sensitive. Several times I wrote register names and bit definitions in lowercase, which resulted in undefined identifier errors during compilation.

```c
// Wrong
p1dir |= bit0;
wdtctl = wdtpw | wdthold;

// Correct
P1DIR |= BIT0;
WDTCTL = WDTPW | WDTHOLD;
```

Another frequent mistake was forgetting semicolons at the end of statements. Even though the error was simple, the compiler messages were sometimes confusing for a beginner.

```c
// Wrong
P1DIR |= BIT0
P1OUT |= BIT0

// Correct
P1DIR |= BIT0;
P1OUT |= BIT0;
```

### What I Learned

* C is case-sensitive, so register names and predefined constants must be written exactly as specified.
* Every statement should end with a semicolon.
* Preprocessor directives such as `#include` must always begin with `#`.
* Compiler error messages are useful and should be read carefully instead of immediately changing random parts of the code.

  ### Error 3: ⏱ Watchdog Timer Confusion

**What happened:**

Ran the blink LED code for the first time. The debugger started and a **yellow arrow appeared on line 5** — I thought this was an error. It's not. It means the debugger is paused at that line, waiting. I couldn't find the Continue button for a while.

Separately, when I forgot the watchdog line, the LED did absolutely nothing — no error, no warning. The chip was silently resetting every 32ms.

**The fix — first two lines of every program:**

```c
void main(void)
{
    WDTCTL = WDTPW | WDTHOLD;   // stop watchdog timer — ALWAYS line 1
    PM5CTL0 &= ~LOCKLPM5;       // unlock GPIO pins  — ALWAYS line 2
    ...
}
```
## Error 4: LCD Interface and Display Issues

Working with the LCD was one of the most challenging parts of my MSP430FR4133 learning process. I encountered several different issues before I was able to display characters reliably.

### Problem 1: Undefined LCD Registers and Macros

Initially, I referred to examples and code snippets written for other MSP430 devices. When I tried compiling them on the MSP430FR4133, the compiler reported multiple undefined identifiers such as:

```c
P5SEL1
LCDMUX_3
VLCD_30
LCDCLR
LCDPRE_5
```

At first, I assumed that some library files were missing from my project. After checking the device documentation and header files, I realized that these definitions belong to different MSP430 variants and are not available on the FR4133.

To solve this, I referred to the MSP430FR4133 device-specific examples and used the LCD driver files provided for the FR4133. This helped me understand that peripheral registers and macros can vary significantly between MSP430 families.

The LCD functions used in my project were:

```c
Init_LCD();
clearLCD();
showChar('A', pos1);
```

### Problem 2: LCD Digits Appearing Faded

After getting the LCD operational, I attempted to display the numbers 1 to 6 when a push button was pressed.

The display behaved unexpectedly. Some digits appeared faint, and the final digit was either very dim or not visible at all.

Initially, I suspected an LCD hardware issue. After checking the code more carefully, I found that `clearLCD()` was being executed continuously inside the main loop. The display was being cleared and redrawn repeatedly, causing the digits to appear unstable.

At the same time, my state-tracking logic was incomplete because the `lastState` variable was not being updated correctly when the button was released. As a result, the display update section continued to execute unnecessarily.

I corrected the logic by updating the LCD only when the button state changed and by properly handling the `lastState` variable for both pressed and released states.

```c
unsigned char lastState = 0;

while(1)
{
    if (!(P1IN & BIT2))
    {
        if (lastState != 1)
        {
            clearLCD();

            showChar('1', pos1);
            showChar('2', pos2);
            showChar('3', pos3);
            showChar('4', pos4);
            showChar('5', pos5);
            showChar('6', pos6);

            lastState = 1;
        }
    }
    else
    {
        if (lastState != 0)
        {
            clearLCD();
            lastState = 0;
        }
    }
}
```

After implementing these changes, all digits became visible and the display remained stable.

### What I Learned

* Code examples from one MSP430 device may not work directly on another MSP430 device.
* It is important to verify register names and macro definitions using the device-specific documentation and header files.
* Repeatedly clearing and redrawing an LCD can create display issues.
* State variables help prevent unnecessary updates and improve program behaviour.
* When debugging display problems, both hardware and software causes should be investigated before drawing conclusions.
## Error 5: Potentiometer and ADC Configuration

As part of learning analog inputs, I connected a potentiometer to the MSP430FR4133 and displayed the ADC reading on the LCD.

The program compiled and ran successfully, but the ADC values did not cover the expected range. Initially, I suspected a problem with the potentiometer wiring or the LCD display code.

While checking the ADC configuration, I discovered that I was using settings taken from examples written for a different MSP430 device. Some register names and configuration values were not valid for the FR4133.

One of the issues was related to ADC resolution configuration. After correcting the ADC setup and reviewing the device-specific documentation, the readings started behaving as expected.

I also encountered a compilation error while configuring the analog input pin because I attempted to use `P1SEL1`, which is not available on the MSP430FR4133. Referring to the device header file helped identify the correct pin configuration method.

### What I Learned

* Always verify peripheral configuration using the device-specific documentation.
* Register names and configuration options may differ between MSP430 devices.
* If ADC readings appear incorrect, both hardware connections and ADC settings should be checked.
* Compiler errors often provide useful clues when an incorrect register or configuration is being used.


## Error 6: HC-SR04 Ultrasonic Sensor Interfacing

The ultrasonic sensor was my first attempt at interfacing an external sensor with the MSP430FR4133. While measuring distance and displaying the result on the LCD, I encountered several issues.

Initially, the distance readings were inconsistent and sometimes completely incorrect. During debugging, I reviewed the timer configuration and clock settings because the distance calculation depends on accurate timing.

Even after making changes, the readings still did not make sense. After spending considerable time checking the software, I discovered that the actual problem was the hardware connection. The TRIG and ECHO wires were connected to different pins than those defined in the program.

Because the ECHO pin was effectively floating, it appeared to remain HIGH and produced unreliable measurements. Once the wiring was corrected to match the code, the sensor started responding properly.

Another issue appeared on the LCD. The last two display positions occasionally showed all segments turned on. This happened because those positions were not being updated or cleared consistently. Explicitly writing blank characters to the unused positions improved the display behaviour.

### What I Learned

* When sensor readings are completely unexpected, hardware connections should be verified before making major code changes.
* Timer and clock configuration are important when working with time-based sensors such as the HC-SR04.
* LCD positions that are not updated may retain previous segment data.
* Debugging embedded systems often requires checking both software and hardware simultaneously.

