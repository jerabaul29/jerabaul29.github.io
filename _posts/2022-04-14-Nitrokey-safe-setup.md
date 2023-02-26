---
layout: post
title: "Safely generating, setting up, and using private keys on (Nitrokey) hardware security tokens"
date: 2022-04-14 15:00:00 +0200
categories: jekyll update
---

*disclaimer: this post is written to the best of my abilities, and I use this solution myself, but I am not a specialist on the topic of cryptography and computer security; **I take no responsibility** if any data are compromised despite / while following the instructions here.*

GnuPG (GPG) is a common tool for encrypting, decrypting, and signing files. GnuPG is open source, and the algorithms and implementations it uses should be as good and reliable as they can be (and very safe as long as no quantum computer is available). Therefore, the main risk when using GPG is with the protection of the private key. A first (and commonly used) "line of defense" is to use a strong passphrase for protecting the private key. Without the passphrase, the private key cannot be decrypted and used. However, even using a strong passphrase, there are still some risks associated with using a GPG key, if the host computer is compromised for example. In particular, the user of a GPG key hosted directly on their computer will need to i) enter the passphrase on the computer, so that ii) the key will be present, decrypted, in computer memory. This means that an attacker who compromised the PC, can have full access to the private key once it is stored, decrypted, in memory. Moreover, there is an issue regarding the distribution and securing of the encrypted GPG private key. Even if protected by a passphrase, a GPG private key is a sensitive piece of information, as the passphrase can be bruteforced, and it is better not to distribute it through unsafe channels, and it should be treated like sensitive information - making distribution, for example across machines, inconvenient.

A solution for this is to use a hardware security token ("token" in the following). These usually take the shape of a USB key, and contain the private GPG key protected in a smartcard or similar "safe" microcontroller that is built to resist software and hardware attack. Once loaded on the token, the private key cannot be extracted, neither through software nor hardware (if the device works as it should and has no vulnerabilities, at least). When using the token, the user has to enter a pin code (typically, entering the wrong pin code several times erases the key contained and resets the key to avoid bruteforce attacks) to unlock the private key, and once this is done, the token itself can perform decryption without the private key ever leaving the token. Therefore, i) the private key never leaves the token, ii) if the smartcard works as intended, it is not possible to bruteforce it, even with physical access. In case the pin code is entered from the computer to which the token is connected, the pin code is potentially compromised, but exploiting this still requires stealing the token, as the key itself is never available on the computer. In the case the token is equipped with a small keyboard used to enter the pin code, the only thing that can be compromised in cased the computer used is compromised is the specific file that was decrypted. As long as the token is working as expected, there is no way to exploit the information extracted from a compromised computer from which the token is used to compromise further the private key (ie to decrypt / sign other files) at all in the case of a token equipped with a keyboard, or without getting the token itself if not equipped with a keyboard. This is quite an improvement over the case where the computer is used to store and decrypt the GPG key.

There are a few technical setup hurdles for using tokens, though. In particular, if you want to be sure that you will always be able to decode files encrypted with your private key, you will need a backup of the key / token. As keys stored on a token can usually never be extracted again, this means that you should actually generate the private key on a secure computer, back the private key up (protected by a really, really strong passphrase), and then upload it on one or more tokens. 

In the following, I describe exactly how I solved this problem myself. I explain i) how to use a Raspberry Pi microcomputer (RPi, though any other computer would be possible to use in theory) that is never connected to the internet after it is set up in order to generate a private key, ii) how to back up the key on the RPi, and load it on one or several Nitrokey tokens. Once this is done, one or more tokens containing the key are available (for example, one can be attached to your keyring, and the other kept as a backup home), and the RPi containing the passphrase encrypted key can be used as an additional backup of the key. I personally keep the RPi in my safe and never connect it to the internet after it has been used to generate the keys, so that there is no way for the encrypted private key to be leaked and attacked.

## Preparing the Raspberry Pi: making sure you can interact with the key

I will assume that a RPi is used to generate the private key and load it on the tokens. For my part, I like best to use a RPi 2, as it has no built-in Wifi or Bluetooth, so that it is very easy to know that the networking and internet access is turned off (unplug the Ethernet cable / Wifi adapter) but you can use a newer RPi if you want - just make sure that you know how to turn off the Wifi and Bluetooth. Of course, a different computer could be used, but the interest of the RPi is that it is cheap enough that it can be used as a "single use computer" specifically for generating and backing up the keys, never be connected again to a network once it has been used to generate the keys, and stored in a safe. If you are a bit less "obsessed" about safety, you could of course use a normal computer to perform these steps.

