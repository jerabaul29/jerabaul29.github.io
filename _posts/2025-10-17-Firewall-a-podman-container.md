---
layout: post
title: "Firewall a podman container"
date: 2025-10-17 12:00:00 +0200
categories: jekyll update
---

*disclaimer: I am not a podman nor a security expert; in addition, these explanations are valid as of 2025-10 and focus on podman rootless containers: things may change with time, and be different for non standard rootless ways to run podman containers!*

This blog post discusses podman container firewalling: why it is important, and how to do it in practice, with examples and explanations.

## A few words about firewalling and podman containers

Containers, such as podman containers, are a convenient way to run apps in an isolated, compartimentalized, reproducible and scalable way, and they are becoming very popular. While running an application in a container gives an inherent security boost (because, among others, of compartimentalization), this does not mean that common security best practices should not apply to containers as well. There are several key best practice aspects to follow, one of them being firewalling and network control. Ideally, containers running should be firewalled individually depending on the exact needs they have - something that in Norway is sometimes called the "VG test", from the name of a newspaper: an application has no reason to be able to read the VG newspaper, and if it does, it likely can access many other IPs too, such as a malware control server.

Out of the box, it is easy to set up a container with no network access at all: for example, the following podman Containerfile:

```bash
# Use the latest Ubuntu image
FROM ubuntu:latest

# Install netcat (nc)
# ping does not work in rootless container as it uses suid or NET_ADMIN which are not allowed
RUN apt-get update && apt-get install -y netcat-openbsd && apt-get clean

# Run the command `nc -w 3 -z 8.8.8.8 53` to check connectivity to Google's DNS server
CMD nc -w 3 -z 8.8.8.8 53 || echo "Error: Network unavailable or unable to reach 8.8.8.8 on port 53"
```

can be built with `podman build -t ping_google_dns .`, and then run:

- with network access:

```bash
/tmp/tests_podman> podman run --rm -it localhost/ping_google_dns:latest
Connection to 8.8.8.8 53 port [tcp/*] succeeded!```

- without network access:

```bashbash
/tmp/tests_podman> podman run --rm -it --network=none localhost/ping_google_dns:latest
Error: Network unavailable or unable to reach 8.8.8.8 on port 53
```

However, there are many cases "in between", where a container needs network access to perform a task, but only a few specific IPs should be possible to contact. Ideally, such a container should be run with the minimum set of network connectivity that is enough to perform its task. This is a common best practice, that limits attack superficy against the container, and limits the possibility of data exfiltration. For example, if a container has been compromised by a supply chain attack, effective firewalling may / should make it impossible for the container to contact non-trusted IPs and, hence, to contact a control server or to exfiltrate secrets. Note that these firewal rules at the container level can / should apply in addition to firewalling applied to the network as a whole that the hosts running the container work from.

Setting up such fine-grained firewalling on a container to container basis is a bit more complex that simply turning network on and off. In the following, we will discuss how this can be done, anno 2025.

## Linux namespaces and podman containers: a handwavy explanation

Before actually implementing firewalling of an individual container, we will quickly discuss a handwavy, high-level explanation of how podman containers work and are made possible. Note that this is a coarse and handwavy explanation (a "cognitive model"), and that I am not an expert about podman implementation details - don't quote me on this. Also, this applies to standard-run rootless containers.

What makes containers such as podman containers possible, is a set of Linux kernel features such as namespaces, chroot, and more. These features allow to isolate and encapsulate how code is run, and to run code with root power inside a given namespace. This allows code to perform root-level actions that are confined into a namespace, without the ability to perform root actions outside of it. At a high level, when starting a podman rootless container, podman sets up a dedicated namespace in which the container will be run. Inside this namespace, it is possible to have root-level rights, such as changing firewall properties that apply to the network of this namespace (even though, of course, firewall modification of the encapsulating host is not possible). When using a rootless container under "standard" conditions (see below), these capabilities are dropped when the container app starts. This means, in practice, that the container app gets firewalled without the possibility to break out.

Let us illustrate this with a small example, with a new containerfile (and yes, I know, I should use a more recent tool than `ifconfig` anno 2025+; but hey, habits are hard dying):

