---
layout: post
title: Ubuntu Ethernet Drops — Quick Reset Notes
slug: ubuntu-wifi-loss
categories: 
  - linux
date: 2023-09-22
image: xiaohei.png
tags:
  - Ubuntu
  - Linux
  - docker
---

## The Odd Failure

Desktop has two NICs: onboard Ethernet and a PCIe Wi‑Fi card. After reassembling hardware (P40 GPU tinkering) and rebooting, the wired NIC was “down”.

Previously I had fixed similar issues by deleting the old wired profile in Ubuntu and re‑adding it. This time I had no monitor available (TV too far for an Ethernet cable), so I limped along on Wi‑Fi for a while.

In theory, resetting the NIC should suffice. In `ifconfig`, a normal NIC shows `inet` fields; the broken one didn’t — the system wasn’t recognizing it properly.

`ifconfig down`/`up` didn’t help.

## Use nmtui (NetworkManager TUI)

`nmtui` is a curses-based TUI for NetworkManager.

Run `nmtui`, choose “Activate a connection”, then review the devices. In the problematic case the Device shows an invalid NIC id.

- Delete the invalid connection
- Add a new one, entering the NIC name you saw in `ifconfig`
- Save and activate

Done.

## Root Cause

After hardware changes, the system reinitialized and assigned a new device id to the onboard NIC, while your saved network profile still referenced the old id. Resetting the device alone doesn’t fix that — you must recreate the connection profile so it binds to the new device id.