I will not go through the details of how to set up a Raspberry Pi, prepare the SD card, boot it, etc: there are many good tutorials online for this. For now, let us just assume that you have a RPi booted and ready to use, connected to a mouse, keyboard, and screen, and the internet. Do not use a wireless keyboard or mouse, so that there is no way sniffing your inputs to the RPi.

At this stage, there is a bit of setup to do on the RPi. First, there is a package that needs to be installed in order to be able to communicate with gpg smartcards:

```bash
$ sudo apt update
$ sudo apt upgrade
$ sudo apt-get install scdaemon
```

Second, in order for the smartcard to be recognized and used without root priviledges, a couple of udev rules are needed. In my case (I use a Nitrokey Pro 2, you will have to find the instructions corresponding to the token you use), the instructions are available [here](https://docs.nitrokey.com/software/nitropy/linux/udev.html) and the udev rules can be set up by:

```bash
$ wget https://raw.githubusercontent.com/Nitrokey/libnitrokey/master/data/41-nitrokey.rules
$ sudo mv 41-nitrokey.rules /etc/udev/rules.d/
```

For these changes to take effect, you should either reload the udev rules:

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

After these steps, you *should* be able to plug the token in the RPi and to get it recognized by GPG without ```sudo``` (*note*: if you have set up the udev rules but cannot see the key, typically getting a ```gpg: selecting card failed: No such device gpg: OpenPGP card not available: No such device``` message when issuing ```gpg --card-status```, try rebooting your computer for the udev rules to take effect; it also seems that on some RPis, the udev rules do not work correctly; in this case, just run the gpg commands for interacting with the smartcard with the ```sudo``` keyword first, i.e. ```sudo gpg --card-status```; this will apply for all following gpg card control commands), typically:

```bash
$ gpg --card-status
Reader ...........: 20A0:4108:000000000000000000009916:0
Application ID ...: D2760001240103030005000099160000
Application type .: OpenPGP
Version ..........: 3.3
Manufacturer .....: ZeitControl
Serial number ....: 00009916
Name of cardholder: [not set]
Language prefs ...: de
Salutation .......: 
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 64 64 64
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```

This shows that your RPi is ready to be used for generating the keys! At this stage, you are ready to generate, back up, and load keys, and you should disconnect the RPi from any network, turn off the Wifi and Bluetooth on it, and consider the RPi as sensitive material.

## Preparing the token: setting pin codes, choosing strongest key settings, personalizing

You can prepare the token by setting up the user and admin pins (do not loose these!). The default for all gpg smartcards coming out of factory will be ```123456``` for the user pin (that has to be at least 6 characters long), and ```12345678``` for the admin pin (that has to be at least 8 characters long). Of course, both should be changed to strong pins. Follow the instructions that appear for setting the pins, and google for more information:

```bash
$ gpg --edit-card 

Reader ...........: 20A0:4108:000000000000000000009754:0
Application ID ...: D2760001240103030005000097540000
Application type .: OpenPGP
Version ..........: 3.3
Manufacturer .....: ZeitControl
Serial number ....: 00009754
Name of cardholder: [not set]
Language prefs ...: de
Salutation .......: 
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 64 64 64
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]

gpg/card> help
quit           quit this menu
admin          show admin commands
help           show this help
list           list all available data
fetch          fetch the key specified in the card URL
passwd         menu to change or unblock the PIN
verify         verify the PIN and list all data
unblock        unblock the PIN using a Reset Code

gpg/card> admin
Admin commands are allowed

gpg/card> help
quit           quit this menu
admin          show admin commands
help           show this help
list           list all available data
name           change card holder's name
url            change URL to retrieve key
fetch          fetch the key specified in the card URL
login          change the login name
lang           change the language preferences
salutation     change card holder's salutation
cafpr          change a CA fingerprint
forcesig       toggle the signature force PIN flag
generate       generate new keys
passwd         menu to change or unblock the PIN
verify         verify the PIN and list all data
unblock        unblock the PIN using a Reset Code
factory-reset  destroy all keys and data
kdf-setup      setup KDF for PIN authentication
key-attr       change the key attribute

gpg/card> passwd 
gpg: OpenPGP card no. D2760001240103030005000097540000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? Q
```

Make sure to use strong non default pins, otherwise someone stealing your token will be able to use the keys it contains. Make sure to remember these pins too! If you enter the wrong user pin 3 times (which blocks the card, only the admin pin can be used to recover the card), and then the wrong admin pin 3 times, the card will "self destroy".

Another useful step is to configure the key to use "as strong as posssible" keys; for example, my Nitrokey is by default using a RSA-2048, but it actually supports RSA-4096, so better to enable this:

```bash
gpg/card> key-attr 
Changing card key attribute for: Signature key
Please select what kind of key you want:
   (1) RSA
   (2) ECC
Your selection? 1
What keysize do you want? (4096) ?
No help available for 'cardedit.genkeys.size'
What keysize do you want? (4096) 0
RSA keysizes must be in the range 1024-4096
What keysize do you want? (4096) 4096
Changing card key attribute for: Encryption key
Please select what kind of key you want:
   (1) RSA
   (2) ECC
Your selection? 1
What keysize do you want? (4096) 4096
Changing card key attribute for: Authentication key
Please select what kind of key you want:
   (1) RSA
   (2) ECC
Your selection? 1
What keysize do you want? (4096) 4096
```

At this stage, you should have a token with strong pins, strong key standard, ready to receive your private key. Note that you can, if you want, further personalize the key (you can google to find more infornation about the conventions around what each field stores; you may or may not want to personalize your key though, as personalizing your key "leaks" a bit of information about you in case your key is randomly stolen or lost):

```bash
gpg/card> name
Cardholder's surname: Surname
Cardholder's given name: Name

gpg/card> url
URL to retrieve public key: some_url_for_public_key

gpg/card> login
Login data (account name): login_name

gpg/card> salutation
Salutation (M = Mr., F = Ms., or space): M

gpg/card> lang
Language preferences: en

gpg/card> list

Reader ...........: 20A0:4108:000000000000000000009754:0
Application ID ...: D2760001240103030005000097540000
Application type .: OpenPGP
Version ..........: 3.3
Manufacturer .....: ZeitControl
Serial number ....: 00009754
Name of cardholder: Name Surname
Language prefs ...: en
Salutation .......: Mr.
URL of public key : some_url_for_public_key
Login data .......: login_name
Signature PIN ....: forced
Key attributes ...: rsa4096 rsa4096 rsa4096
Max. PIN lengths .: 64 64 64
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```

## Generating the key, backing it up on the Raspberry Pi, and loading it on one or more tokens

The private keys are stored within tokens on a "check in, never check out" basis. This means that, once the key creation process is over, there is no way to export / recover the private keys stored on the token. This is a security feature: if the token implementation (in particular, the safe element it contains) is done correctly, there is no way to steal your keys. However, that also means that special care must be taken to make sure that you can actually back up the keys present on the token as needed. Actually, this goes even further: by default, when e

Also note that we say "keys" not "key". explanation out of scope, but should have different keys for the different purposes, see for example discussion at XXRef. 

There are two ways to have the secret keys on the token backed up: i) create the keys on the token exporting them at creation, ii) create on the computer, back up

