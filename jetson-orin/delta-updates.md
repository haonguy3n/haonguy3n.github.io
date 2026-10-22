---
title: Delta updates on Jetson Orin
layout: page
permalink: /jetson-orin/delta-updates/
toc: true
tags: [jetson, updates]
description: >-
  A read-only A/B image, raw ext4, and a delta from version X to Y: how SWUpdate and RAUC
  fit on Jetson Orin, how to generate the delta, and what to test before trusting it.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/jetson-orin/' | relative_url }}">← Jetson Orin notes</a></p>

> A delta is only safe when the running system is **exactly** the archived old image. That is why
> this page starts with a read-only root and a raw ext4 image, not with the delta tool. The tool is
> the easy part.

<div class="jo-flow">
  <div class="jo-step"><strong>Old image X</strong><span>read-only, raw ext4, archived</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>New image Y</strong><span>built the same way</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Delta X→Y</strong><span>changed blocks only</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Inactive A/B slot</strong><span>verify, then switch</span></div>
</div>

## 1. The prerequisite: a read-only root

A block delta compares bytes at the same offset. If the running root is writable, logs, caches and
configuration change it, and the delta base no longer matches. The old slot must be **bit-for-bit**
the image that was archived.

What this means in practice:

- root is mounted **read-only**; the kernel command line does not contain `rw` for root;
- the ext4 filesystem has **no journal** and a **fixed UUID** (one UUID for the product, not random
  per build);
- persistent state lives on a separate writable partition (`/data`, `/mnt/...`), not under `/`;
- the application does not call `mount -o remount,rw /` and there is no init script that does;
- `/var`, `/tmp` and other volatile paths are tmpfs or bind-mounted from the writable partition.

A Yocto image recipe can express the base of that model:

```text
IMAGE_FEATURES += "read-only-rootfs"
IMAGE_FSTYPES = "ext4.zst"

# One fixed UUID for this product. A random UUID changes ext4 metadata on every build.
EXTRA_IMAGECMD:ext4 = "-O ^has_journal -U 11111111-2222-3333-4444-555555555555"
IMAGE_ROOTFS_SIZE = "8388608"
```

The exact variable names depend on your Yocto release; the three properties are the point:
**read-only root, no journal, fixed UUID**. Keep the image size fixed too, so the slot layout does
not change between X and Y.

Example `/etc/fstab` intent:

```text
/dev/root                         /       ext4  ro,noatime        0 1
/dev/disk/by-partlabel/data       /data   ext4  rw,noatime        0 2
tmpfs                             /var/volatile tmpfs defaults     0 0
```

On the device, verify rather than assume:

```sh
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /
findmnt -no TARGET,OPTIONS /data
touch /cannot-write            # must fail
touch /data/can-write          # must succeed
```

If the application still writes under `/opt`, `/var` or `/etc`, move that path to `/data` and
bind-mount it to the expected location. Do not “fix” it with a remount; that breaks the delta base.

## 2. A/B and raw ext4

Two root filesystems, A and B, are enough for a safe update:

1. boot from A;
2. write the new image to B (the inactive slot);
3. verify the written image;
4. arm the bootloader to switch to B;
5. if B does not boot, fall back to A.

The full update writes the **raw ext4 image**, not a tarball:

```text
software = {
    version = "2.0.0";
    images: (
        {
            filename = "rootfs.ext4";
            device = "/dev/disk/by-partlabel/rootfs_b";
            type = "raw";
            compressed = "zstd";
        }
    );
    bootenv: (
        { name = "boot_slot"; value = "b"; }
    );
};
```

`type = "raw"` is the important line. An `archive` handler unpacks files into a formatted filesystem;
a block delta needs the image bytes, so the slot must be a raw image with the same geometry.

## 3. SWUpdate example

SWUpdate packages a `.swu` (a cpio archive) with `sw-description` first, then the artifacts. It is a
good fit when you want one push model, A/B, signatures and your own installer.

### Full update

Use the raw image above. Sign `sw-description` with the private signing key; install it with:

```sh
swupdate -i update-2.0.0.swu
```

The bootloader environment decides which slot is active. The new slot is written first; the switch
happens only after the image and the post-install checks pass.

### Delta with rdiff

SWUpdate has an `rdiff_image` handler. Generate the patch from the exact old image:

```sh
rdiff signature rootfs-x.ext4 rootfs-x.sig
rdiff delta rootfs-x.sig rootfs-y.ext4 rootfs-x-to-y.rdiff
```

Then package the patch instead of the full image:

```text
images: (
    {
        type = "rdiff_image";
        filename = "rootfs-x-to-y.rdiff";
        device = "/dev/disk/by-partlabel/rootfs_b";
        properties: {
            rdiffbase = [ "/dev/disk/by-partlabel/rootfs_a" ];
        };
    }
);
```

`rdiffbase` is the running slot; `device` is the inactive slot. SWUpdate applies the patch to the
inactive slot, so a failure does not touch the running system.

