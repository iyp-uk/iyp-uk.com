---
title: "Getting Started With Bluetooth Low Energy"
date: 2017-08-26T18:14:18+01:00
tags: [ "IoT", "bluetooth", "BLE" ]
---

## Talking electronics

Hang on, what is Bluetooth in the first place?
Have a look at the definition on the [official bluetooth website](https://www.bluetooth.com/what-is-bluetooth-technology/how-it-works).

One important thing to notice is that Bluetooth is a wireless technology standard using [UFH](https://en.wikipedia.org/wiki/Ultra_high_frequency) (Ultra High Frequency) **radio waves**.
This means it's in the same family of telecommunications than GSM, GPS, WiFi, walkie-talkies and many more.

As you have probably discovered on their site, Bluetooth comes in different flavours:

* Basic Rate/Enhanced Data Rate (BR/EDR)
  * This is for **point to point** connections (1:1) and allows large amounts of data to be communicated. Your bluetooth speaker uses that :)
* Low Energy (LE) - *This is the one we're interested in*
  * Allows **point to point** (1:1) for connected device products, such as fitness trackers and health monitors
  * Allows **Broadcast** networking (1:m) making it ideal for beacon solutions, such as point-of-interest (PoI) information and item and way-finding services
  * Allows **Mesh** networking (m:m) for building automation, sensor network, asset tracking and any solution where multiple devices need to reliably and securely communicate with one another

On the Low Energy version, the key is as its name suggests, keeping energy consumption low. 
So the device actually sleeps most of the time. Read on, and we'll know more about this technology and why it matters.

> You can find a nice summary [here](https://docs.google.com/gview?url=https://www.bluetooth.com/~/media/files/marketing/bluetooth-connectivity.pdf).

## Bluetooth Low Engery: an introduction

Bluetooth Low Energy, or BLE or even Bluetooth Smart, is part of the Bluetooth 4 [core specification](https://www.bluetooth.com/specifications/bluetooth-core-specification).
It is optimized for low cost, low bandwidth, low power, and low complexity.
An illustration of that is its max range (2-5 meters) and transmission rate (10KB/s - even there you'd drawn your battery fairly quickly).

> Currently, the latest Bluetooth core specification is version 5, which enhances BLE for Internet of Things usage. 
It increases range, and the amount of data which you can communicate, but the principles remain the same. 

One major difference with the previous spec, is the possibility to operate in modes which are not *point to point*. 
No need to "pair" with Bluetooth LE, devices rather communicate with one another (see *Broadcast* and *Mesh* descriptions above)



