---
layout: post
title: "Setting up a secure reverse SSH tunnel"
date: 2022-03-12 20:00:00 +0200
categories: jekyll update
---

Reverse SSH tunnels are very convenient for accessing servers that are behind firewalls / NATs and similar. For example, I have a RPi server for backup purposes at the home of some family member. I do not want to set up port forwarding and change the parameters of their internet routers the way I do it home, nor to set up some dynamic DNS pointing to their router, but I do want to be able to SSH to my backup server from outside of their local network.

For this, the solution is to set up a reverse SSH tunnel that allows me to connect from my main home server (called "main" in the following, for which I have set up a port forwarding rule on my router and dynamic DNS registered at [https://desec.io/](https://desec.io/), so I can reach it from everywhere on the internet), to the remote backup server (called "backup" in the following). This reverse SSH tunnel is set up by the "backup" server, by ssh-ing into the "main" server with the correct set of options. Here, I describe how to do so as safely as possible.

## Possible security risks associated with setting up a reverse SSH tunnel automatically using a non-passphrase-protected SSH key

There are security risks associated with setting up a reverse SSH tunnel, from a "backup" server at a location that is not fully under one's control, into a critical "main" server, where "backup" needs to be able to boot unattended to recover from issues / power losses etc:

- if the location of "backup" is compromised, then "backup" is compromised, and given that it needs to be able to reboot on its own to be fully unattended (ie, no full disk encryption or similar is applicable on the remote server, and there is no passphrase on the SSH keys used by "backup"), the account used for setting up the reverse tunnel on "main" may be compromised and the keys leaked,

- if "backup"'s keys are compromised, one has to make sure that i) the keys are not enough to perform anything dangerous on "main", ii) the keys are not enough to even perform local instead of reverse ssh tunnel and port forwarding, neither dynamic port forwarding; else, "main" could be used by the attacker either as a proxy server (not nice, if this is used for bad purposes), or to ping the local network behind the internet router / NAT / firewall on "main" side, which may compromise the home network, be used to hack into badly secured IoT devices, etc.

## Setting up a secure account and configuring sshd\_config to only allow reverse tunneling

To make things safe, the solution is to provide "backup" with a SSH key (without passphrase) that is enough to set up a reverse SSH tunnel, but not to do anything else. This way, even if "backup" is compromised and its key leaked, the only thing an attacker can with the SSH key do is to established a reverse SSH tunnel from another server than backup - i.e., possibly giving me access to their local network and machine. Not much extra harm for me (relatively to the fact that "backup" has been compromised).

To do so, the solution is to create a dedicated user on "main" (let us call this user "limited"), that:

- is only a "normal" user, i.e. not a member of the sudo group,
- only accepts SSH keys as a way to log in, no password login accepted (since password login is generally weaker than SSH keys),
- is dedicated solely to the purpose of establishing the SSH reverse tunnel (so revocating "limited" if "backup" is compromised would be a low threshold operation),
- cannot do anything on "main"; for that, "limited" is given a dummy shell that cannot be used to actually do anything,
- cannot do forwarding of ports local to the backup into the main server.

To create the user "limited" (you will be asked to provide a password; just provide a "dummy" password for now):

```bash
$ sudo adduser limited
```

To assign "limited" with a dummy shell that does not allow to do anything, the administrator can edit ```/etc/passwd``` so that it looks like (note that the UID and GID, here both 1001, may be different in your case):

```bash
$ cat /etc/passwd
# many lines with other users...
# set the shell (last field) to /usr/sbin/nologin
limited:x:1001:1001:,,,:/home/limited:/usr/sbin/nologin
```

To disable password login, the administrator can edit ```/etc/shadow``` so that it looks like the following, when the important point is that the field after limited is let blank, i.e. ```limited::``` is blank between the double ```::```:

```bash
$ sudo cat /etc/shadow
# many lines with other users...
limited::19057:0:99999:7:::
```

At this stage, it is not possible to log into the account of the user "limited". This can be easily verified by attempting to log into the user's account, which returns the error message provided by ```/usr/sbin/nologin``` and does nothing else:

```bash
$ sudo su limited
This account is currently not available.
```

To make it possible to ssh into the user, it is necessary to create a ssh key (I personally create it on "main" and put it into the ```.ssh``` folder of the "limited" user, to keep track of it, though it could as well be generated from the "backup" machine and the public key only transferred to "main"), and add it to the list of authorized hosts by adding it to the ```authorized_keys``` file of the user "limited":

```bash
# user "limited" is not active, we need to be sudo to create new folder 
$ sudo mkdir /home/limited/.ssh

# create the ssh key, without passphrase
# need sudo to be able to put the key into the folder
# /home/limited/.ssh/id_rsa_backup_reverse_ssh
$ sudo ssh-keygen -t rsa -b 4096
# Generating public/private rsa key pair.
# Enter file in which to save the key (/root/.ssh/id_rsa): /home/limited/.ssh/id_rsa_backup_reverse_ssh
# Enter passphrase (empty for no passphrase):
# Enter same passphrase again:
# Your identification has been saved in /home/limited/.ssh/id_rsa_backup_reverse_ssh.
# Your public key has been saved in /home/limited/.ssh/id_rsa_backup_reverse_ssh.pub.

# set up authorized_keys and copy the newly created public key
$ sudo touch /home/limited/.ssh/authorized_keys
$ cat /home/limited/.ssh/id_rsa_backup_reverse_ssh.pub | sudo tee -a /home/limited/.ssh/authorized_keys
```

At this point, it is possible to use the newly created ssh key to "access" the "limited" account. For example, I can do the following on another machine (i.e., on my "working" machine, but it would be the same from the "backup" server, it is just more convenient for me to do it directly from "working" for illustrative purposes):

```bash
# create a new file to host the private key for the user "limited"
~/.ssh$ touch id_rsa_backup_reverse_ssh

# copy the private key (note: I do this simply in vim by copy pasting the output of
# $ cat /home/limited/.ssh/id_rsa_backup_reverse_ssh
# obtained on "main" ); assume that "working" ~/id_rsa_backup_reverse_ssh now contains
# the copy of the private key

# edit the rights of the private key so that ssh-add accepts to load it
~/.ssh$ chmod g-rw id_rsa_backup_reverse_ssh 
~/.ssh$ chmod o-r id_rsa_backup_reverse_ssh 

# add the key to my keyring
~/.ssh$ ssh-add id_rsa_backup_reverse_ssh 

# attempt to log onto "main"; this will succeed but return immediately,
# with the expected nologin message:
# in my case, "main" is accessible from "working" as 192.168.0.3 :
~/.ssh$ ssh limited@192.168.0.3 -o PreferredAuthentications=publickey
This account is currently not available.
Connection to 192.168.0.3 closed.
```

So the key is already partially "safe" to use on the less-trusted "backup" machine in the sense that, even if compromised, the attacker cannot get a shell and do anything on "main". However, it is still possible to set up any kind of port forwarding, including the reverse forwarding which we are interested in and is not particularly dangerous, but also more dangerous local and dynamic forwarding. To avoid this, the solution is to edit the configuration of the ssh agent on "main", to only allow the user "limited" to set up reverse tunnels, and nothing else. This can be done by adding the following to ```/etc/ssh/sshd_config``` on "main" (note: editing this file will require root privileges, but it is readable for all users), i.e., enable only starting remote forwarding tunnels for the user limited:

```bash
$ cat /etc/ssh/sshd_config
# many skipped lines...

Match User limited
        AllowTcpForwarding remote
```

You will need to restart your ssh daemon on "main" for the changes to take effect:

```bash
sudo systemctl restart sshd.service
```

You can test that this is working as it should. For example, reverse SSH tunneling works just fine, i.e., you can access a port on "working" through a local port on "main":

```bash
# in a terminal, on "working"
# set up a dummy web server on some arbitrary port
~$ WORK_DIR=$(mktemp -d)
~$ cd $WORK_DIR
/tmp/tmp.tp6VVGMHib$ echo "hello, from working" >> index.txt
/tmp/tmp.tp6VVGMHib$ python3 -m http.server 8093 --bind 127.0.0.1

# in another terminal, on "working", to check that the server works:
~$ curl 0.0.0.0:8093/index.txt
hello, from working

# in another terminal, on "working", set up a reverse ssh tunnel
# forwarding connections from a dummy port on "main" to the server on "working"
$ ssh -NR 8093:localhost:8093 limited@192.168.0.3 -o PreferredAuthentications=publickey

# in a terminal, on "main", you can now access the server on "working"
# through the local port 8093:
$ curl 0.0.0.0:8093/index.txt
hello, from working
```

You can also check that both local port forwarding, and dynamic port forwarding, are disabled:

```bash
# setting up a local or dynamic port forwarding will be blocked;
# from "working", given that a web server is available on port 8093
# of "main" similar to the previous example:
$ ssh -NL 8093:localhost:8093 limited@192.168.0.3 -o PreferredAuthentications=publickey &
[1] 558291
$ curl 0.0.0.0:8093
channel 2: open failed: administratively prohibited: open failed
curl: (56) Recv failure: Connection reset by peer
# i.e., the connection was refused by the ssh daemon on "main":
# no local port forwarding
$ kill 558291  # stop the background ssh command

# similarly the dynamic port forwarding will be blocked; from "working":
$ ssh -ND 8095 limited@192.168.0.3 -o PreferredAuthentications=publickey &
[1] 592333
$ curl -x socks5h://0.0.0.0:8095 https://example.com
channel 2: open failed: administratively prohibited: open failed
curl: (7) Failed to receive SOCKS5 connect request ack.
# i.e., dynamic port forwarding is blocked too.
$ kill 592333  # stop the background ssh command
```

Let us summarize: the user "limited" on "main" is now:

- only accepting SSH key authorization, no password
- provided with a nologin shell that does not allow to run any command
- limited to performing reverse port forwarding.

So, setting up the key on "backup" lets us do exactly what we wanted initially, i.e., provide only the ability to perform reverse forwarding from "main" to "backup", and nothing else.

## Automating reverse tunnel setup using systemd, and making the reverse ssh tunnel robust

The last remaining step for fully automating the setup of the reverse tunnel, and making sure that "backup" automatically makes itself available through reverse tunneling to "main" as soon as it is up, is to write a custom systemd service calling scripts that set up a robust ssh tunnel.

When setting up a systemd service to start and keep up the ssh reverse tunnel, it is important to i) wait for the network stack to be available, ii) restart the script if something unexpected happens and it crashes. For this, the following systemd service can be set up (in my case, this is happening on a raspberry pi, with main user pi):

