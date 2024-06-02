---
title: Pulsating LEDs using Arduino and Rust
---

In my [last post](./2024-04-16-program-arduino-using-rust.md), I went over the basics of programming an Arduino using Rust to make an LED blink.

In this post, I'm going to show a slightly more complex example that makes a set of LEDs pulse in-and-out at a rate that you can change by twisting a dial.
I introduce how to read analog inputs, how to use pulse-width modulation to make the Arduino's pins output analog-like signals, and how to use the serial monitor for easy print debugging.

### The circuit

For this post, I'll be using a simple circuit that can be made with the materials in the [Arduino Student Kit](https://www.arduino.cc/education/student-kit/).
(These diagrams were taken from the kit as well).
It's a circuit with three LEDs powered by pins on the Arduino, as well as a potentiometer (a variable resistor that can be changed to have no resistance or a great deal of resistance), with one more pin used to detect the voltage difference induced by the potentiometer.
The LEDs will be programmed to smoothly pulse in and out.

<center>

<img alt="A circuit diagram." src="/blog/assets/images/arduino-dimmer-schematic.png" width="500">
<br>
<br>
<img alt="A diagram showing how to concretely wire the previous circuit using an Arduino and a small breadboard." src="/blog/assets/images/arduino-dimmer-wiring.png" width="500">

</center>

There's two ideas worth covering here: (1) how does the input pin detect changes to the potentiometer, and (2) how does the Arduino output partial power to the LED?

