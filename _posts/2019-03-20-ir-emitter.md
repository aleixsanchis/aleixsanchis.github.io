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
* It has a PWM, which will be very useful in creating the carrier signal

After finishing the firmware, I saw that I could have chosen something even smaller but I wanted to err on the safe side. Shipping isn't free! (i actually had a coupon and it was).

This project is meant to be complimentary with an USB IR Receiver which is still a Work in Progress, aimed at controlling my computer from a distance while watching movies; for example to pause it or change the volume.

## NEC Protocol
<figure>
    <img src="https://techdocs.altium.com/sites/default/files/wiki_attachments/296329/NECMessageFrame.png" alt="A NEC Frame">
    <figcaption>In the above image we can see a NEC frame, which shows the timings of each fragment.</figcaption>
</figure>

## The PIC10F322

This microcontroller contains:
* 0.896 KB Flash
* 64 B SRAM
* 16 MHz Internal Oscillator