---
layout: post
title:  "Installing packages starting from a non admin account by first changing user: linux trivia tips"
date:   2021-05-17 12:00:00 +0200
categories: jekyll update
---

This is a "linux trivia and tips" post. A (probably good) idea on shared machines (for example, with other family members, some of whom may not be well known to linux) is to have user accounts for which the admin rights are disabled. This way, users cannot use ```sudo``` and make core changes to the system. However, this is also sometimes a bit painful - let's say, if I need to install some package for a user, and they just hand me in a laptop that is logged from their session; there is no way to directly ```sudo``` and fix things, and I do not want to go through the hazzle of using the GUI to change user and log on my admin account.

Luckily, there is a simple (linux trivia) solution: first change user in a terminal to get a single "admin user" terminal, and then to sudo from there. So something like:

```bash
jj@MBX20:~$ sudo su
[sudo] password for jj: 
jj is not in the sudoers file.  This incident will be reported.
jj@MBX20:/home$ su admin_jr
Password: 
admin_jr@MBX20:/home$ whoami
admin_jr
admin_jr@MBX20:/home$ sudo su
[sudo] password for admin_jr: 
root@MBX20:/home# whoami
root
```

And here we are, now being root and able to alter the system, install packages, etc.
