---
layout: post
title: "Linux trivia: mounting a folder from another user, with custom permissions, using bindfs"
date: 2022-03-19 19:00:00 +0200
categories: jekyll update
---

Sometimes, it is convenient to mount a folder as readonly for another user, under other permissions than what the "true, original" files have. For example, root may be the owner of some folder containing data that need to be backed up on some remote server, but it would be better for security and isolation to copy these data using a restricted account with read only access to the data. 

This is exactly what ```bindfs``` is for. For exammple, using this:

```bash
sudo bindfs -r -u $(id -u) -g $(id -g) SOME_SOURCE SOME_MOUNT_POINT
```

will mount (as read-only thanks to ```-r```) ```SOME_SOURCE``` into ```SOME_MOUNT_POINT```, for the user and group of the user issuing the command. Running this command through ```sudo``` will make it work, even if ```SOME_SOURCE``` is owned by root, with access for root only.

In order to umount the folder, you need to simply:

```bash
sudo umount SOME_MOUNT_POINT
```

