---
title: "Planespotting with Rust: using nom to parse ADS-B messages"
date: "2023-09-02"
---

Using *software defined radio* (SDR) to listen to [ADS-B](https://mode-s.org/decode/content/ads-b/1-basics.html) transmissions has been on my mind for a while. This article is inspired by the [cool experiment by Charlie Gerard](https://charliegerard.dev/blog/aircraft-radar-system-rtl-sdr-web-usb/), who used JavaScript to parse and display ADS-B messages as they come through the WebUSB interface right in the browser. I wanted to do something similar, but use my go-to parsing library [nom](https://github.com/rust-bakery/nom) and the terminal.

## ADS-B

ADS-B is a protocol used by aircrafts to broadcast their position, altitude, speed, and other information. Nowadays, the majority of aircraft broadcast ADS-B messages constantly. Anyone with the right equipment can listen to these messages. You can buy a relatively cheap USB dongle with an antenna on Amazon and then read its manual that probably explains how to install drivers for it on Linux. In my case I used [usbipd-win](https://github.com/dorssel/usbipd-win) to mount the USB device inside Ubuntu running in WSL2. Then I installed the Linux drivers and [dump1090](https://github.com/flightaware/dump1090), a program that makes use of these drivers and then outputs ADS-B messages in a format that is easy to parse. While you can use `dump1090` to display a neat table full of information about aircrafts, I wanted to use its raw output capabilities to parse ADS-B messages myself. It starts a simple TCP server that outputs raw ADS-B messages wrapped in Mode-S Beast *frames*. I'm not sure what `Beast` means, but I found something that looks like its spec [here](https://github.com/firestuff/adsb-tools/blob/master/protocols/beast.md).

The [ADS-B specification](https://mode-s.org/decode/content/ads-b/1-basics.html) is pretty easy to read. Basically every message is 112 bits long and has the following structure:

```
+----------+----------+-------------+------------------------+-----------+
|  DF (5)  |  CA (3)  |  ICAO (24)  |         ME (56)        |  PI (24)  |
+----------+----------+-------------+------------------------+-----------+
```

The DF (downlink format) field can be used to figure out whether this is an ADS-B message. This time I only care about DF=17 or DF=18 messages, the only messages in ADS-B format. CA (capability) is kind of boring: capabilities of the hardware that broadcasts the message. ICAO (International Civil Aviation Organization) 24-bit address or (informally) Mode-S "hex code" is a unique identifier of the aircraft that normally never changes. ME (message extension) is the actual message. The contents of this field depend on its first 5 bits, the [typecode](https://mode-s.org/decode/content/ads-b/1-basics.html#ads-b-message-types). For example, aircrafts identify themselves and send their altitude information in messages with different typecodes. This time I want to at least recognize aircrafts' callsigns (TC 1-4) and their altitude (TC 9-18, 20-22).

