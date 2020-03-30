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

## NEC Protocol

I decided to implement a standard protocol rather than a homemade one, as a learning exercise. Since I will also be the one implementing the receiver, it really wasn't necessary, however this code can now be easily adapted to interface with a real appliance that uses the same protocol.

A multitude of protocols exist, but i chose the NEC one. It's popular and well documented.
<figure>
    <img src="https://techdocs.altium.com/sites/default/files/wiki_attachments/296329/NECMessageFrame.png" alt="A NEC Frame">
    <figcaption>In the above image we can see a NEC frame, which shows the timings of each fragment.</figcaption>
</figure>

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

However, there's a tiny detail left. While the signal sent may appear as a steady '1' or '0', that's actually not true. They are modulated at a carrier frequency of 38Khz. This means that, actually, the signal is rising and dropping 38 thousand times per second. If we were to zoom in to a NEC transmission we would observe the following:

<figure>
    <img src="/assets/images/nec_38khz.PNG" alt="Zoomed in version of an IR transmission">
    <figcaption>As we can see, the signal is modulated at 38 Khz.</figcaption>
</figure>

The PWM module of the PIC10F322 will be very useful at implementing the carrier wave. It would still be possible without it, by bit-banging the output pin, but this makes it much easier.

Now that we've seen how the NEC protocol works, let's move to our microcontroller.
## The PIC10F322

This microcontroller contains:
* 0.896 KB Flash
* 64 B SRAM
* 16 MHz maximum internal oscillator
* 4 GPIO (one of them is input only)