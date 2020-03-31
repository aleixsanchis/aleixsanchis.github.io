---
layout: post
title: "IR Emitter using PIC10F322"
description: "Implement NEC codes using a PIC10F322 to control different appliances"
comments: true
keywords: "PIC10F322, PIC, IR, Remote, NEC, Embedded"
---

In this post we will explore how to implement an IR Emitter using a microcontroller, in this case a [PIC10F322](https://www.microchip.com/wwwproducts/en/PIC10F322). I chose this microcontroller because:

* It's cheap
* It's tiny
* It has a PWM module, which will be very useful in creating the carrier signal

After finishing the firmware, I saw that I could have chosen something even smaller but I wanted to err on the safe side. Shipping isn't free! (i actually had a coupon and it was).

This project is meant to be complimentary with an USB IR Receiver which is still a Work in Progress, aimed at controlling my computer from a distance while watching movies; for example to pause it or change the volume.

The full code for this project is available at https://www.github.com/aleixsanchis/PICRemote

## The NEC Protocol

I decided to implement a standard protocol rather than a homemade one, as a learning exercise. Since I will also be the one implementing the receiver, it really wasn't necessary, however this code can now be easily adapted to interface with a real appliance that uses the same protocol.

A multitude of protocols exist, but i chose the NEC one. It's popular and well documented.
<figure>
    <img src="https://techdocs.altium.com/sites/default/files/wiki_attachments/296329/NECMessageFrame.png" alt="A NEC Frame">
    <figcaption>In the above image we can see a NEC frame, which shows the timings of each fragment.</figcaption>
</figure>
<br>
In the NEC protocol the messages are encoded as follows:

* Logical '0': 562.5µs pulse burst followed by a 562.5µs space
* Logical '1': 562.5µs pulse burst followed by a 1.6875ms space

A NEC transmission begins with a 9ms burst, followed by a pause of 4.5ms. This is immediately followed by the address of the intended receiver, LSB first. After the address, the inverted address is transmitted. For example, if we were transmitting for address 0b0000\_0100, the inverted address would be 0b1111\_1011. This is aimed at increasing the reliability of the protocol; if one value is not the exact inversion of the other, that means there was a transmission error and the message can be discarded.

After the address the actual command is sent, which will be interpreted by the receiver. And finally, followed by the command, the inverted bitwise value of the command will be sent, just like in the address. The frame is ended by sending a final 562.5us pulse burst to signify the end of message transmission.

So, all in all, we got:

* a 9ms leading pulse burst (16 times the pulse burst length used for a logical data bit)
* a 4.5ms space
* the 8-bit address for the receiving device
* the 8-bit logical inverse of the address
* the 8-bit command
* the 8-bit logical inverse of the command
* a final 562.5us pause

Now, there's a tiny detail left. While we are sending a burst, the signal appears to be a "block" or a continuous stream of data. However, that's actually not true. The signal is modulated at a carrier frequency of 38Khz. This means that, actually, the signal is being switched on and off 38 thousand times per second. If we were to zoom in to a burst we would observe the following:

<figure>
    <img src="/assets/images/ir_led/nec_38khz.png" alt="Zoomed in version of an IR transmission">
    <figcaption>As we can see, the signal is not continuous, but rather is oscillating at 38 Khz.</figcaption>
</figure>
<br>
The reason of this modulation is to prevent other sources of IR to interfere in the transmission. For example, you can still use your remote in bright daylight, despite the sun being a source of infrared light much stronger than the tiny LED in the remote. This is because the receiver on your TV filters any signal that is not at that specific frequency (sometimes other frequencies are used, like 36 or 40 Khz).

The PWM module of the PIC10F322 will be very useful at implementing the carrier wave. It would still be possible without it, by bit-banging the output pin, but it makes it much easier.

Now that we've seen how the IR and NEC protocol works, let's move on to our microcontroller.
## The PIC10F322

<figure>
    <img src="/assets/images/parts/pic10f322.png" alt="A picture of a PIC10F322">
</figure>

This microcontroller contains:
* 0.896 KB Flash
* 64 B SRAM
* 16 MHz internal oscillator
* 4 GPIO (one of them is input only)

More than enough for our project! (Actually I wish I had one extra pin but that's what you get for not doing enough research beforehand).

To program it we will use the MPLABX IDE with the XC8 compiler provided by Microchip. Both are gratis, but not free software and can be downloaded from Microchip's website.

## The circuit

Since I wanted my remote to have eight buttons, a Parallel-In Serial-Out (PISO) shift register was needed to have enough inputs. A PISO shift register is a device that allows us to increase the amount of inputs by storing a parallel set of data and then outputting them one by one on each clock. This way we can read the eight buttons with only 3 pins. In this project, I used a 74HC165 from Philips.

<figure>
    <img src="/assets/images/ir_led/schematic.png" alt="Schematic of the IR circuit">
    <figcaption>The image above shows the completed circuit. Only 3 buttons are shown here to keep the schematic simpler.</figcaption>
</figure>
<br>

The pins are connected in the following way:

* RA0 (output) -> Shift Register SH/!LD. This pin controls when the inputs are loaded to the shift register.
* RA1 (output) -> IR LED. This pin is driven by the PWM module.
* RA2 (output) -> Shift Register CLK. On each rising edge of this signal, the register will shift a new value to its output
* RA3 (input) <- Shift Register Serial Output. This pin will read the state of the buttons.

Each button has its corresponding pull-up resistor, so a Low value will mean the button is pressed.

## The Code

#### Device configuration

To configure the device I used the following parameters:

* Internal Oscillator at 8Mhz
* Watchdog Timer and Brown-out Reset Disabled
* Low Voltage Programming disabled (more on this now)

The PIC10F322 has 4 GPIO, but RA3, aside from being input-only, also serves as !MCLR. This means that, under "normal" conditions (with Low-Voltage Programming, LVP, enabled), the device would reset when this input is driven Low. To be able to use this pin as an input, LVP must be disabled. Then, to enter programming mode, RA3 needs a high voltage, in the order of 8-12V. For more information about LVP, and how to configure the device, consult the [datasheet](http://ww1.microchip.com/downloads/en/DeviceDoc/40001585D.pdf).

So, my configuration is as follows:

{% highlight c %}
// CONFIG

#pragma config FOSC = INTOSC    // Oscillator Selection bits->INTOSC oscillator: CLKIN function disabled

#pragma config BOREN = OFF    // Brown-out Reset Enable->Brown-out Reset enabled

#pragma config WDTE = OFF    // Watchdog Timer Enable->WDT disabled

#pragma config PWRTE = OFF    // Power-up Timer Enable bit->PWRT disabled

#pragma config MCLRE = OFF    // MCLR Pin Function Select bit->MCLR pin function is digital input, MCLR internally tied to VDD

#pragma config CP = OFF    // Code Protection bit->Program memory code protection is disabled

#pragma config LVP = OFF    // Low-Voltage Programming Enable->High-voltage on MCLR/VPP must be used for programming

#pragma config WRT = OFF    // Flash Memory Self-Write Protection->Write protection off
{% endhighlight %}

#### The main loop

In the following code block we see the main loop used in the firmware:

{% highlight c linenos=table %}
void main(void) {
    // initialize the device
    SYSTEM_Initialize();

    while (1) {
        // LOAD INPUTS TO SHIFT REGISTER
        SHIFT_REG_SH_NLD = 0;
        SHIFT_REG_CLK = 0;
        // SHIFT EACH INPUT ONE BY ONE
        for (uint8_t i = 0; i < 8; i++) {
            // CHECK IF ANY INPUT IS PRESSED
            uint8_t input = SHIFT_REG_INPUT;

            // IF PRESSED (PULLED TO LOW) SEND TO IR EMITTER
            if (input == LOW) {
                ir_emit(i);
            }
            // ENABLE SHIFTING
            SHIFT_REG_SH_NLD = 1;
            // MAKE SURE THE SHIFT IS ENABLED
            __delay_us(1);
            // RISE CLK
            SHIFT_REG_CLK = 1;
            // MAKE SURE THAT CLK STAYS LOW ENOUGH TIME
            __delay_us(1);
            SHIFT_REG_CLK = 0;
        }
    }
}
{% endhighlight %}