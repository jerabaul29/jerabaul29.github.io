---
layout: post
title: "How to create a Windows 10 bootable USB key from Ubuntu"
date: 2021-11-06 19:00:00 +0200
categories: jekyll update
---

*Notes: 1) the content of this post may loose in relevance over time; for example, for now, i) Windows 10 is the goto Microsoft OS, and ii) Microsoft is providing a free iso for it. Both points may change. Similarly, 2) the optimal methods of creating bootable drives have changed in the past, and may change again.*

I am using Linux in form of the Ubuntu distro (which I really love) for all my day-to-day work and digital activity, but there is one single case where I need to run Windows: to play a good old game of Age of Empires II once a year with some friends :) . The challenge then is to get my hand on a computer running Windows 10. Luckily, I have some slightly old but still perfectly functioning and quite powerful laptops lying around home, and astonishingly enough, Microsoft has started to provide free Windows 10 isos. It looks like these are full-featured isos (though they can be used in "unregistered" mode without a license key, i.e. for free as in beer), and I have not met any restrictions so far for my (very limited) use. I guess Microsoft is understanding that providing their OS for free is a better model than making it painful to use and update Windows. So, all I need is to set up a bootable Windows USB key, install Windows 10, play a few games, and if I need the laptop for something else, wipe out Windows with Ubuntu when I am done or dual boot it.

This is all fairly easy to do and simple, but I had one point of mild difficulty that took me a bit of time to sort out well, namely, how to best set up a bootable Windows 10 USB key from Ubuntu. So this post is just here to document the simplest way I found (note that this is only my opinion, maybe there are some even simpler methods) for setting up a bootable Windows 10 USB key from Ubuntu.

The process starts of course by downloading the iso for Windows 10 in the right version, in my case [from this page at Microsoft](https://www.microsoft.com/en-us/software-download/windows10ISO). I needed the 64 bits version since I have a recent 64 bits processor. Once a few options are chosen, the download of the iso (over 5GB, this is seriously bloated as expected...) can start. An important step is then to check the checksum of the downloaded iso, which can be obtained from Microsoft [in my case, from the same webpage](https://www.microsoft.com/en-us/software-download/windows10ISO). Checking the iso is business as usual (of course the checksum will change when Microsoft updates the iso):

```bash
$ echo "6911E839448FA999B07C321FC70E7408FE122214F5C4E80A9CCC64D22D0D85EA *Win10_21H1_English_x64.iso" | shasum -a 256 --check
Win10_21H1_English_x64.iso: OK
```

At this stage, we have a Windows 10 iso which has been verified, and we are ready to create a bootable USB drive. The drive will naturally need to be bigger than the iso (in my case, a bit over 5GB). The simplest way I found to "burn" the iso on the drive together with a bootloader is to use a third-party Open Source tool called [Ventoy](https://github.com/ventoy/Ventoy). I did not find an ```apt-get``` way to install it, but the instructions are fine enough to follow:

- get a copy of the ```.tar.gz``` from their [release page](https://github.com/ventoy/Ventoy/releases/).

- check the integrity of the sources against the sha256sum they provide at the same location (as usual, the exact digest will change as they update their code):

```bash
$ echo "f4b3fabfbc3d635aa8b3ecf6fc424f1d1fbcb658daa495abef52ea8eb10057aa *ventoy-1.0.58-linux.tar.gz" | shasum -a 256 --check
ventoy-1.0.58-linux.tar.gz: OK
```

- untar:

```bash
$ tar -xf ventoy-1.0.58-linux.tar.gz
```

- run Ventoy with Web GUI as root:

```bash
$ cd ventoy-1.0.58/
$ sudo ./VentoyWeb.sh 
[sudo] password for jr: 

===============================================================
  Ventoy Server 1.0.58 is running ...
  Please open your browser and visit http://127.0.0.1:24680
===============================================================

################## Press Ctrl + C to exit #####################
```

At this stage, opening the provided IP address in browser provides a simple GUI that allows to choose which device should be processed by Ventoy to make it bootable. In my case, this is the USB thumb drive that I had inserted, and its name was as expected within the list of possible devices. Runny Ventoy on it will simply set up a bit of partitioning and write a few files, with a small bootable partition being be set-up, and a separate partition with the remaining space being created.

Once Ventoy is finished preparing the drive, all what is needed is simply copy as many isos as wanted into the partition dedicated to hosting the isos. If several isos are present, a menu will let the user choose which one to boot when the machine starts from the Ventoy drive. In my case, I just want to set up the Windows 10 iso. For this, I:

- find out which partitions are available on my USB drive (note that in my case the USB drive is mounted as ```/dev/sda```, but of course you may get a different name depending on what is plugged in your machine already):

```bash
$ sudo fdisk -l /dev/sda
Disk /dev/sda: 28,64 GiB, 30752000000 bytes, 60062500 sectors
Disk model: Ultra           
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x990d04c0

Device     Boot    Start      End  Sectors  Size Id Type
/dev/sda1  *        2048 59996959 59994912 28,6G  7 HPFS/NTFS/exFAT
/dev/sda2       59996960 60062495    65536   32M ef EFI (FAT-12/16/32)
```

You can see here the small bootable partition ```/dev/sda2```, and the larger partition that is here for receiving the isos, ```/dev/sda1```.

- copy the iso:

```bash
/dev/sda1 $ cp ~/Download/windows10_images/Win10_21H1_English_x64.iso
```

And here we are, with a bootable Windows 10 USB key burnt from Ubuntu and ready to use :) . At this stage, the only steps that remain are i) inserting the drive inside the machine that should be installed with Windows 10, ii) go through the BIOS and make sure that the machine should boot from the drive, iii) boot from the drive, and select the Windows 10 iso, iv) install Windows 10 (and this is by far what takes the most time: there are many steps, you will need to have a Microsoft account and use it to set up the machine, and many cycles of reboots will happen... but in the end, it works out well).
