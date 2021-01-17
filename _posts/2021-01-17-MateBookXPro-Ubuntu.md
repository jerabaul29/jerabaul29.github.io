---
layout: post
title:  "MateBook X Pro 2020, Ubuntu 20.04"
date:   2021-01-17 12:00:00 +0200
categories: jekyll update
---

I recently changed my main private PC. I am usually a Lenovo Thinkpad fanboy, but my local store had some really good sales on the MateBook Pro X (MBX) 2020 (with core i7-10510U, 16GB RAM, and 1Tb SSD), so I decided to go for it. I use exclusively Ubuntu LTS (so 20.04 at the time of writing this), and I must say that this is the best laptop I have ever owned, and that the Ubuntu experience on it is perfect, including some nice features like battery charge thresholds.

I did not find much "field reports" for this laptop, so I report my experiences setting up Ubuntu 20.04 here!

## Installing Ubuntu

Installing Ubuntu is very easy, business as usual:

- I used an old Ubuntu machine I had to create a bootable USB key (using the latest AMD 64 image for Ubuntu 20.04 LTS).
- I "installed" Windows without registering or setting up internet, and started Windows.
- I plugged in the USB Ubuntu stick.
- On Windows, I searched for "boot", and selected the "booting from USB drive" option.
- At this stage, Windows rebooted and I was able to select the USB Ubuntu key as the boot volume.

After that, it was the usual Ubuntu installation experience. A few notes:

- I used only Ubuntu and removed Windows altogether.
- I set a BIOS password.
- I encrypted the SSD drive.
- I installed third party software, since the MBX has a dedicated nVidia graphics card.

Everything worked flawlessly.

## Laptop setup

I did a few changes to the setup of the machine:

- The nVidia card uses quite a lot of power on Ubuntu, which makes the laptop hot if running all the time. To avoid that, the solution is to only activate the GPU when needed. This can be done by:

    - Opening nvidia X Server Settings
    - Changing to NVIDIA On-Demand and reboot

This works out of the box: now my machine runs nice and cold, without the fan being active at all for casual use. As a note, the GPU is fully recognized, and the output of the  ```nvidia-smi``` looks good. Without any more power tuning, I get about 10 hours of battery in casual use.

- The most amazing thing from my point of view is the possibility to set up battery thresholds out-of-the-box, without any tweaking requested. To increase my battery life, I like to limit it to 65% charge level at maximum, and to start again charging only when the level hits 50%. This avoids the "dead battery for always plugged in laptop" symptom. On all laptops I had owned previously, this was either impossible to set on Ubuntu 20.04, or needed to install additional software (tlp for Thinkpads). But everything is ready-to-use on the MBX with Ubuntu 20.04:

    - I can set the maximum charge by writing it through: ```sudo vim /sys/class/power_supply/BAT0/charge_control_start_threshold``` and simply writing the max charging in percent (i.e. 65).
    
    - I can set the start charge threshold in the same way by writing to ```sudo vim /sys/class/power_supply/BAT0/charge_control_start_threshold``` and writing the threshold (50).
    
This is the first time I get my hand on a laptop where this works out of the box on Ubuntu!

## Ubuntu customizations

I also performed the usual Ubuntu customizations, like auto-hiding of the top bar and task launcher, changing to dark theme, etc. Just a few notes, as it is always a bit annoying to need to spend time googling for the solutions to these customizations:

- To get the tiled desktops, I installed the ```Workspace Matrix``` extension.
- To turn the App launched into a Dash with much more customization possibilities I installed the ```Dash to Dock``` extension.

Setting parameters for both extensions already improves things quite a bit. In addition, I needed a few tweaks to get all auto hiding to work as I want, in particular for the top bar. This requested:

- ```sudo apt install gnome-tweaks```
- ```sudo apt install gnome-shell-extension-autohidetopbar```
- setting the parameters in the gnome-tweak applet
- restarting gnome (```Alt-F2``` and then executing the command ```r```).

## Conclusion

These few steps provide an amazing, sleak Ubuntu 20.04 laptop. Without doubt the best machine I have ever experienced.
