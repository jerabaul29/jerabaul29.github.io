---
layout: post
title: "A note about RPi power supplies"
date: 2022-03-19 12:00:00 +0200
categories: jekyll update
---

Raspberry Pis are nice little machines, that are fully able to run production workloads home. For example, a RPi4 can be used as a gateway, a home file server, the host of some personal Nextcloud instance, a data backup server, etc. I use a few RPi4s at home, together with cases that allow to connect a SSD drive to them (in particular, I like well the "Argon ONE M.2", which I have coupled with both "WD Green M.2 240GB Internal SSD", and "WD Red SA500 M.2 2TB SATA SSD", both working fine). The only issue I have had so far running these, is power supplies.

## Getting a power supply strong enough

RPis need 5V, and are quite picky about the power supply quality: too much ripple or transient voltage drop, and the RPi may brown out or misfunction. This is especially an issue when using the more powerful RPi models (at the present time, the RPi4), in a case equipped with a fan, connected to an external SSD and a few peripherals. The tricky thing there, is that even "powerful" USB-C chargers you may have lying around home may not be adapted. Indeed, the USB-C chargers used for laptops and phones typically can deliver quite high power, but at an elevated voltage, not at 5V.

For example, when using a high power USB-C charger to charge a phone, the phone and the USB-C charger will "discuss", so that the phone will tell the USB-C charger that it can "accept up to 20V". In this case, the charger may provide, for example, up to 65W: around 3.25A at 20V. By contrast, the USB-C charger may be limited to providing a much lower level of power when limited to only 5V, for example, at most 1.0A and 5W at 5V.

The situation may be even worse when considering voltage losses in the cables between the charger and the RPi: as the RPi draws some current, there will be some voltage fall in the wires due to the Ohm law. This may not be an issue for a laptop pulling 65W at 20V, as the laptop will anyways need to regulate the 20V to something more usable. But, if a charger is set to provide 5V, and the RPi draws 1A, one will easily end up with an effective voltage of 4.9 or 4.8V on the RPi side - low enough to be unreliable. Therefore, many "dedicated RPi power supplies" actually provide more than 5V (typically 5.1V or 5.25V, sometimes as high as 5.35V), in order to remain above 5V on the RPi side, even with voltage drop in the wires.

## Field tests

I have found out that, when powering my "standard" file backup RPi server, which is assembled with i) an Argon ONE M.2, ii) a WD Red SA500 M.2 2TB SATA SSD:

- using the 65W high-power USB-C charger of my laptop does not work reliably; sometimes the RPi does not even manage to boot, and sometimes it manages to boot but hangs randomly later on. This is not so surprising: looking at the datasheet of the charger, it should be able to provide up to 1.2A at 5.0V. This means that, as soon as the RPi, case, and SSD draw a bit of current, the voltage given to the RPi will fall well below 5V, making operation unreliable.

- using the "RPi4 official power supply" gives reliable operation: this supply gives, according to the datasheet, up to 3.0A at 5.1V: enough intensity to fulfill the needs of the RPi, case, and SSD, with a voltage slightly over 5.0V so that it will remain outside of the brownout domain, even with a bit of voltage drop over the wires.

However, I suspect that even the "RPi4 official power supply" leads to situations that are a bit at the limit of brownout during boot, when the fan goes at maximum speed, or if I add a couple of peripherals. I want to buy and test the Argon ONE Raspberry Pi 4 power supply when I can get my hand on one of them with a EU power plug. This charger provides, according to datasheet, up to 3.5A at 5.25A output voltage, which should be more than enough to remain out of the brownout domain for all operation conditions.

