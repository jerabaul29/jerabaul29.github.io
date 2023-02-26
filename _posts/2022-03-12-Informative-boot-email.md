---
layout: post
title: "Linux trivia: sending an informative status email at boot and periodically after"
date: 2022-02-12 19:00:00 +0200
categories: jekyll update
---

This is a small post about how to send an informative email at boot, and periodically after, using systemd. I have a number of Raspberry Pis spread around, and I like to get some notification when one of them (unexpectedly) reboots, as well as some information about their status, periodically.

## Scripting the email body creation and sending

To do so, I set up a script that can create an email with all the content needed, and send it; in my case, I use something like the following (the content of the EMAIL\_FILENAME file should be adapted to your needs):

```bash
$ cat /home/pi/Scripts/prepare_and_send_email.sh
# exit if a command fails; to circumvent, can add specifically on commands that can fail safely: " || true "
set -o errexit
# make sure to show the error code of the first failing command
set -o pipefail
# do not overwrite files too easily
set -o noclobber
# exit if try to use undefined variable
set -o nounset
# on globbing that has no match, return nothing, rather than return the dubious default ie the pattern itself
# see " help shopt "; use the -u flag to unset (while -s is set)
shopt -s nullglob

# start by waiting a bit - make sure network gets good time to get up etc
echo "sleep..."
sleep 30

# parameters for the script
# path to the email file
EMAIL_FILENAME="/home/pi/Scripts/email_content.txt"

echo "wait for network"

while ! ping -c 1 -W 1 8.8.8.8; do
    echo "Waiting for 8.8.8.8 - network interface might be down or network not available..."
    sleep 300
done

echo "prepare the email content"

# create an empty file
>| "${EMAIL_FILENAME}"

echo "From: some_email@gmail.com" >> "${EMAIL_FILENAME}"
echo "To: some_email@gmail.com" >> "${EMAIL_FILENAME}"
echo "Subject: SOME_RPI backup RPi4 status" >> "${EMAIL_FILENAME}"
echo "" >> "${EMAIL_FILENAME}"

# custom information
echo "IP: $(curl ifconfig.me)" >> "${EMAIL_FILENAME}"
# information about the system
echo "current date: $(date)" >>  "${EMAIL_FILENAME}"
echo "uptime: $(uptime)" >>  "${EMAIL_FILENAME}"
echo "df -h output:" >> "${EMAIL_FILENAME}"
df -h >> "${EMAIL_FILENAME}"
echo "" >> "${EMAIL_FILENAME}"

# add any information to the body of the email
echo "SOME_INFORMATION" >> "${EMAIL_FILENAME}"
echo "" >> "${EMAIL_FILENAME}"

echo "send the email"

# sent the email
sendmail some_email@gmail.com < ${EMAIL_FILENAME}

echo "done"
```

To make sure this script is correctly set up, that all the packages are installed as they should, etc, I then run it from the command line "by hand", to make sure that the email is well sent with all the correct settings: a simple ```bash prepare_and_send_email.sh``` command does the job.

## Automatically sending the email at boot

To automatically send the email at boot and periodically after, there are several options:

- 1 (discouraged) Using cron: simply add an entry to the crontab by editing it for the user (```crontab -e```), and adding the corresponding entry: ```@reboot sleep 30; cd /home/pi/Scripts/; bash prepare_and_send_email.sh``` to send at boot. You can add some additional entries to send periodically after that, using the crontab calendar schedule. Note that this may not work as expected as the environment of the cron is not the same as the environment of the user. Execution of the script is also not logged - you could log it by redirecting the output of the command to a text file, but then you will also need to ```logrotate``` the log file etc.

- 2 (recommended) Using systemd: systemd is a better way to automate jobs running in modern linux distros. It is logged, has an interface to get information about the jobs that have been run and their status, etc. To use systemd, set up a service file and associated timer:

```bash
$ cat /etc/systemd/system/informative_email.service 

[Unit]
Description=Send informative email
Wants=informative_email.timer

[Service]
User=pi
ExecStart=/bin/bash /home/pi/Scripts/prepare_and_send_email.sh
Type=simple

[Install]
WantedBy=multi-user.target
```

```bash
$ cat /etc/systemd/system/informative_email.timer
[Unit]
Description=Send informative email at boot and periodically after
Requires=informative_email.service

[Timer]
Unit=informative_email.service
OnBootSec=1m
OnCalendar=*-*-* 03:00:00
AccuracySec=5m

[Install]
WantedBy=timers.target
```

Once the service and timer files are ready, activating the timer and testing it is done by:

```bash
sudo systemctl daemon-reload
sudo systemctl enable informative_email.timer  # enable the timer, not the service
# sudo systemctl disable informative_email.timer  # disable the timer
sudo systemctl start informative_email.timer  # start the timer immediately
# sudo systemctl stop informative_email.timer  # stop the timer immediately
```

The status and logs of the service can be obtained by:

```bash
systemctl status informative_email.timer
```