Let's start with (1).
Following the circuit, suppose 5V of conventional current comes down the wire to the potentiometer.
If the potentiometer is turned to its lowest setting, then about 0 ohms of resistance will be applied, so all 5V can "flow" to the input pin.
Conversely, if the potentiometer is turned to its highest setting, a high amount of resistance will be applied (the potentiometer I'm using goes up to 10,000 ohms), and a lower voltage will be received at the input pin.

Ok, now what about (2)?
When it comes to outputting signals on pins 9, 10, and 11 to power the LEDs, the Arduino isn't actually able to send an analog value to the circuit (some value between 0% and 100%).
But it's possible to simulate analog values using a process called pulse width modulation (PWM).
At a high level, this just means the hardware will oscillate between sending ON and OFF signals at a rate that matches the target analog value.
The technical term for the proportion of on time to off time is the **duty cycle**.

<center>

<img alt="Graph showing pulse-width modulation with 25%, 50%, and 75% duty cycles." src="/blog/assets/images/duty-cycle.png" width="300">

</center>

Once you have the circuit built (if you wanna do that), let's move on to coding.

### Programming the circuit

Before we jump into the Rust, let's make a tangent and look at how signals are read and set within Arduino's beginner-friendly IDE.
The code is straightforward - you can simply use the provided `analogRead()` and `analogWrite()` functions:

```c
int readValue = 0;
int writeValue = 0;

void setup() {
  pinMode(9, OUTPUT);
  pinMode(10, OUTPUT);
  pinMode(11, OUTPUT);
}

void loop() {
  readValue = analogRead(A0);  // read an analog input - returns a value between 0 and 1023
  writeValue = readValue / 4;  // divide by 4, since LED output range is 0 to 255
  analogWrite(9, writeValue);  // write an analog output
  analogWrite(10, writeValue); // ...
  analogWrite(11, writeValue); // ...
}
```

In this small Arduino sketch, we're reading from pin A0, and outputting a corresponding signal to pins 9, 10, and 11.

To achieve a similar feat in Rust, a little more set-up is needed.
It's arguably in the spirit of Rust's ethos to force the programmer to learn something about their underlying system before their code can compile, so no surprises here.

We'll start with a little bit if `arduino-hal` starter code:

```rust
#![no_std]
#![no_main]

use arduino_hal::prelude::*;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    loop {
        // TODO
    }
}
```

First, let's see if we can read the analog input.
We know we want to read from the A0 pin, so we can try writing something like this in our main function:

```rust
let dial = pins.a0.into_analog_input();
```

Unfortunately, this doesn't compile.
Rust tells us we need something called an `Adc` as an input:

```rust
error[E0061]: this method takes 1 argument but 0 arguments were supplied
   --> src/main.rs:16:26
    |
16  |     let dial = pins.a0.into_analog_input();
    |                          ^^^^^^^^^^^^^^^^^-- an argument of type `&mut avr_hal_generic::adc::Adc<Atmega, arduino_hal::pac::ADC, _>` is missing
```

An ADC is shorthand for an analog-digital converter.
This is a piece of the Arduino hardware that's responsible for reading the voltage signal and converting it into a digital number (from 0 to 1023) that we can read.
We can fix the bug by constructing the ADC and passing it in:

```rust
let mut adc = arduino_hal::Adc::new(dp.ADC, Default::default());
let dial = pins.a0.into_analog_input(&mut adc);
```

It might seem pedantic to require this, but these pin methods like `into_analog_input` can be used with many different boards, so the extra parameter ensures that you'll get a compile time error (instead of a runtime error) if you try using them with a different set of peripherals that doesn't have an ADC included.

Next, we want to send an "analog" output to the LEDs using some form of pulse-width modulation.
Looking through our available APIs, if we try setting one of our pins as outputs, there's an `into_pwm()` that seems like it should convert the output into a pulse-width modulated one.
Nice. Let's try that out...

```rust
let mut led1 = pins.d9.into_output().into_pwm();
```

Uh oh, the compiler doesn't like it:

```rust
error[E0061]: this method takes 1 argument but 0 arguments were supplied
  --> src/main.rs:22:42
   |
22 |     let mut led1 = pins.d9.into_output().into_pwm();
   |                                          ^^^^^^^^-- an argument of type `&arduino_hal::simple_pwm::Timer1Pwm` is missing
```

In this case, a chip-specific "timer" needs to provided, which provides hardware-level support for modulating the LED on-and-off according to the system's internal clock.
I imagine we could try setting the LED on and off rapidly ourselves, but letting the hardware do it for us will be more efficient.

For Arduino, it seems you need to use a different timer component depending on which pin you're pulse-width modulating, and there's an option called a "prescaler value" which configures how the timer clock is calculated relative to the system clock.
The article [Secrets of Arduino PWM](https://docs.arduino.cc/tutorials/generic/secrets-of-arduino-pwm/) goes into a lot more depth on this, but for our purposes, we can simply construct two timers for our three LEDs like so:

```rust
let timer1 = Timer1Pwm::new(dp.TC1, Prescaler::Prescale64);
let timer2 = Timer2Pwm::new(dp.TC2, Prescaler::Prescale64);

let mut led1 = pins.d9.into_output().into_pwm(&timer1);
let mut led2 = pins.d10.into_output().into_pwm(&timer1);
let mut led3 = pins.d11.into_output().into_pwm(&timer2);

led1.enable();
led2.enable();
led3.enable();
```

Finally, let's set up our main loop.
The idea will be that we read the potentiometer's value, and then do a cycle of pulsing the light from off (duty cycle set to 0%) to on (duty cycle set to 100%) and then back.
The time waited before each loop will be determined by the potentiometer value, meaning you can make the LEDs flash slower or faster.

```rust
loop {
    let read_value = dial.analog_read(&mut adc); // returns a number between 0 and 1023

    for x in (0..=255).chain((0..=254).rev()) {
        led1.set_duty(x); // expects a number between 0 and 255
        led2.set_duty(x);
        led3.set_duty(x);
        arduino_hal::delay_us(read_value as u32);
    }
}
```

Here's the final program in its entirety:

```rust
#![no_std]
#![no_main]

use arduino_hal::prelude::*;
use arduino_hal::simple_pwm::*;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    // analog-to-digital converter
    let mut adc = arduino_hal::Adc::new(dp.ADC, Default::default());
    let dial = pins.a0.into_analog_input(&mut adc);

    let timer1 = Timer1Pwm::new(dp.TC1, Prescaler::Prescale64);
    let timer2 = Timer2Pwm::new(dp.TC2, Prescaler::Prescale64);

    let mut led1 = pins.d9.into_output().into_pwm(&timer1);
    let mut led2 = pins.d10.into_output().into_pwm(&timer1);
    let mut led3 = pins.d11.into_output().into_pwm(&timer2);

    led1.enable();
    led2.enable();
    led3.enable();

    loop {
        let read_value = dial.analog_read(&mut adc);

        for x in (0..=255).chain((0..=254).rev()) {
            led1.set_duty(x);
            led2.set_duty(x);
            led3.set_duty(x);
            arduino_hal::delay_us(read_value as u32);
        }
    }
}
```

Here's how it looks controlling the flashing:

(TODO: video)

### Debugging

That's great, but what if my sensor isn't working or is giving wonky values?
Is there any way to debug it, beyond just connecting the sensor to an LED and trying to guess at how bright it is?

Thank goodness the answer is yes.
Embedded programming supports print debugging after all.
(I'd be a little worried otherwise).

The way this can be done on Arduino is with the serial monitor.
The serial monitor lets the device communicate data back to us (via the USB) - this could be debug information, real time sensor data, etc.

To add this to our program, we create a USART, right after we've created `dp` and `pins`:

```rust
let mut serial = arduino_hal::default_serial!(dp, pins, 57600);
```

Then, inside of our `loop` we can print our the current value of the input pin:

```rust
ufmt::uwriteln!(&mut serial, "Dial value: {}\r", read_value).unwrap_infallible();
```

[`ufmt`](https://docs.rs/ufmt/latest/ufmt/) is a small crate that produces slightly smaller and faster code for embedded use cases, as an alternative to Rust's built-in formatter utilities.

Once we run `cargo run` and the program is flashed to the device, we start getting print outs of our debug information. Super handy.

```
cargo run
   Compiling blink v0.1.0 (/Users/rybickic/Developer/blink)
    Finished `dev` profile [optimized + debuginfo] target(s) in 1.39s
     Running `ravedude uno -cb 57600 target/avr-atmega328p/debug/blink.elf`
       Board Arduino Uno
 Programming target/avr-atmega328p/debug/blink.elf => /dev/cu.usbmodem144101
avrdude: AVR device initialized and ready to accept instructions
avrdude: device signature = 0x1e950f (probably m328p)
avrdude: erasing chip

avrdude: processing -U flash:w:target/avr-atmega328p/debug/blink.elf:e
avrdude: reading input file target/avr-atmega328p/debug/blink.elf for flash
         with 952 bytes in 1 section within [0, 0x3b7]
         using 8 pages and 72 pad bytes
avrdude: writing 952 bytes flash ...
Writing | ################################################## | 100% 0.20 s
avrdude: 952 bytes of flash written
avrdude: verifying flash memory against target/avr-atmega328p/debug/blink.elf
Reading | ################################################## | 100% 0.13 s
avrdude: 952 bytes of flash verified

avrdude done.  Thank you.

  Programmed target/avr-atmega328p/debug/blink.elf
     Console /dev/cu.usbmodem144101 at 57600 baud
             CTRL+C to exit.

Dial value: 1023
Dial value: 1023
Dial value: 1023
Dial value: 1023
Dial value: 1023
Dial value: 997
Dial value: 957
Dial value: 942
Dial value: 939
...
```
