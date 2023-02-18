---
layout: post
title: Extending Battery Life Tremendously by Utilizing Deep Sleep on Microcontrollers
categories: [Embedded, Microcontrollers]
---

Learn to utilize deep sleep and get extended battery life for free. This post applies to nearly every popular microcontroller that is out there, including all Espressif models, Arduinos and even Raspberry Pis.

## What You Will Need

- a Microcontroller of your choice (in my case, a ESP8266)
- flashed with a firmware of your choice (in my case, MicroPython for ESP)
- basic programming knowledge for your firmware
- basic understanding of electricity

## Status Quo

Let's say you've programmed your microcontroller to do sensor stuff - classic home IoT. Take the following scenario: You own a ESP8266 microcontroller, you have flashed a firmware on it that fits your needs and you've programmed it to measure a DHT22 temperature & humidity sensor and transmit it to your favorite MQTT broker. If we cut that scenario into its steps, it will look something like this:

1. Boot the controller
2. Set up WiFi connection
3. Measure your temperature/humidity sensor
4. Transmit the measured data to a MQTT broker
5. Go to 1.

This is very straightforward and describes how many of our Home IoT devices work. Nothing special, you might say, but that's what they're intended to do: Let sensors sense.

This scenario leads to several questions:

- How long does booting take?
- Do we really need WiFi?
- How much does our sensor draw when running?
- How often do we need to repeat those steps?

Let's get back to our practical scenario to answer those questions.

## Measured Stuff

First, booting takes _about_ a second on my ESP8266+MicroPython. MicroPython is by no means intended to be super fast, and I have to clarify that WiFi connection is (in my scenario) being established right on boot-up. While booting up, the microcontroller needs about **75mA** do to its stuff.

Measuring and transmitting the data via MQTT was not possible for me to take apart because it happens...very fast. I measured about **45mA** while my ESP8266 did it. It took another 2 seconds.

After doing all this, I first decided to let my controller sleep with on-board Python libraries, using `time.sleep(60)`. When it sleeps, the controller still needs about **30 to 40mA**!

## Battery

I use simple lithium AA batteries. My controller needs at least 3V to work properly and to prevent brownfield issues, so I put 2 of 1.5V batteries in series connection to reach 3V. The batteries supply about 1733mAh.

## How Long Does It Last?

The most important thing when running on batteries is: How long does it last before I need to charge/replace? (This does not apply for solar powered controllers, obviously)

In above scenario, the battery would last about a day. For me, this means visiting my basement every day to swap batteries and charge the other ones up.

## But Why?

Sleeping isn't always sleeping. The main thread of your program might be asleep, but everything else is fully awake and operable. Your language interpreter, your runtime, your connected sensors, most importantly your CPU and your WiFi connection.

This is where 'deep sleep' comes in. If I put my controller described above into deep sleep, it needs about **5µA** to keep running and wake up after a certain amount of time. This small change leads me to having a battery that lasts 6 days before I need to charge it. Sounds sweet? Think deeper.

## How Often Do I Need To Sense?

In the wild internet, you will find people that solder out power LEDs and several other components off their boards. This is _very_ nice and shows that you can really cut your boards to size to match your needs. In my case, I want a stock board that will be capable of stock things when my future me is not anymore interested in measuring basement temperature and humidity.

Thus, this post is not meant for bleeding edge tinkerers, but extending battery life with on-board software.

I came to the decision that it is enough to update my basement IoT dashboard every 15 minutes. Guess what? This leads to extending battery life to **3 years**. Yes, you read correctly.

## How can I calculate this?

This one's simple. You first need to measure what your board needs when running. Buy a cheap multi-meter, they often measure sufficiently exact what it needs. Round that number up for your first calculation. Then, look up a data sheet for your board and find out what it needs when deep-sleeping. For ESP8266 controllers, it's about 5µA.

Then you need your battery capacity. I need 2 batteries to get 3V, which unfortunately keeps me on running with 1733mAh. This means, the battery could power a device that consumes 1733mA for one hour.

The calculation I did was simple _rule of three_. Find out what is consumed in "one scenario" which takes one time span. Calculate this time span up to one hour. Then, divide your battery capacity by the capacity needed in one hour by your device. Voilá, you now know how many hours you can run without changing batteries.

## Sum Up

Use deep sleep! Often, you don't really need to know every second / every minute what your sensors sense. Also, think about if you really need WiFi. There are very neat standards that utilize lower frequencies which consume much less energy than WiFi.

Also, and I couldn't emphasize more, re-think and re-design your code. If your first implementation works, that's fine. It works for the breadboard and it _could_ possibly even run over a longer time span. But seriously, no one writes perfect code on the first attempt. There's always possible tweaks that can be made in software.