if generate on the nitrokey: can never back up. break, loose the nitrokey

therefore: generate on RPi. each time load private key on the token, will be erased from the machine and replaced by a stub. export the private key first. then upload on nitrokey. then restore the private key from the exported. then upload on backup nitrokey. exported file still available if need it.

at this stage, ...

put the RPi in a safe. in theory, as long as really strong passphrase, ok if accessed, but don t take the chance and protect it physically in addition

todo: user and admin pin code...

## Using a token: setup on a new computer and examples

load public key
make sure all packages are installed and udev rules are in place
should be possible to use the private key on the token
use pin code
use key

example: encrypt / decrypt file
example: use the pass password manager

## A few thoughts

- keys with or without keyboard? without keyboard => no bruteforce attack, no distribution issue, but still need to trust the computer from which working to not compromise the pin code; I like nitrokeys very well (well documented, code looks robust, Germany), but do not come with a keypad; onlykey: feel less confident about it (firmware was a mess, not sure how would resist physical tempering attacks, made in US) but has a keypad...

- quantum safe algorithms?

- system clipboard; what do you trust or not, what is the attack superficy?

- made my life much easier: 

- signature vs encryption key: why should be different; UK law; possible attack

to check against:
https://wiki.debian.org/GnuPG/AirgappedMasterKey
https://keyring.debian.org/creating-key.html
https://wiki.debian.org/GnuPG/SmartcardSubkeys

- why different kinds of keys

- generate keys on the token vs on the computer

- RPis are cheap enough that can be used as "single use computers"
