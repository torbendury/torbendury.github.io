---
layout: post
title: Set Up Your Raspberry Pi As a VPN
categories: [Home]
---

Learn how to set up your spare Raspberry Pi as a VPN into your home network. Browse the internet safely through your known network and hide your traffic from e.g. open WiFi hotspots.

**NOTE:** This was tested with a Raspberry Pi Zero 2W (new model with WiFi), so _I'd_ say it will work with basically every Pi available.

Let's jump in!

## Flashing your operating system to SD card

If you've already flashed a operating system for your Raspberry Pi to a SD card, you can skip this. Go ahead and install a headless Raspberry Pi OS Lite, enable SSH and WiFi to remotely access it. After that, jump [here](#configuring-openvpn).

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
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=<Insert 2 letter country code here>
network={
 ssid="MY-WIFI-NAME"
 psk="MY-PASSWORD"
}
```

Hit save, exit, insert your SD card into your Raspberry Pi and power it on!

## Configuring OpenVPN

In the next few step, we are going to:

- Connect to our Raspberry Pi
- Update the Pi
- Install OpenVPN software
- Secure our installation
- Create client certificates

### Connect

When powered on, your Raspberry Pi is going to try to connect to your configured WiFi. If you have DHCP enabled, your Pi will automatically get a local IPv4 address. Locally, it's reachable with hostname `raspberrypi`.

Open a shell / terminal and type

```bash
 $ ping raspberrypi
 # There should be a response here
```

_If you can't reach it by host, try to look the IP up in your local DHCP provider, e.g. your router._

When you get a successful response, log into your Raspberry Pi:

```bash
 $ ssh pi@raspberrypi
 # OR
 $ ssh pi@<assignedIP>
 [...]
```

The default password is `raspberry`.

When logged in, change your password by typing `passwd` and enter a new, secure password.

### Update the Pi

After that, we first will do an update:

```bash
 $ sudo apt update && sudo apt upgrade -y
 [...]
```

### Download PiVPN

In this tutorial, we make it simple and use PiVPN. If you're into it, you can build everything up from scratch, but when there's a simple solution, we should use it.
First, hit [the official PiVPN website](https://www.pivpn.io/) and copy the install command.

_Yes, this utilizes "curl pipe to bash" and I hate it, but the install script is safe to use and makes everything pretty easy._

This will download some tooling. Based on your Pi board, this can take some time, have a coffee.

After that, it will open up a terminal GUI. You're asked to assign your Pi a static IP in your home network. This is crucial, do not skip it! I own a FritzBox router, which enabled me to assign the Pi a static IP by simply ticking a checkbox. If you're unsure how to accomplish this in your home network, there's a solution for (basically) every router publicly available - go to your favorite search engine and look it up!

#### Settings

When prompted to choose a user, you can either stick to the default (username `pi`) or create a new one. For this tutorial, we will stick to `pi`.

Choose `OpenVPN` when asked if you prefer WireGuard or OpenVPN. For our purposes (mainly speed!) we rely on using UDP. For simplicity, we stay on the default UDP port `1194`.

Since you will need to be able to connect from the "public" internet and your internet provider most probably changes your public IPv4 address everye 24/48 hours, you will need a service called DynDNS. This is a third party tool that gives you a public domain name and routes to your ephemeral IPv4 address. I use [dynu.com](https://dynu.com) - it is safe to use, free and super easy to setup. Make an account there, setup your router to use it and come back when you're done.

![How traffic is routed to your Pi](https://www.plantuml.com/plantuml/svg/RP2nQiD038RtUmf1bzQ3Io1BtL02oP0kmJIqI_7Wt2csY2qPHUVWjw_hrj12Lad_z_reVR5IhIKERTcvvFEkeQgsId4eiar3oEPMNWA-kCFt8NXXHcya32RmlispnU9fwNOb1v0U5VmK0ezgT29V6hhLum_XsIMZu5gJOP5jzmVeL7eAgBC2qog5C71ClRHEyI9DZm46YGfTF3RauJM_xvSF_v1pwMCJiO2Tj0Xl4ct49jEoKaGkifm-ylrisjJepxUwJaamBK_Z08XDe1w9Zj6kekS_uZLo-FtR5m00)
_From left to right: How your traffic is routed_

When we entered our DNS information, we can choose to enable elliptic curve algorithms for encryption and stay with a standard 256 Bit encryption (if you're paranoid, go for the 521 Bits).

You will be asked to install "unattended upgrade" utilities. This is going to be installed via `pip` (package managing for python modules). It will ensure that at least security-relevant upgrades will be installed without your supervision. Do it!

When the installer is done, reboot your Pi.

### Create client certificates

Your devices will connect to your VPN via a DNS, using a client certificate and a password of sufficient length (which, again, depends on your paranoia).

To create one, log into your Pi and run:

```bash
 $ ssh pi@pi-ip-address
 [...]

 $ pivpn add
```

This will start an interactive session where you can enter:

- The name for the new client
- How long the certificate should last before it needs to be renewed
- a secure password

Note down the password (safely) and copy your new created certificate from `/home/pi/vpn/myclientcert.pem` to your local device.

Basically, we're done now. Make sure to configure your router to allow incoming traffic on port UDP 1194 to your Raspberry Pi.

### Aftermath: Securing your installation

Let's take some security considerations for now. Basically, your router has opened a port which allows _some_ access into your home network. Everyone that successfully authenticates against your OpenVPN installation will be able to mess around in your home net. Let's avoid this!

- ONLY allow UDP 1194 on incoming traffic from **outside** your network (can be done on your router firewall)
- On your Raspberry Pi, install `ufw` by `sudo apt install ufw` and configure it to...
  - allow UDP 1194 from everyone
  - allow TCP 22 from inside your home network
  - Don't forget to enable it! `sudo ufw enable`

Keep your system updated. `unattended-upgrades` is sufficient for system-stuff, but also keep your OpenVPN server updated on a frequent basis.

On your router, you are able to restrict traffic inside your own home network. Make use of it! Clients which enter your network through your OpenVPN Pi are mostly not going to need to do stuff in your network, they often just want to access a home cloud or use it for tunnelling traffic to the public internet. Restrict everything as needed and use _least privilege_ whenever possible.
