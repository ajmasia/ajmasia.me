---
title: "A NixOS emergency"
canonicalURL: https://ajmasia.me/en/posts/2022/a-nixos-emergency
author: Antonio José Masiá
tags:
    - nixos
date: 2022-11-06
draft: false
---

## Background

This afternoon, while doing some cleaning on my work computer, I noticed that I was running low on space in the system partition. One of the advantages of NixOS is that every time you make changes to the configuration, dependencies are kept linked so you can revert to them as needed. However, I accidentally deleted all the system's bootloader files that point to each of these generations.

My intention was only to keep the latest one, but I ended up deleting them all. The solution would have been simple at that moment if I had realized what had happened: `sudo nixos-rebuild switch`. But, anticipating things, I restarted the system and... panic moment, no version of the system appeared, making it impossible to start.

> There's nothing like letting things sit to see them from a new perspective.

Given the situation and after trying several resources without luck, I decided to leave it temporarily, reassured by the knowledge that I would find a solution and that the system was still there.

## Cooling-Off Phase

When you get stuck on something and keep going in circles without solving anything, the best thing to do is to cool off. After this period of calm, I started reading documentation about it and quickly saw the light. The solution was seemingly simple: rebuild a generation from the existing configuration in the partition. And, how do you do this when you can't access any TTY of the system? Fortunately, we have Live USBs.

## Resolution Guide

1. First, you need a USB with the NixOS image. In my case, I always have one with the minimal installation. I like using the terminal ;-)
2. Once the Live USB is booted and to avoid having to constantly use sudo, run `sudo -i`
3. Now we need an internet connection so that NixOS can bring everything it needs to build a new generation of the system. For this, follow these steps:

```bash
# enable wifi connection
systemctl start wpa_supplicant

# connect to our wifi network via the cli
wpa_cli

# execute the following sequence of commands
add_network
set_network 0 ssid "myhomenetwork"
set_network 0 psk "mypassword"
set_network 0 key_mgmt WPA-PSK
enable_network 0

# exit the cli and verify our network connection
ping 8.8.8.8
```

4. Next, we proceed to mount our partitions, in my case:

```bash
mount /dev/nnvm0n1p1 /mnt/boot
mount /dev/nnvm0n1p3 /mnt
mount /dev/nnvm0n1p4 /mnt/home
swapon /dev/nvm0n1p2
```
5. Now we can enter our system by executing `$(which nixos-enter)`. This command allows us to access a NixOS installation from the Live USB.
6. Once inside, we will go to our user `su <user>` and simply run a `sudo nixos-rebuild switch` to rebuild our system and thus have a new generation, and voilà...
Once this new system generation is built, we will regain access from the boot loader.

## Learnings

> Experiences without learning are useless.

In this case, it would have been very useful to have clearly known the consequences of an error in the operation I was performing. Had that been the case, caution would have shone much brighter. When it comes to delicate system operations, no amount of caution is too much. Fortunately, NixOS is not just any Linux distribution and has a solution for every problem ;-).

After this episode, I am even more convinced that NixOS is the Linux distribution that has surprised me the most by far. My intention is to continue delving deeper and gradually share my experiences and learnings with you. Meanwhile, in case you're curious, here's a link to the NixOS documentation :-).