**Caveat:** rdiff is excellent for tiny changes and can grow when the two images differ a lot. For
an ext4 image whose metadata or large regions moved, compare **4 KiB blocks** and write only the
changed offsets; that is the block-delta model. Whichever method you use, check the base before
writing and the result hash before switching.

### If you need a block delta

The shape is:

1. read the old image (or the running slot) in 4 KiB blocks;
2. hash each block, compare with the new image;
3. package only the changed blocks, with their offsets;
4. on the device, verify the old slot, write each changed block to the inactive slot, hash the
   result, then switch.

This is an installer you own. It is small, but it must be signed and tested like any other update
path. Do not ship a delta that writes before checking the base.

## 4. RAUC example

RAUC uses a bundle and a manifest. It has built-in **adaptive updates** for block devices; the bundle
still contains the full image, but the installer reads only the blocks it cannot find locally.

### Manifest

```ini
[update]
compatible=my-board
version=2.0.0

[image.rootfs]
filename=rootfs.ext4
adaptive=block-hash-index
```

Build a signed bundle:

```sh
rauc bundle --cert=release.crt --key=release.key manifest.raucm update-2.0.0.raucb
```

### Adaptive update

With `adaptive=block-hash-index`, RAUC creates a SHA-256 index for each 4 KiB block and stores the
index in the bundle. During installation it reads both slots and reuses blocks that already match;
it downloads only the missing blocks when the bundle is streamed over HTTP(S).

Important limits:

- the target must be a **block device** (raw ext4 works; a `.tar` bundle does not);
- adaptive updates are intended for **streaming**, not for a small offline `.raucb`;
- the index is about 0.8% of the image size;
- small changes can still mean a meaningful fraction of the bundle if the change pattern is
  scattered;
- the first adaptive update may generate the index on demand and take longer.

RAUC is a strong choice if you want a standard bundle, signature verification, D-Bus integration and
streaming without writing your own installer. SWUpdate is a strong choice if you want one `.swu`
push path, custom handlers and a block-delta installer tailored to the image.

## 5. Generate X → Y

The repeatable sequence:

1. **Build X** with the read-only/no-journal/fixed-UUID model; archive:
   - `rootfs-x.ext4.zst`;
   - the uncompressed `rootfs-x.ext4` (or a way to reproduce it);
   - its SHA-256 and version.
2. **Build Y** with the same machine, partition layout, filesystem size, block size, UUID and
   image properties. If any of those change, X is not a valid base.
3. **Choose the delta form**:
   - SWUpdate `rdiff` for a small, file-like change;
   - a 4 KiB block delta for a general ext4 change;
   - RAUC adaptive bundle for a streaming/block-index workflow.
4. **Package and sign** the delta or bundle.
5. **Install**:
   - verify the running slot is the exact X base;
   - write only the inactive slot;
   - verify the result hash before touching the bootloader;
   - arm the switch, reboot, mark the boot successful;
   - keep the old slot until the new one has booted.
6. **Fallback**: a missing/wrong base, a corrupt delta or a failed result must leave the old slot
   untouched and offer a full update.

## 6. Test before trusting it

- root is `ro`; `findmnt /` shows `ro`; a write to `/` fails; `/data` writes succeed;
- full raw update installs and boots;
- delta from the exact X installs and the result hash matches Y;
- wrong base is refused **before any write**;
- corrupt delta / wrong result is refused before the switch;
- power cut during patch, during sync and during bootloader arm leaves a bootable slot;
- A/B fallback works when the new slot is deliberately broken;
- a device on an old release gets a full update first;
- signed and unsigned packages behave as configured;
- repeated updates do not leak disk or grow the root;
- the update survives a consumer that is slow to stop (web/RTSP) or a full disk.

## 7. Common mistakes

- **`archive` instead of `raw`**: the slot becomes a filesystem tree, not a byte-identical image.
- **Writable root or journal**: metadata changes every boot; the base check fails or, worse, passes
  on a different image.
- **Random UUID or changing image size**: every build looks different before any app change.
- **`mount -o remount,rw /` in a script**: breaks the base and hides the real write path.
- **Delta without a base check**: writes a patch onto an unknown image.
- **No result check**: the bootloader switches to a half-written slot.
- **Keys or base images in the repository**: keep signing material offline and artifacts in release
  storage.
- **One update path only**: keep a full image fallback for every release.

## Public references

- [SWUpdate documentation](https://sbabic.github.io/swupdate/)
- [SWUpdate manual — rdiff handler](https://manpages.debian.org/experimental/swupdate/swupdate.1)
- [RAUC — adaptive updates](https://rauc.readthedocs.io/en/latest/advanced.html)
- [Yocto Project — image features](https://docs.yoctoproject.org/ref-manual/features.html)

<div class="jo-note">
  A delta is a promise that the old image is still exactly the old image. Read-only root, raw ext4
  and a checked base are what make that promise true; SWUpdate and RAUC are the packaging around it.
</div>
