---
title: "Hardware and Configuration"
date: 2021-01-06T10:59:29Z
weight: 1
tags:
- Hardware
- Raspberry Pi
- PoE
- Python
- VLAN
---

The hardware used for the first step of the Homecloud project is made up of two Raspberry Pis, one 3b with 1 GB and one 4b with 8 GB of RAM.
The former is intended to run the control plane while the latter will be running workloads.

The Pis are powered over PoE from an [Netgear GS510TLP](https://www.netgear.com/business/wired/switches/smart-managed-pro/gs510tlp) managed switch using the [Waveshare PoE hat](https://www.waveshare.com/wiki/PoE_HAT_(B)).

![Pis with Switch](/img/switch.jpg)

## Operating System

The Pi 3 is running Ubuntu 20.10 32-bit, the Pi 4 the 64-bit variant. These are flashed from the original images and boot off of 64GB Samsung Evo Plus microSD cards.

After initial boot, a local screen and keyboard help identify the DHCP-assigned IP addresses of both machines.

## Verifying the Hardware

After having had some trouble with an earlier Pi and PoE setup, it felt advisable to test the stability of both machines under load before proceeding.


### Installing vcgencmd

The most accurate way to measure temperature and core frequency is the `vcgencmd` binary which is only available from a `ppa` since it is not open-source. Further, at the time of writing, the `ppa` does not provide packages for the latest versions of Ubuntu, although previous versions do work.

To install `vcgencmd`, the apt source thus needs to be modified:

```
$ sudo add-apt-repository ppa:ubuntu-raspi2/ppa

# Replace "xenial" keyword with "bionic" and save
$ sudo nano /etc/apt/sources.list.d/ubuntu-raspi2-ubuntu-ppa-xenial.list

$ sudo apt update
$ sudo apt install libraspberrypi-bin

$ sudo usermod -aG video ubuntu

$ sudo reboot
```

The addition of user `ubuntu` to the `video` group is necessary to allow the binary access to the video core device when not running as `root`.

### Stress-Testing the Pis

Both Pis were then subjected to a monitored stress test using the `stress` utility while monitoring core frequency and temperature:

```
$ while true; do vcgencmd measure_clock arm; vcgencmd measure_temp; sleep 10; done & stress -c 4 -m 2 -t 900s
```

The results were satisfactory. Sitting on the bench with the PoE hat fans enabled, both Pis settled at around 53°C with no throttling.


## Using the Hat Display

The PoE hats feature a tiny OELD screen that can be used to display useful information like temperature and IP address.
The screen is accessed via I²C and Waveshare provide sample code for how to do this.

Unfortunately the C code provided did not work at all. The Python code functioned at first, but the screens exhibited some corruption after some time which could be fixed by restarting the Python script. More investigation into this will be required.

A link to the example code is available on [Waveshare's product wiki](https://www.waveshare.com/wiki/PoE_HAT_(B)).

The listed prerequisites are a mix of C and Python dependencies and partly out of date. The following combination ultimately provided everything that's needed:

```
$ sudo apt install p7zip-full libtiff5 libopenjp2-7 libatlas-base-dev \
    python3-pil python3-numpy python3-rpi.gpio python3-smbus

$ wget https://www.waveshare.com/w/upload/b/b7/PoE_HAT_B_code.7z

$ 7z x PoE_HAT_B_code.7z -r -o./PoE_HAT_B_code

$ sudo python3 PoE_HAT_B_code/python/example/main.py
```

![Hat Display](/img/poe_hat.jpg)