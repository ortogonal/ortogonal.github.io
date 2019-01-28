---
layout: post
title: "Lose weight with Qt"
description: "Custom written post descriptions are the way to go... if you're not lazy."
tags: [qt, c++, cycling]
modified: 2018-10-10
image:
  path: /images/lose-weight.jpg
  feature: lose-weight.jpg
  credit: Zwift
  creditlink: http://www.zwift.com
---

> This post first appeared on my LinkedIn page.

So is it really possible to lose weight using Qt - a cross platform application C++ framework? Well - yes!

I've been programming computers since I was ten years old. I loved it from day one and still does. For the past ten years Qt, QML and C++ have taken up quite some time. But in the same time my bike has taken up quite some time as well. I've rode my bike from Jönköping-Sweden down to Paris-France to raise money to the Swedish Child Cancer Foundation. I've tested my legs on the horrible cobbles at Paris-Roubaix (aka. Hell on North). I even tested one of the toughest stages of Tour de France when I finished Etape du Tour.

But now how to lose weight using Qt... That last year my training motivation has been - let's say not at the top... So... Hmm... How to boost the motivation... What if I connect my indoor bike to a computer game and start racing there! Everything connected to a computer motivates me...

Zwift is a fast growing computer game and social network where you connect your smart trainer/indoor bike and use the power meter to race against other riders in a virtual world. If you push really hard on your bike your "watts" will increase and you go faster. If you position your self behind another rider you can pedal a bit easier due to the reduces air resistance in the virtual world. Only one problem. How can I connect my indoor bike to Zwift? I've already spend a bit to much money on my indoor bike and it's not possible to connect to a computer, but it can measure power really simple. Buying a new was not an options.

Well a Raspberry Pi 3, an magnetic sensor/switch, an accelerometer and a really simple Qt application is what I need.

![Monark test bike]({{ site.url }}/images/monark.jpg)
{: .image-right}

I use a Monark 828e indoor bike. It's stable, massive and feels really like a real bike does. Calculating the power from this bike is simple. You multiply your cadence with the value of the pendulum scale (the metal gray round thing on the upper right part of the wheel). When I increase the resistance the pendulum will raise to a higher value on the scale. Measure the cadence is simple - a magnetic sensor/switch and a magnet on on of the pedals. Connect the switch to a digital I/O on the Raspberry Pi with a pull-up resistor and you're done. Measure the angle of the pendulum is a bit trickier. But placing a small accelerometer on the pendulum makes it possible to calculate the angle and calibrate it against the scale on the bike. The accelerometer is connected to the Raspberry Pi using I2C. So now I have everything needed to calculate the power.

Now I only need to send it, using Bluetooth Low Energy, to my iPad where the Zwift game is running. And this is were Qt solves my problem really easy. Using Qt Bluetooth LE API is really simple. The first BLE server example that Qt provides is a heart rate monitor. I just changed that example so instead of using the BLE HeartRateService it uses the CyclePowerMeter service. When I had my hardware setup and connected it took about two hours of programming and calibration to get everything up and running! The hardest part was understanding how the data in the BLE message should be packaged.


And now with this up and running I'm connected to the virtual training world and can start pushing the pedals harder and harder. So is Qt the makes me lose weight? - Well that's up to you to decide.

The code is available on GitHub.

- [Buildroot repo for RPi3](https://github.com/ortogonal/buildroot-monark)
- [Source code for Qt application](https://github.com/ortogonal/bluepower)

And what about motivation? - Well two hours of hard work yesterday
