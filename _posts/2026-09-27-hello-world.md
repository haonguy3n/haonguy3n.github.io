---
title: "Hello, world"
date: 2026-09-27 10:00:00 +0700
categories: [Meta]
tags: [embedded-linux, yocto]
description: Why I'm starting this site, and what I plan to put on it.
pin: true
---

Every platform project I've worked on leaves behind the same thing: a scratch file of details
that took days to find. Which layer a BitBake variable actually has to be set in. What changed in
the boot chain when Secure Boot was turned on. Which OTA state a device can't recover from if it
loses power at the wrong moment.

Those notes never leave my machine, and six months later I can't find them either. So: this site.

## What I'll write about

- **Yocto in practice** — BSP integration, meta-layer structure, and keeping builds reproducible
  across boards and SoC vendors.
- **Secure Boot and device hardening** — boot chains, disk encryption, dm-verity, read-only root
  filesystems, and secure elements.
- **OTA updates** — A/B pipelines with Mender and SWUpdate, and upgrading devices that are
  already deployed in the field.
- **Debugging stories** — a symptom, the wrong theories I chased, and what it actually was.

## How to reach me

If something here helped you, or if I got something wrong, I'd like to hear about it.
Email is [{{ site.social.email }}](mailto:{{ site.social.email }}), and I'm on
[GitHub](https://github.com/haonguy3n) and
[LinkedIn](https://www.linkedin.com/in/hao-nguyen-018961202/).

First real post soon.
