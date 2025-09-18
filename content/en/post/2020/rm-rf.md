---
layout: post
title: Ops Tale The Day rm -rf /* Took My Box
category: linux
date: 2018-11-20
tags:
- linux
- rm
---

# Ops Tale: The Day rm -rf /* Took My Box

## The Trigger

Users reported mobile web showing a blank page. On desktop: same — HTML loads, but a core JS fails. Error: “We’re sorry but house doesn’t work properly without JavaScript enabled…”. Another network error: `ERR_INCOMPLETE_CHUNKED_ENCODING`.

Tried the usual:

- Restart the Docker container — no luck
- Roll back build — no luck
- Restart nginx — no luck
- Nginx logs — nothing helpful

Lightbulb: maybe disk full. `df -h`: 99.99% used. One Docker container using 34G — probably Elasticsearch. Killed it, cleared data, restarted nginx — back to normal. Problem solved… for now.

But ES needs to come back. Let’s attach and mount a separate data disk.

## The Mistake

Followed the cloud console to attach a disk, partitioned, and attempted to mount. Saw a warning, tried another device, mounted again, then “clean up the disk” — and ran:

```sh
rm -rf ./*
```

Suddenly some files were “busy”, then `ls` vanished, `cat` vanished, `cp` failed… only `cd` still worked. Yep — I had mounted the system disk path and wiped it.

## Postmortem

- Didn’t format before mounting; first mount failed
- Didn’t check why; tried another mount blindly
- Mounted the system disk path and deleted blindly
- `rm -rf ./*` — game over

## Recovery Attempt

Services still running via Docker; images kept in registry, so recoverable. But configs/certs lived on the host. With most tools gone, not much to read. A teammate suggested checking for snapshots — I had a system image from two weeks earlier.

Restored from image, updated service images — back online.

## Takeaways

- Separate data and system disks; don’t let app data kill the OS
- Enable snapshot policy if budget allows; at worst you roll back a week
- Before destructive ops, create a system image — a literal lifesaver

