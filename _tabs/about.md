---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
title: About
---

I'm **Nguyen Ngoc Anh Hao** (Hao Nguyen), a senior embedded software engineer specializing in
embedded Linux, Yocto, and platform security. Born 17 November 2000, based in Ho Chi Minh City,
Vietnam, and open to remote work or relocation.

For 4+ years I've shipped production Linux firmware for security cameras, drones, and IoT
devices. I own the platform layer end to end: Yocto / OpenEmbedded BSPs, Secure Boot and
disk-encryption chains, and A/B OTA pipelines on NVIDIA Jetson, Ambarella, and NXP i.MX SoCs.
I led the migration of a deployed camera fleet from Ubuntu to a custom Yocto distribution, ran
field upgrades on live devices, and have upstreamed patches to the Mender project. Day to day I
work across hardware, firmware, and customer teams in Vietnam, China, and Europe.

## Experience

### Embedded Linux Engineer · Iritech
*June 2026 – present · Remote · iris-scanner biometric device*

- Designed the Yocto build and OS image for an iris-scanner product on a custom NXP i.MX6UL
  board and on SigmaStar SoCs.
- Owned platform security compliance — Secure Boot, encrypted storage, and a hardened rootfs —
  through the customer's security review.
- Coordinated daily with hardware and software teams in Vietnam and China from board bring-up
  to delivery.

### Software Engineer · Qualgo Technologies
*Jan 2026 – May 2026 · Ho Chi Minh City · drone and robotics embedded systems*

- Architected a modular Yocto OS stack shared across NVIDIA Jetson, Raspberry Pi, and
  BeagleBone, with A/B OTA updates, Secure Boot, and dm-verity for rootfs integrity.
- Published a reusable Yocto meta-layer that lets customers build their own OS images and OTA
  pipelines without vendor hand-holding.
- Developed robot applications on ROS 2, across the full robotics middleware stack.

### Software Engineer · Motorola Solutions
*Jul 2023 – Jan 2026 · Ho Chi Minh City · security camera platforms (NVIDIA Jetson, Ambarella)*

- Led the migration from NVIDIA's Ubuntu (L4T) image to a custom Yocto build system; owned
  meta-tegra integration and custom BSP layers across Jetson Orin Nano / NX products.
- Shipped production firmware for the L6Q solar-powered, LTE-connected field camera and
  platform software for the L6A Jetson Orin camera.
- Implemented full-disk encryption, Secure Boot, and read-only root filesystems; supported
  UEFI-level debugging on NVIDIA platforms.
- Built and maintained OTA pipelines on Mender and SWUpdate; upstreamed patches to the
  open-source Mender project.
- Ran critical field upgrades moving deployed cameras from Ubuntu 18.04 to 22.04 over the OTA
  pipeline with no bricked units.
- Ported the Ambarella SDK into custom Yocto meta-layers; managed systemd services for process
  and resource control.

### 5G Software Engineer · DEK Technologies (Ericsson)
*Apr 2022 – Jul 2023 · Ho Chi Minh City · Ericsson 5G Radio Unit software*

- Developed and maintained C++ components for Ericsson 5G Radio Unit software in a global Agile
  team across Vietnam, China, and Croatia.
- Wrote unit and integration tests with Google Test and JCAT; diagnosed and fixed customer
  field-deployment issues as Level 3 support.
- Improved team throughput with Bash tooling, Gerrit code reviews, and CI/CD pipeline updates.

### 5G Software Engineer Intern · TMA Solutions
*Nov 2021 – Mar 2022 · Ho Chi Minh City*

- Built a client–server chatbot with Unix socket programming in C; studied telecom and network
  protocols (OSI, TCP/IP).

## Skills

| Area | |
|:--|:--|
| Embedded Linux | Yocto Project / OpenEmbedded, BitBake recipes and meta-layers, BSP development (meta-tegra, custom layers), Linux for Tegra, systemd, D-Bus, U-Boot / UEFI boot flow |
| Security | Secure Boot (UEFI, signed UKI), full-disk encryption, dm-verity, read-only rootfs, secure elements (NXP SE05x, PKCS#11), mbedTLS |
| OTA | Mender, SWUpdate, A/B updates with rollback |
| Languages | C, C++17/20, Python, Bash, Go |
| Platforms | NVIDIA Jetson Orin (Nano / NX), Ambarella, NXP i.MX6UL, SigmaStar, ARM Cortex-A, Raspberry Pi, BeagleBone |
| Middleware & tools | ROS 2, Qt, GStreamer / H.265 hardware encoding, Google Test, JCAT, Git, Gerrit, CI/CD, QEMU, cross-compilation toolchains |

## Projects

- **[orin-nx-camera-app](https://github.com/haonguy3n/orin-nx-camera-app)** · C++20, Yocto,
  Jetson Orin NX — dual-sensor MIPI camera software: hardware-encoded H.265 streaming over a
  custom encrypted USB transport (ECDHE P-256 + ChaCha20-Poly1305), on-device GPU face
  detection, A/B OTA via SWUpdate, and a Qt host viewer.
- **[se05x](https://github.com/haonguy3n/se05x)** · C++, mbedTLS, NXP Plug & Trust —
  provisioning CLI for the NXP SE05x secure element: on-chip RSA key generation, CSR and
  certificate binding for device identity, exposed at runtime through PKCS#11; cross-compiled
  for ARM Linux.
- **[osb](https://github.com/haonguy3n/osb)** · Go, QEMU, UEFI — single-binary OS builder
  producing bootable Alpine / Debian / Ubuntu images for x86_64 and arm64 with signed UKI
  Secure Boot, dm-verity read-only roots, A/B rollback, SBOM output, and reproducible,
  content-addressed builds.

## Open source

Patch contributor to the [Mender](https://mender.io) OTA project; active in the NVIDIA
meta-tegra and OpenEmbedded / Yocto communities.

## Education

**B.S. Electronics & Communications Engineering** · Ho Chi Minh City University of Science
(HCMUS, VNU-HCM) · 2018 – 2022 · Minor in Embedded Systems

## Get in touch

- Email: [{{ site.social.email }}](mailto:{{ site.social.email }})
- GitHub: [@haonguy3n](https://github.com/haonguy3n)
- LinkedIn: [hao-nguyen-018961202](https://www.linkedin.com/in/hao-nguyen-018961202/)
