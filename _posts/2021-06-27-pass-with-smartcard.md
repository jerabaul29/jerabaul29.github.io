---
layout: post
title:  "Using pass password manager with smartcard"
date:   2021-06-27 15:00:00 +0200
categories: jekyll update
---

I am using password-store, the standard unix password manager, for keeping control of my passwords, usernames, SSH passphrases, etc. The content of my password manager is synchronized through a github private repository, which makes it very easy to keep things synchronized across machines. In order to have even higher security, I use a Nitrokey Pro 2 to hold my private key in a safe smartcard and perform all encryptions and decryptions. This is hopefully quite sure, and very convenient to use. I have this set up nicely now (that may be the point of another post some day), but still, there is a tiny bit of setup to do each time I use a new machine, which I describe in this post.

## Seting up pass on a new machine

To setup pass and the pass repo on a new machine:

```bash
~$ sudo apt install pass
~$ git clone YOUR_GIT_REPO_CLONE_SSH_GIT_URL .password-store
~$ gpg --import YOUR_PATH_TO_PUBLIC_KEY
~$ gpg --edit-key YOUR_KEY_ID
	# and inside the gpg CLI tool:
	trust  # edit the trust of your key
	5  # or set your trust level accordingly to what is reasonable
	quit  # quit saving edits
```

## Setting up the GPG smartcard


There is a bit of installation to be able to use GPG smartcards on a machine. First, if this is the first time a GPG smartcard is used at all:

```bash
$ sudo apt install scdaemon
```

You also may need to install some udev rules so that the smartcard is recognized by your system. For example for my Nitrokey, I need to follow the instructions at [https://www.nitrokey.com/documentation/frequently-asked-questions-faq#openpgp-card-not-available](https://www.nitrokey.com/documentation/frequently-asked-questions-faq#openpgp-card-not-available).

Once this is done, it is possible to register that the private key is available from the smartcard by doing:

```bash
$ gpg --card-status
```

## Using another smartcard

I actually have several physical smartcards with copies of the same private key (for safety / in case I loose or damage one). While I have read that future GPG versions will support using several smartcards, the version available currently on Ubuntu 20.04 only handles a single smartcard per key. That means that, in order to use another card, I need at present to 1) "unregister" the private key and associated card, 2) register the new card. For this:

```bash
$ gpg --list-secret-keys
$ # use the output to find which private key corresponds to the smartcard, and note its ID
$ gpg --delete-secret-keys KEY_ID
$ gpg --card-status
```

Now the old smartcard is unregistered, and the new one is registered and ready to use. Of course, to use the other card back again, the same process must be done again.
