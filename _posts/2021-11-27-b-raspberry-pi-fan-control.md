---
layout: post
title: Building a fan controller for your Raspberry Pi
categories: [Embedded, Raspberry Pi, Hardware Programming]
---

Learn how to build a simple and affordable fan controller for your Raspberry Pi to connect a simple fan with temperature control.

## What you will need

- 1x [Raspberry Pi Board](https://www.berrybase.de/raspberry-pi/raspberry-pi-computer/boards/raspberry-pi-4-computer-modell-b-8gb-ram?c=319)
- 1x standard 5V fan (or a [cool case with a fan included](https://www.berrybase.de/neu/geh-228-use-f-252-r-raspberry-pi-4-mit-l-252-fter-stackable-transparent/schwarz?c=2389))
- 1x [4,7k Ohm resistor](https://www.berrybase.de/bauelemente/passive-bauelemente/widerstaende/metallschichtwiderstaende/0-25w-1/metallschichtwiderstands-kit-2-7k-75k-ohm-0-25w-177-1-axial-durchsteckmontage)
- 1x [standard NPN resistor](https://www.berrybase.de/bauelemente/aktive-bauelemente/transistoren/transistoren-p../pn2222abu-bipolarer-transistor-npn-40v-1a-300mhz-hfe-35-to-92-3-pin), the graphic will show a BC547-B but the linked one also does the trick
- some wires, maybe a PCB if you want to solder it. For tinkering, a solid [breadboard](https://www.berrybase.de/raspberry-pi/raspberry-pi-computer/prototyping/breadboard-mit-830-kontakten) with some [jumper cables](https://www.berrybase.de/raspberry-pi/raspberry-pi-computer/kabel-adapter/gpio-csi-dsi-kabel/40pin-jumper/dupont-kabel-set-je-1x-f-f/m-m/f-m-10cm) are sufficient.

## What we will build

<div style="text-align: center"><img src="/images/2021-11-28/01.png"/></div>

To connect with our Raspberry Pi, we will use a **5V** output. Also, we'll utilize the **GND** of our Pi. To control our fan, we will use **GPIO14** as a data output which controls the transistor.

Our fan is connected to the **5V** output and to the transistor.

In between our Pi and our transistor, we put a **4,7k Ohm** resistor.

All in all, your setup could look like this on a breadboard:

<div style="text-align: center"><img src="/images/2021-11-28/02.png"/></div>

## Configuring your Raspberry Pi

When you're all hooked up, connect to your Raspberry Pi via SSH and type:

```bash
  $ sudo raspi-config
```

From there on, head to the "Performance" tab and enable the "Fan control". You can set a target temperature (e.g. 80°C) at which the Raspberry Pi is going to power the fan on to cool itself down. When asked to put in a pin number, we will choose our connected **GPIO14** pin. Hit `Enter` and you're done!

## Extra: Testing our setup with a stress-test

When you're all built and set up, you can run a simple test directly on your Raspberry Pi. When connected via SSH, type the following:

```bash
  # Update our repository first and upgrade installed software (generally recommended)
  $ sudo apt update && sudo apt upgrade -y
  # Install `stress`
  $ sudo apt install stress
  # Install an easy to use Python module made for stress-testing Raspberry Pis
  $ pip3 install stressberry
```

When you're done installing everything, you can run a test and generate some report output:

```bash
  $ stressberry-run output.dat
```

Soon, you will start to hear your fan running to cool the Raspberry Pi down.

It is proven that external fans help cooling down your Pi system - especially Pi 4 boards which tend to get a little hotter than previous boards. Any doubts? Disable your fan control in your `raspi-config` and run the `stressberry` test again. Compare the data outputs and convince yourself!

## Conclusion

Building your own fan control is cost-effective and can be done in less than half an hour. It is very helpful to learn more about transistors and how to control circuits with it on a very high level.
