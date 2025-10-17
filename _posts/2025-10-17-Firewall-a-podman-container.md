---
layout: post
title: "Firewall a podman container"
date: 2025-10-17 12:00:00 +0200
categories: jekyll update
---

*disclaimer: I am not a podman nor a security expert; in addition, these explanations are valid as of 2025-10 and focus on podman rootless containers: things may change with time, and be different for non standard rootless ways to run podman containers!*

This blog post discusses podman container firewalling: why it is important, and how to do it in practice, with examples and explanations.

## A few words about firewalling and podman containers

Containers, such as podman containers, are a convenient way to run apps in an isolated, compartimentalized, reproducible and scalable way, and they are becoming very popular. While running an application in a container gives an inherent security boost (because, among others, of compartimentalization), this does not mean that common security best practices should not apply to containers as well. There are several key best practice aspects to follow, one of them being firewalling and network control. Ideally, containers running should be firewalled individually depending on the exact needs they have. Out of the box, it is easy to set up a container with no network access at all: for example, the following podman Containerfile:

```bash
# Use the latest Ubuntu image
FROM ubuntu:latest

# Install netcat (nc)
# ping does not work in rootless container as it uses suid or NET_CAP which are not allowed
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

Setting up such fine-grained firewalling on a container to container basis is a bit more complex that simply turning network on and off. In the following, we will discuss how this can be done, anno 2025 - things may change, and this may become easier in the future.

## Linux namespaces and podman containers: a handwavy explanation

Before

- handwavy explanation: I am not a specialist
- may change with time
- also, different cases depending on how a container is run; see e.g.
- linux kernel features including namespaces, chroot, etc make container possible. without going into details, allows to isolate and encapsulate how code is run, and give code the illusion (but also the power, locally), to run as root within its own little domain
- at a high level, when starting a podman container rootless, set up a dedicated namespace for the container process, and inside this namespace which is isolated from the machine it is as if the container process is root
- so within this namespace, can for example set up network and change its properties such as firewall, that otherwise on the host machine as a whole requires being root
- even within the container namespace, leverage linux features to make this flexible and controllable. for example, use capabilties to decide if can change network. in particular, NET_CAP is by default for a podman rootless container switched off once the app inside the container starts. compare:
  - podman run without NET_CAP and show cannot do an iptable command
  - podman run with NET_CAP and show can do an iptable command

## Firewalling podman rootless containers in an effective and safe way

- individually firewall containers, in a way that running apps in the container cannot circumvent
- note that here for rootless podman container, with NET_CAP not set ie the container at runtime has no network modification capability; if doing things differently, your mileage may vary
- for this: put the firwalling into the container namespace, which is used to set up the container network, before the NET_CAP is dropped
- once the container is set and the NET_CAP get dropped, the container user app gets started; at this point, firewalling has been set up in the container network namespace, and this is not modifiable any longer as long as NET_CAP has been dropped

## Examples and tests

- 101 example on my repository at: XXREF
- disclaimer: for bad reasons use iptables, should probably use something more recent anno 2025 (nft or similar)
- walkthrough of the repo
  - hook configuration
  - hooked script
  - run with the hooks loaded and an annotation that put the hooks on for the container
- running tests and checks

## To go further

- discussion at XXLINK
- podman repository and documentation may be more up to date
- asked for simpler to use hooks, for exmaple providing them together with podman run command - this may become simpler in the future!