```bash
FROM ubuntu:latest

RUN apt-get update && apt-get install -y iptables && apt-get clean
```

And building and running it:

```bash
/tmp/barebone_podman> podman build -t barebone_iptables:latest .
/tmp/barebone_podman> podman run --rm -it barebone_iptables:latest /bin/bash
root@4b91b7598738:/# iptables -L
iptables v1.8.10 (nf_tables): Could not fetch rule set generation id: Permission denied (you must be root)
```

You can see that, although we are logged into the podman container as root, the `NET_ADMIN` capabilities have been dropped before we got the terminal, so that we cannot perform iptables operations. Compare with:

```bash
/tmp/barebone_podman> podman run --rm -it --cap-add NET_ADMIN barebone_iptables:latest /bin/bash
root@918d3d60f321:/# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
```

Here, we add the `NET_ADMIN `capability also at the container run time, so that iptables operation are allowed for the root inside the container. Naturally, the iptables commands only apply to the network configuration inside the container namespace, and these commands cannot modify anything on the host network. But this is a good reminder, to be careful not to provide extra capabilities to podman run if the app running inside it should be e.g. firewalled.

## Firewalling podman rootless containers in an effective and safe way

The rights a podman container has inside its own namespace, as well as the capabilities dropping mechanism, can be used to effectively firewall apps. A way to do so, that we will use in the following, is to use OCI hooks to run code during the container creation, after the container namespace is created, but before the NET_ADMIN capability is dropped. For more details and alternative ways to do so, see [this discussion on the official podman github repository](https://github.com/containers/podman/discussions/27099) (this is anno 2025 and the state of podman, recommended ways to go forward, etc, may change with time). I also have a self contained example [on this github repository](https://github.com/jerabaul29/2025_podman_iptable_rules).

At the moment (this may change with time if simpler ways to run OCI hooks get implemented into podman), this is a bit technical and requires a few steps:

- OCI hooks run bash scripts and can be triggered at different points in the container lifetime - this means that one needs to write the corresponding bash scripts
- the hooks themselves are defined in .json files; they point to the bash scripts to run, and contain information about when they should be triggered
- the podman run command should point to where the hooks live (if not using a default location), and have attached e.g. annotations that are used to trigger different hooks.

We can take a simple example inspired from [the github repository proof of concept](https://github.com/jerabaul29/2025_podman_iptable_rules), the key elements being:

- A hook to run the firewalling bash script:

```bash
/tmp/example_firewall> cat hook/iptables_hook.json 
{
    "version": "1.0.0",
    "hook": {
        "path": "/tmp/example_firewall/firewall.sh"
    },
    "when": {
        "annotations": {
            "^apply_iptable_hook$": ".+"
        }
    },
    "stages": ["createContainer"]
}
```

You can observe a few things here: the path to the script to run needs to be a fully qualified path; the hook triggering mechanism is to detect a regexp matching to the `apply_iptable_hook` string to one of the annotations attached to a podman run command; the stage at which the bash script will be run is `createContainer`, which is a point in the container running chain when the container namespace is set up, but the container app is not started yet and the NET_ADMIN capabilities are not dropped yet.

- The firewalling script itself:

```bash
/tmp/example_firewall> ls -lsrth firewall.sh 
4,0K -rwxrwxr-x 1 myuser myuser 964 okt.  21 13:22 firewall.sh
/tmp/example_firewall> cat firewall.sh 
#!/bin/bash

echo "Start firewall.sh hook" > /tmp/example_firewall/log_firewall.log 2>&1

echo -n "whoami: " >> /tmp/example_firewall/log_firewall.log 2>&1
whoami >> /tmp/example_firewall/log_firewall.log 2>&1

# Block traffic to 8.8.8.8 but not other stuff
iptables -P INPUT DROP >> /tmp/example_firewall/log_firewall.log 2>&1
iptables -P OUTPUT DROP >> /tmp/example_firewall/log_firewall.log 2>&1
iptables -P FORWARD DROP >> /tmp/example_firewall/log_firewall.log 2>&1
# allow traffic to and from 1.1.1.1 as an example
iptables -A INPUT -s 1.1.1.1 -j ACCEPT
iptables -A OUTPUT -d 1.1.1.1 -j ACCEPT

# Log the applied rules (optional, for debugging)
echo "Applied iptables rules:" >> /tmp/example_firewall/log_firewall.log 2>&1
echo -n "iptables -L -v: " >> /tmp/example_firewall/log_firewall.log 2>&1
iptables -L -v >> /tmp/example_firewall/log_firewall.log 2>&1

echo "Finished set_iptable_rules_denayll.sh hook" >> /tmp/example_firewall/log_firewall.log 2>&1
```

Here too a few things can be observed: the script should have executable rights; for debugging purposes, we log the different commands.

- Finally we need a basic containerfile:

```bash
/tmp/example_firewall> cat Containerfile 
FROM ubuntu:latest

# Update and install wget
RUN apt-get update && \
    apt-get install -y netcat-openbsd --no-install-recommends && \
    apt-get install -y libcap2-bin --no-install-recommends && \
    apt-get install -y iptables --no-install-recommends && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

ENV TERM xterm-256color
ENV COLORTERM truecolor
```

- We can now run this example as:

```bash
/tmp/example_firewall> podman build -t firewalling:latest .

/tmp/example_firewall> podman run --rm -it --annotation apply_iptable_hook=true --hooks-dir /tmp/example_firewall/hook/ firewalling:latest /bin/bash
root@94a09787a893:/# capsh --print
Current: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap=ep
Bounding set =cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap
Ambient set =
Current IAB: !cap_dac_read_search,!cap_linux_immutable,!cap_net_broadcast,!cap_net_admin,!cap_net_raw,!cap_ipc_lock,!cap_ipc_owner,!cap_sys_module,!cap_sys_rawio,!cap_sys_ptrace,!cap_sys_pacct,!cap_sys_admin,!cap_sys_boot,!cap_sys_nice,!cap_sys_resource,!cap_sys_time,!cap_sys_tty_config,!cap_mknod,!cap_lease,!cap_audit_write,!cap_audit_control,!cap_mac_override,!cap_mac_admin,!cap_syslog,!cap_wake_alarm,!cap_block_suspend,!cap_audit_read,!cap_perfmon,!cap_bpf,!cap_checkpoint_restore
Securebits: 00/0x0/1'b0 (no-new-privs=0)
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
 secure-no-ambient-raise: no (unlocked)
uid=0(root) euid=0(root)
gid=0(root)
groups=
Guessed mode: HYBRID (4)
root@94a09787a893:/# nc -w 3 -z 1.1.1.1 53 || true
Connection to 1.1.1.1 53 port [tcp/domain] succeeded!
root@94a09787a893:/# nc -w 3 -z 8.8.8.8 53   
root@94a09787a893:/# echo $?
1
root@94a09787a893:/# nc -w 3 -z vg.no 80 || true
nc: getaddrinfo for host "vg.no" port 80: Temporary failure in name resolution

/tmp/example_firewall> cat log_firewall.log 
Start firewall.sh hook
whoami: root
Applied iptables rules:
iptables -L -v: Chain INPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  any    any     1.1.1.1              anywhere            

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  any    any     anywhere             1.1.1.1             
Finished set_iptable_rules_denayll.sh hook
```

This shows that the whole chain of events is working as expected, and that the runtime app and commands run inside the podman container are effectively firewalled:

- The log_firewall.log logs show that the hook was well called and iptable rules have well been set up; all traffic is denied, except to and from the IP 1.1.1.1 .
- Inside the container, the cap_net_admin is dropped (as visible in the capsh output by the !cap_net_admin)
- We can observe that the firewalling rules are well working inside the container: it is possible to talk to the 1.1.1.1 IP, but not to the 8.8.8.8 IP; the vg.no server cannot be reached - communication to it is firewalled, but this fails even earlier: the request to DNS servers to get the IP of the vg.no server cannot even be reached.

## To go further

This illustrates how applications run inside a podman container in rootless mode, without NET_ADMIN capabilities, can be effectively firewalled. This is only one of the way to go - it could be possible in different podman run modes to leverage other mechanisms to firewall applications, see [the discussion on the podman repository](https://github.com/containers/podman/discussions/27099). In addition, the APIs and ways to run hooks may change, or new ways to run them may be added in the future - follow the discussions on the podman repository and the podman documentation!
