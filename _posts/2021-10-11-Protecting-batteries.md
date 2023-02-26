---
layout: post
title: "Protecting batteries"
date: 2021-10-11 19:00:00 +0200
categories: jekyll update
---

Batteries are everywhere in modern devices. Given how good electronics have become, and the switch from rotating hard drives to solid state drives, batteries have become one of the components that often are first to fail in laptops, smartphones etc. A very unfortunate and common symptom is the "laptop always plugged in power supply ending up with a dead battery after a couple of years" symptom. This comes from the nature of modern lithium batteries. These lithium batteries:

- do not like remaining at a high storage percent for a long period of time (i.e., Li batteries degrade over time when kept at high charge levels),
- do not like being discharged to a low percentage and kept there for a long period of time,
- do not like low or high temperatures,
- do not like high charging currents.

By "do not like", we mean that any of these points can damage a lithium battery over time. So, in order to keep a lithium battery healthy and to allow it to keep its capacity, it is best to keep its charge between typically 40% and 75%. It is also better to not charge it with high current if possible (i.e. using a low power charger). Unfortunately, the typical use case of laptops used as main machines, being always plugged in, and quick charged the few times they are back from a travel, is at the opposite of preserving battery life. Also, these advices are quite the opposite of what was actually good for NiMh and NiCd batteries, so older advices and habits people may have from these older technologies need to be phased out when shifting to Li batteries.

Fortunately, it does is possible to enforce all these points and to preserve the health of a battery by applying a few rules:

- limiting charge current,
- limiting max battery charge by putting a max charge threshold,
- avoiding battery charge hysteresis (i.e., waiting until there has been significant battery discharge down from the max charge threshold before starting to charge again),
- avoiding to fully discharge the battery by switching off the computer before reaching a min charge threshold.

In the following paragraphs, we discuss quickly how to apply each of these measures. If you (like me, and most people I know) primarily use your laptop battery as a built-in UPS, or a small buffer to go from one meeting room to the other, then the points below will considerably extend your battery life length.

## Preserving battery health in day to day use

# Setting charge thresholds

There are several tools available for 1) setting a maximum battery threshold, 2) avoiding hysteresis, i.e. waiting for the battery to have lost a significant charge from the maximum threshold before trying to charge again. One very useful tool for this (in particular for Lenovo Thinkpads, but also a few extra brands have been added lately) is to use TLP, see in particular:

- [tlp faq about battery health](https://linrunner.de/tlp/faq/battery.html),
- [the Arch Linux tlp page](https://wiki.archlinux.org/title/TLP),
- [the tlp github repo](https://github.com/linrunner/TLP).

This will let you set a maximum battery charge threshold after which the battery will stop charging (typically 75% for example), and a start charging threshold that the battery level needs to fall below before starting charging (typically 65% or 70% for example).

If you do not want to use tlp, or if your laptop model or brand is not supported, there is also increasing support for battery charge thresholds natively in Linux and modern linux distros. For example, for modern distros like Ubuntu 20.04, it is possible to set battery thresholds for a wide range of models by editing the config files (battery name(s) and number may depend on your machine hardware):

- ```/sys/class/power_supply/BAT0/charge_control_end_threshold```,
- ```/sys/class/power_supply/BAT0/charge_control_start_threshold ```.

Note that, at the time of writing this post, these settings get erased at each machine startup, so the way to get this activated automatically is to add for examle a small crontab entry:

```bash
~$ sudo crontab -e
@reboot sleep 20; echo 70 > /sys/class/power_supply/BAT0/charge_control_end_threshold 
@reboot sleep 30; echo 55 > /sys/class/power_supply/BAT0/charge_control_start_threshold 
```

# Preventing battery deep discharge

While the charge threshold control limits how your battery charges, you should also avoid deep discharges and limit how your battery discharges. For this, the best (even if a bit brutal) solution is to display a low battery warning, and shut down your computer, at a much higher battery level than what is set by default by the manufacturer. There are several methods for doing this:

- The most common tool in use for displaying warning and forcing a low battery shutdown on modern Linux distros such as Ubuntu 20.04, is UPower. UPower can be configured through its config file available at ```/etc/UPower/UPower.conf```. You may edit this config file to get custom behaviour, for example displaying a low battery warning at a percentage of 30%, and shutting down at 25%. Note that, once you have update the config file, you will need to reboot your machine for your changes to take effect. Something like this should work fine for example:

```bash
[...]

UsePercentageForPolicy=true

[...]

PercentageLow=35
PercentageCritical=30
PercentageAction=25

[...]

CriticalPowerAction=PowerOff
```

- If you really want to hack something yourself and to avoid any risk that your battery discharges too low following for example a UPower update that may replace and erase your change to the config file, you can always write a small script to do this yourself. I have a small example [at this location](https://github.com/jerabaul29/config_scripts_snippets/blob/main/scripts/power_management/script_shutdown_low_battery.sh) that can be set up with the cron and can check for low battery level while AC is plugged off, and shut down the computer following some threshold you can set. Note that this works fine on my machine, but if your machine is different, you should check (and adapt if needed) that the functions reporting AC state and charge level work for you.

Also note that, with both methods, your computer will "brutally" shutdown, and you will loose unsaved data, so be careful when running low on battery!

# Limiting charge current

It is also quite easy to limit the charging power / speed that you apply on your battery. As USB-C is becoming a standard across laptops and electronic devices, it is possible to just buy a charger with a lower power rating than the factory stock charger. For example, my latest laptop comes with a 65W charger, but a 45W or even a 25W charger works just fine - and there is quite little extra power available for fast charging the battery too hard. Just make sure that you use a USB-C power supply that is able to supply enough power to sustain full power use of your laptop, otherwise your battery will discharge when you use your laptop even plugged in the charger.

## Getting full battery capacity available when needed

Of course, if you do want to use your battery to the full of its capacity because you are on a travel or else, you can perfectly well decide to charge it up to its max capacity and discharge it to a low level. Most modern batteries will have some form of margin built in anyways (i.e., there should be already a bit of "buffer" enforced by the firmware of your battery that prevents it from truly fully charging or discharging, even if it looks so from your computer). For getting full capacity for one cycle, if you use tlp, you can use the ```fullcharge``` command, and you can update either the UPower config or the values in the low battery shutdown script. You can also update the value of the crontab job setting the charge control thresholds if you are using this method (a machine reboot may be needed for the changes to take effect).

## Other devices

These advices apply to all Li batteries, also outside of laptops, either the device is a smartphone or an electric car or any other Li battery-powered device. You can significantly extend battery life (by a factor up to 20 according to [this blog post](https://chargie.org/blog-list/page/2/)) by applying these advices. In the case of your car, avoid supercharing when you can and set charge thresholds or manually disconnect your charger when you hit 80%. In the case of your smartphone, avoid fully charging and discharging, and avoid using the most powerful chargers. You can also consider using a small smart dongle to enable smart control of your smartphone, for example the very convenient [chargie smart dongle](https://chargie.org/). I have a number of older Li batteries (in particular in some laptops that are 8+ years old) that are still perfectly fine, so these advices do work in practice.

