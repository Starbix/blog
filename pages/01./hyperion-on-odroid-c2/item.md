---
title: 'Hyperion on Odroid C2'
date: '09-10-2017 18:29'
taxonomy:
    category:
        - blog
    tag:
        - 'Odroid C2'
        - software
        - hardware
visible: true
---

I wrote this tutorial because Hyperion doesn't support Odroid C2 out of the box and I didn't find an easy tutorial for it. I'm using LibreELEC, but with minor changes it'll work on any other OS.
#### Requirements
- Odroid C2
- Arduino Uno
- APA102 LED stripe
- USB type B to USB type A cable
- Jumper cables
- 5V Power Supply

### Introduction
I'm not going into how to create the Hyperion config and how to install the LEDs on your TV. There are already a lot tutorials for that, like [Awesome Pi](http://awesomepi.com/diy-breath-taking-ambilight-for-your-own-tv-raspberry-pi-2-tutorial-part-1/). Instead of the WS2801 LEDs I recommend the APA102 LEDs which have better colors and the cabling needs to be different as we're going to use an Arduino between the LEDs and Odroid.

Now, there are two problems with the Odroid C2. The first one is, that it doesn't support SPI, a serial communication interface used to communicate with the APA102. This can be solved by using an Arduino between the Odroid and the LEDs.
The second problem is, that there's no official hyperion binary for the Odroid C2. Which meant that I had to compile it on my own, but more on that later.

### Hardware
First you need to prepare your Arduino. For that you need to download the Arduino IDE from [here](https://www.arduino.cc/en/Main/Software).
After you've installed the Arduino IDE, you need to select your Board Type under **Tools>Board** where you select Arduino/Genuino Uno and just below you also need to select the serial port for the Arduino, this can change from computer to computer.
You then need to paste the code from [here](https://raw.githubusercontent.com/hyperion-project/hyperion/master/dependencies/LightberryHDUSBAPA1021.1/LightberryHDUSBAPA1021.1.ino) into the IDE and then upload it to the Arduino.
You can then plug the LED pins into the Arduino with jumper cables. SPI data out is digital pin 11 and clock is digital pin 13. If you want to test if you've connected it correctly you can also uncomment the test pattern part. You should then see a red, green and blue flash.

### Software
Now comes the interesting part. 

So you don't have to compile Hyperion for yourself, I have a [Github Repository](https://github.com/Starbix/hyperion-files) with precompiled binaries. If you don't trust me, you can also compile them yourself. I also have a slightly modified [Gtihub Repo](https://github.com/Starbix/hyperion) for that, as it won't compile otherwise. You need to use this command: ``cmake -DENABLE_DISPMANX=OFF -DENABLE_SPIDEV=OFF -DENABLE_AMLOGIC=ON -DCMAKE_BUILD_TYPE=Release -Wno-dev ..``

Install [LibreELEC](https://libreelec.tv) on your Odroid.

In order to install Hyperion on your Odroid you need to make sure to **enable SSH**, you might need to restart after that. Then you need to connect to it over SFTP. For that I use Cyberduck, but any other SFTP client should work. Navigate to `/storage` and create a `hyperion` directory.

To get the files you need to put into the `hyperion` folder, navigate to my [precompiled binaries](https://github.com/Starbix/hyperion-files). There are different folders for different binaries. You need to use the **hyperion (v1) odroid-aml patched**. Upload its content to the previously created `hyperion` directory. Make sure you delete my config and replace it with **your config**.


After uploading the binaries, the effect folder and **your own config** you need to SSH into your Odroid. I use iTerm2 for that, but any \*nix based OS can do that from its terminal. Log into it with password `libreelec`
```
$  ssh root@IPADDRESSOFTHEODROID
$  chmod +x hyperion/bin/*
``` 

To make Hyperion work with HEVC/H.265, you need to execute `echo 1 | tee /sys/module/amvdec_h265/parameters/double_write_mode` which can decrease performance, so if there are alot of dropped frames you can just remove that line for the autostart.sh. But you'll get pastell colors if you don't enable it.
##### Autostart
To start Hyperion on boot, you need to create autostart.sh in /storage/.config with the following content.
```
#!/bin/bash
echo 1 | tee /sys/module/amvdec_h265/parameters/double_write_mode
(
	/storage/hyperion/bin/hyperiond.sh /storage/hyperion/config/hyperion.config.json
) &
```
You can use `nano /storage/.config/autostart.sh` for that. Afterwards you need to execute `chmod +x /storage/.config/autostart.sh`