---
layout: post
title:  "GPG and smartcards: a few tips"
date:   2021-05-16 12:00:00 +0200
categories: jekyll update
---

*This post is a work in progress, and I complete it as I need to use GPG and smartcards on new computers.*

## Motivation for using GPG and smartcards

- take private key with me in a safe way; no way to steal it
- password manager
- safety
- encrypt / decrypt without the need to trust the computer on which I am

## General setup

I used the following setup:

- maybe not perfect but fine enough
- my goal: multiple copies of the key on USB dongles and saved on a RPi / USB dongle
- take my key with me on smartcard

- use RPi offline to generate private / public key, passphrase protected
- export it to file, as migrating a key to smartcard wipes from computer each time...
- generate several smartcards (2)
- store RPi / USB with copy inside in safe

- in combination with password-store
- store the encrypted keys on github
- use on any machine even if I do not trust it

## Using on a new computer

- gpg --import public_key
- gpg --card-status

Problem: on many slightly old versions of gnupg, can only use 1 smartcard at a time; what if several smartcards, and have another one than last time with you? Need to erase the old smartcard and register the new one.

- gpg --list-keys; destroy the smartcard key if another smartcard with delete command
- copy the public key
- import it jj@MBX20:~/Desktop/Jean$ gpg --import public_key 
- gpg --card-status

