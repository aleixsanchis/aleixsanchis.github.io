---
layout: post
title: "USB IR Receiver using a PIC16F455"
description: "Implement a HID USB device to receive IR commands and control media in the computer."
comments: true
keywords: "PIC16f455, PIC, IR, Remote, NEC, Embedded, USB, HID"
---

In this post we will explore how to implement an USB IR receiver which will send HID keycodes to the host computer. This project is meant to be complimentary with an IR Emitter, which you can find [here](../ir-emitter), aimed at controlling my computer from a distance while watching movies; for example to pause it or change the volume.

To implement the USB communications I used a PIC16F455, a cheap MCU with built-in USB support, and it comes in a PDIP package (for example, no such chip exists for the AVR family).
