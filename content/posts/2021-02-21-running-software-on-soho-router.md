---
date: 2021-02-21
layout: post
title: Running services on your SOHO router for the greater good
description: Detailing the process of hacking a router all the way to running your own blocking DNS server on it in a maintainable way.
author: kmhn
updated_date: 2021-03-07
---
Recently, I've been wanting to run a [PiHole](https://pi-hole.net) server for ad-blocking in my home network, but I didn't want to set up a machine exclusively for it.
So, I thought I could reuse a Raspberry Pi 4 that is running [OctoPrint](https://octoprint.org) for a PiHole setup. This worked ok, but I wasn't fan of the setup for a couple of reasons:
1. The RPi4 is hooked up to the network over WiFi. Ideally, the recursive DNS resolver that my entire home network will be using should not have a lot of latency, and I would rather not have it be on a WiFi connected node since most of our personal devices at home are also on WiFi.
2. I don't keep the RPi4 running at all times to conserve energy, but I would have to if I'm going to run a DNS server on it.

I could live with the above, but can't one do better? Just run it on the one device that actually has to be always on and is still (hopefully) low power, your WiFi home router!

In this post, I'll briefly go over what the methodology I follow entails and what I ended up doing in this particular instance.

### TL;DR
TODO
If you have a TP-Link Archer C3200 or a "similar enough model" and you'd like to run your own tools/services on it, [follow the instructions in this GitHub repository]().

## 0x01 SETUP
The WiFi router we have is a [TP-Link Archer C3200](https://www.tp-link.com/us/home-networking/wifi-router/archer-c3200/) which we got on sale a couple of years ago. It is not exactly a great starting point if you'd like to run any other piece of software the vendor didn't want you to. The vendor simply does not allow you to execute your own code. There is _some_ access to the device over Telnet but that is highly limited and presents configuration options that are (mostly) in the Web UI anyway. SSH is listening and accepts log-ins, but you cannot actually get a shell session or execute any commands..

All hope is not lost, yet! One could get around these limitations by, say, running an open source router firmware like [OpenWRT](https://openwrt.org/) or [DD-WRT](https://dd-wrt.com/). However, none of these projects have any builds available for the TP-Link Archer C3200.. That's because well, the vendor locked the device down and it would only accept signed firmware from the vendor.

Ah well, none of this is surprising and is actually standard fare for pretty much every cheap - if you consider $100+ cheap - consumer-grade device out there to my knowledge, so let's just hack it up.

<p align="center">
  <img src="https://media.tenor.com/images/82aad35c011c912856810c247183ec0c/tenor.gif" alt="hack the planet" />
</p>

## 0x02 INFORMATION GATHERING
One of the things I've seen hardware hackers do when they get their hands on a device is take the time to figure out how it's physically put together. Then they would delicately (mostly) take it apart in such a way that it can be put back together, take a ton of pictures for future reference and take stock of the different subsystems that comprise the hardware.

As for me, unlike actual hardware hackers, I'm an amateur. This means that unless I'm reaaaaaaaaaally out of options, I wouldn't even dare open up the device. I simply don't trust myself to successfully put the device back together, so more often than not, I constrain myself to the software side. First order of business is get a copy of the firmware, usually in some binary format, which we can dissect to try and do a plethora of things:
- Get an idea for how the software side is put together, so what kind of software does the device run, and more importantly, can we use it for our purposes?
- Find default keys or passwords for services like SSH and/or telnet (*cough* backdoors *cough*)
- Get our grubby hands on some executables that we can reverse engineer for more information or to find vulnerabilities that we could exploit

The list goes on and on, you get it.

Luckily, though, I won't be needing to open up the device to get the firmware binary since it's available for [download from the vendor's website](https://www.tp-link.com/en/support/download/archer-c3200/#Firmware). We'll grab ourselves a copy of the latest firmware and find the desired binary blob inside the zip file. At this point, the usual procedure I would go through is to run [`binwalk`](https://github.com/ReFirmLabs/binwalk), extract bootloaders/filesystems/whatever and pray that there is no annoying, weird obfuscation. As luck would have it again, there's a very obvious squashfs filesystem that binwalk + sasquatch extracted without a hitch.

```sh
$ binwalk -e "Archer_C3200v1_0.9.1_0.1_up_boot(160712)_201
6-07-12_15.24.12.bin"

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
115204        0x1C204         LZMA compressed data, properties: 0x5D, dictionary size: 65536 bytes, uncompressed size: 268788 bytes
263168        0x40400         LZMA compressed data, properties: 0x5D, dictionary size: 65536 bytes, uncompressed size: 3658912 bytes
1769984       0x1B0200        Squashfs filesystem, little endian, version 4.0, compression:xz, size: 11220560 bytes, 696 inodes, blocksize: 131072 bytes, created: 2016-07-12 07:22:00

$ ls "_Archer_C3200v1_0.9.1_0.1_up_boot(160712)_2016-07-12_15.24.12.bin.extracted/squashfs-root"
bin  dev  etc  lib  linuxrc  mnt  proc  sbin  sys  tmp  usr  var  web
```