```bash
$ cat /etc/systemd/system/tunnel.service 
[Unit]
Description=Set up and maintain reverse tunnel
After=network.target

[Service]
User=pi
ExecStart=/bin/bash /home/pi/Scripts/start_tunnel.sh
Type=simple
Restart=always
RestartSec=15
KillMode=mixed

[Install]
WantedBy=multi-user.target
```

The corresponding script for starting and keeping the tunnel up should also be as robust as possible. In particular, the script should check for availability of the network by pinging an external IP, and it should make sure that the SSH tunnel is active, and that if it freezes, it gets restarted:

```bash
$ cat /home/pi/Scripts/start_tunnel.sh 
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

# parameters for the script

# we connect from the current host in a way similar to: ssh USER_ON_REMOTE@REMOTE_NAME_OR_IP -p SSH_PORT_ON_REMOTE
REMOTE_NAME_OR_IP="something.dedyn.io"
USER_ON_REMOTE="limited"
# the port to use to ssh to "main" from the internet;
# I set a forwarding rule from a randomly chosen
# public port to the "main" port 22, to keep cleaner logs...
SSH_PORT_ON_REMOTE="some_port_1"

# the remote port REMOTE_NAME_OR_IP:FORWARDED_PORT_ON_REMOTE will be forwarded to the local port localhost:FORWARDED_PORT_ON_LOCAL
FORWARDED_PORT_ON_REMOTE="some_port_2"
FORWARDED_PORT_ON_LOCAL="some_port_3"

echo "sleep..."
sleep 30

echo "wait for network"
while ! ping -c 1 -W 1 8.8.8.8; do
    echo "Waiting for 8.8.8.8 - network interface might be down or network not available..."
    sleep 300
done

echo "start ssh agent and add identity"
eval `ssh-agent`
ssh-add ~/.ssh/id_rsa_some_key

while true; do
    # actually, do not use autossh: it requires opening extra ports
    # for the double checks, and keeping track of these if several
    # tunnels, etc, is a pain...
    # autossh -v -o ServerAliveInterval=120 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes -NR "${FORWARDED_PORT_ON_REMOTE}":localhost:"${FORWARDED_PORT_ON_LOCAL}" "${USER_ON_REMOTE}"@"${REMOTE_NAME_OR_IP}" -p "${SSH_PORT_ON_REMOTE}"
    # ssh with alive options is enough to keep things up,
    # together with a bit of sshd setup
    echo "start autossh tunneling"
    ssh -o ServerAliveInterval=120 -o ServerAliveCountMax=3 -NR "${FORWARDED_PORT_ON_REMOTE}":localhost:"${FORWARDED_PORT_ON_LOCAL}" "${USER_ON_REMOTE}"@"${REMOTE_NAME_OR_IP}" -p "${SSH_PORT_ON_REMOTE}"
    sleep 300
done

echo "something bad happened; ssh returned; this should not happen; try again!"

```

The service is now ready to be enabled and started:

```bash
sudo systemctl daemon-reload
sudo systemctl enable tunnel
sudo systemctl start tunnel
```

## Connecting over SSH from "main" to "backup"

Once this is set up, you can easily ssh to "backup" from "main": set up a SSH key that allows "main" to log into "backup", add it to the ssh agent, and it should be possible to ssh to "backup" (in my case, a raspberry pi with main user pi), from "main", by using the reverse forwarded port, even if "backup" is not reachable from the internet, or is behind a firewall or NAT:

```bash
# to ssh from "main" to "backup":
$ eval "$(ssh-agent -s)"
$ ssh-add .ssh/id_rsa_main_to_backup
$ ssh pi@localhost -p 8080 -o PreferredAuthentications=publickey
pi@raspberrypi:~ $
```
