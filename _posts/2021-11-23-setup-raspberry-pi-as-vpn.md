---
layout: post
title: Set Up Your Raspberry Pi As a VPN
categories: [Cloud]
---

Learn how to set up your spare Raspberry Pi as a VPN into your home network. Browse the internet safely through your known network and hide your traffic from e.g. open WiFi hotspots.

**NOTE:** This was tested with a Raspberry Pi Zero 2W (new model with WiFi), so I'd say it will work with basically every Pi available.

## Flashing your operating system to SD card

If you've already flashed a operating system for your Raspberry Pi to a SD card, you can skip this. Go ahead and install a headless Raspberry Pi OS Lite.

For everyone else, here's the easiest and safest way to flash your SD card and install an operating system on it.

## Download & Install the Official Imager

If you're using Windows, download the official Imager [here](https://downloads.raspberrypi.org/imager/imager_latest.exe) which is provided by [raspberrypi.org](https://raspberrypi.org).

MacOS users can get their version [here](https://downloads.raspberrypi.org/imager/imager_latest.dmg). This is the latest build packed as an `.dmg` file.

For Debian'ish Linux users: The Imager is available in official `apt` repositories and can be installed from your terminal:

```bash
sudo apt update && sudo apt install -y rpi-imager
```

## Flash the SD card and prepare the operating system

Next, insert your SD card into your machine.

No matter which OS you're on, open the search bar and type "Raspberry Pi Imager" to find the application and open it. Under "Operating System", choose the _Raspberry Pi OS Lite (ARM)_. Select your SD card and hit the "WRITE" button.

When you're finished, unplug and re-insert your SD card into your machine. Your favorite file explorer will pop up and show you the content of the SD card.

### Enable remote SSH

Create a new empty file called `ssh`. Done. On first startup, your Raspberry Pi will know what to do with it. You will be able to `ssh` into the Pi with the following standard credentials:

```bash
User: pi
Pass: raspberry
```

You don't have to remember it, because I generally recommend changing it as soon as you logged into the machine (I'll show you later how to accomplish that). Remember: This Raspberry Pi will be _quite_ open to the Internet, so choose a good password.

### Enable and configure WiFi

Create another file called `wpa_supplicant.conf` on the SD card. Open it with your favorite text editor and enter the following information:

```conf
# wpa_supplicant.conf
network={
       ssid="MY-WIFI-NAME"
       psk="MY-SUPER-SECRET-PASSWORD"
       key_mgmt=WPA-PSK
}
```

Hit save, exit, insert your SD card into your Raspberry Pi and power it on!


