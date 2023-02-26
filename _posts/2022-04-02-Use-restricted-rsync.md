---
layout: post
title: "Use restricted-rsync (rrsync) to allow safe backup of a server"
date: 2022-04-02 19:00:00 +0000
categories: jekyll update
---

Needing to take a backup of a server is a common situation, for which ```rsync``` is a convenient solution. I will not discuss the features of ```rsync``` in details here, but it basically allows to take incremental backup based on the signature of the files in the folder to backup. It works seamlessly over ```ssh```, which is very convenient for performing remote backup of a server. In the following, this is the solution we will consider, and we will refer to the server to backup (one specific folder, with path "target", will have to be backed up) as "main", while the server on which the data get backed up will be referred to as "remote".

The backup will take place in the following way: "remote" will start a ```rsync``` command to backup over ssh the "target" on "main", and "main" will be set up to accept the backup of "target" to take place. Our goal in this post is to describe how to do so in a way that is as safe as possible.

## General setup

For making the process as safe as possible for "main", it is possible to set up a dedicated user "rsyncbackupuser" on "main" that can be used to perform the backup, and nothing else (setting up a restricted user can be done similarly to what is described in the post [Setting up a secure reverse SSH tunnel](https://jerabaul29.github.io/jekyll/update/2022/03/12/Secure-reverse-ssh-tunnel.html), with a few specificities that we will describe below). In case some of the folders to back up on "main" are restricted to some other users and cannot be accessed by "rsyncbackupuser", it is possible to make these available to the backup user [by using bindfs](https://jerabaul29.github.io/jekyll/update/2022/03/19/Mount-a-folder-as-readonly-bindfs.html).

In order to be able to perform the backup, "remote" should hold a private key (without passphrase if the backup is to be run automatically) to log onto "main" as user "rsyncbackupuser". In order to make things safer for "main" in case "remote" gets compromised, "rsyncbackupuser" should only be able to provide backup of the specific "target" over ssh, and should not be able to do any other action or access any other folder. This is possible by using restricted rsync, or ```rrsync```. Basically, ```rrsync``` is a Perl script that only allows to perform a ```rsync``` backup of a specific folder by checking the commands and arguments it receives, and actually running the corresponding commands only if they match with the set of operations that are allowed. Adding ```rrsync``` with adapted options as the forced command for the "rsyncbackupuser" in the ```~/.ssh/authorized_keys``` file for "main", by adding at the start of the entry for the authorized ssh "rsyncbackupuser" key the ```command="/usr/bin/rrsync OPTS DIR",``` entry, will effectively limit the owner of the corresponding key to only perform the specific ```rsync``` backup against the DIR path that is pre-configured.

## Detailed steps to set up ```rrsync``` backup

- Create a user account on "main" for the "rsyncbackupuser" (provide a dummy password, we will erase it in the next step):

```bash
$ sudo adduser rsyncbackupuser
```

- Disable password login on "main" for "rsyncbackupuser": for this, there are at least 2 methods that I know of (that can be applied on top of each other, and I would definitely recommend that you apply the first at least to disable the dummy password set at the previous step):

    - set a non matchable password in ```/etc/shadow```; this way, it is impossible to log to rsyncbackupuser using any password, in any conditions; this is done by putting a blank second field (ie the ```::``` after the user name):

    ```bash
    $ sudo cat /etc/shadow
    [many lines ...]
    rsyncbackupuser::19092:0:99999:7:::
    ```

    - Edit the ssh daemon configuration ```/etc/ssh/sshd_config``` by adding the ```PasswordAuthentication no``` line, to disable password authentication over SSH; note that this will disable it for all users, if you want to disable it only for the specific user you should use a match, like:

    ```bash
    Match User rsyncbackupuser
        PasswordAuthentication no
    ```

- Set up ssh key pair: for this, you should first generate a private-public key pair using for example ```ssh-keygen -t ed25519 -a 100```. Set up a key without passphrase if you want to be able to perform the backup automatically. The public ssh key that was just created (ie with the .pub suffix) should then be added on "main" to the ```authorized_keys``` file of "rsyncbackupuser", creating the necessary file and folder:

```bash
$ sudo mkdir /home/rsyncbackupuser/.ssh
$ sudo touch /home/rsyncbackupuser/.ssh/authorized_keys
$ sudo echo "CONTENT_OF_PUB_FILE" | sudo tee -a /home/rsyncbackupuser/.ssh/authorized_keys
```

and the private ssh key you just generated should be added to "remote" using ```ssh-add```.

- Add the correct ```command="", ``` directive into the ```authorized_keys``` file of "rsyncbackupuser". For this, you should decide in advance which folder you want to be able to back up from "main" to "remote". In my case, I want to back up ```/home/rsyncbackupuser/some_backup_folder```, in read-only mode (i.e., ```rrsync``` should only allow to read, not write, to the filder). For this, the ```authorized_keys``` or "rsyncbackupuser" should look like:

```bash
$ sudo cat /home/rsyncbackupuser/.ssh/authorized_keys
command="/usr/bin/rrsync -ro /home/rsyncbackupuser/some_backup_folder", ssh-ed25519 AAA[content of the public key continues...]
```

- Check that this works as it should, from "remote":

    - Add the ssh private key:

    ```bash
    ssh-add PATH_TO_KEY
    ```

    - Check that you cannot log with user "rsyncbackupuser" on "main" from "remote" using the key:

    ```bash
    $ ssh rsyncbackupuser@IP
    /usr/bin/rrsync: Not invoked via sshd
    Use 'command="/usr/bin/rrsync [-ro|-wo] SUBDIR"'
        in front of lines in /home/rsyncbackupuser/.ssh/authorized_keys
    Connection to IP closed.
    ```

    - But that you can backup the content of the target read only folder:

    ```bash
    $ rsync rsyncbackupuser@IP:* .
    # now the local folder on "backup" should contain the same files as were present in the target directory on "main"
    # note that the paths expressed in the rsync command on "backup" are relative to the read only target folder
    # set in the command="", directive on "main"; i.e., the "*" pattern matches /home/rsyncbackupuser/some_backup_folder/*
    ```

    - And that you cannot write to the target directory:

    ```bash
    $ rsync * rsyncbackupuser@IP:*
    /usr/bin/rrsync sending to read-only server not allowed
    rsync: connection unexpectedly closed (0 bytes received so far) [sender]
    rsync error: error in rsync protocol data stream (code 12) at io.c(235) [sender=3.1.3]
    ```

## Making the backup run automatically

Taking a backup is good, taking regular backups is better! To make the backup run regularly, a good solution is to use a ```systemd``` service calling the ```rsync``` command from "remote" against "main", and a timer to execute it periodically on "remote". This can be done for example in the following way, assuming that the user running the backup on "remote" is ```pi``` (as I was running this on a RPi myself :) ):

```bash
$ cat /etc/systemd/system/rrsync_backup.service
[Unit]
Description=Perform a rrsync backup
Wants=rrsync_backup.timer

[Service]
User=pi
ExecStart=/bin/bash /home/pi/Scripts/backup_script.sh
Type=simple

[Install]
WantedBy=multi-user.target
```

```bash
$ cat /etc/systemd/system/rrsync_backup.timer
[Unit]
Description=Perform a rrsync backup at boot and after that once a day at night
Requires=rrsync_backup.service

[Timer]
Unit=rrsync_backup.service
OnBootSec=2h
OnCalendar=*-*-* 03:00:00
AccuracySec=30m

[Install]
WantedBy=timers.target
```

```bash
$ cat /home/pi/Scripts/backup_script.sh
#!/bin/bash
eval "$(ssh-agent -s)"
ssh-add /home/pi/.ssh/PRIVATE_SSH_KEY
rsync -azh --stats rsyncbackupuser@IP_OR_DOMAIN_OF_MAIN:* /home/pi/Backup/
```

To enable and immediately start the timer (but not the service), all what is needed then is:

```bash
sudo systemctl daemon-reload
sudo systemctl disable rrsync_backup.service
sudo systemctl enable rrsync_backup.timer
sudo systemctl start rrsync_backup.timer
```

## Bonus: a few words about the ```authorized_keys``` forced command

Just to illustrate how the ```command="",``` works (and how ```rsync``` works), it is possible to write a few lines of script. Setting up the following:

```bash
$ cat /home/rsyncbackupuser/Scripts/verbose_shell.sh
#! /bin/bash

if [[ -n $SSH_ORIGINAL_COMMAND ]] # command given, so run it
then
    echo "$SSH_ORIGINAL_COMMAND" >> /home/rsyncbackupuser/Scripts/verbose_shell.txt
    exec /bin/bash -c "$SSH_ORIGINAL_COMMAND"
else # no command, so interactive login shell
    echo "providing interactive login shell"
    exec bash -il
fi
```

and using it as the ```command``` argument:

```bash
$ cat /home/rsyncbackupuser/.ssh/authorized_keys

command="/bin/bash /home/rsyncbackupuser/Scripts/verbose_shell.sh", [content of the key ...]
```

This can be then used to show what the ssh client actually receives; we show a few examples of commands and associated ```verbose_shell.txt``` dump below:

- asking for a command to be run:

```bash
ssh rsyncbackupuser@IP "ls"
```

results simply in:

```bash
$ cat /home/rsyncbackupuser/Scripts/verbose_shell.txt 
ls
```

- and the content of the txt log following the ```rsync``` command is not that different:

```bash
$ cat /home/rsyncbackupuser/Scripts/verbose_shell.txt 
rsync --server --sender -logDtprze.iLsfxC . *
```

So basically, all the ```rrsync``` Perl script does, is check that the paths the remote user attempts to copy are under the disk location specified as a DIR, and if so, let the backup proceed.
