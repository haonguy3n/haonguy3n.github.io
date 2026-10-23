---
title: NVMM and NvBufSurface on Jetson Orin
layout: page
permalink: /jetson-orin/nvmm/
toc: true
tags: [jetson]
description: >-
  What NVMM buffers are, how NvBufSurface describes them, the CPU, CUDA and dma-buf access paths,
  pitch-linear versus block-linear layouts, and the synchronization rules for Jetson Orin.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/jetson-orin/' | relative_url }}">← Jetson Orin notes</a></p>

> NVMM is the multimedia memory used by the Jetson camera, video and display stack. The public
> handle is **`NvBufSurface`**. It is not one DRAM type: it is a buffer descriptor for DMA-capable
> memory that can be mapped by the CPU, registered with CUDA, or shared as a dma-buf.

<div class="jo-flow">
  <div class="jo-step"><strong>Producer</strong><span>camera, decoder, converter</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>NvBufSurface</strong><span>one DMA-capable buffer</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Views</strong><span>CPU, CUDA, dma-buf, VIC/NVENC</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Consumer</strong><span>no pixel copy</span></div>
</div>

## What “NVMM” means

“NVMM” is commonly used for the Jetson multimedia memory/buffer stack. The API you program against
is **`NvBufSurface`** from the Jetson Linux Multimedia API.

An NVMM buffer can be:

- allocated by a camera, decoder or converter;
- backed by system DRAM (Jetson has unified CPU/GPU memory);
- **pitch-linear** or **block-linear**;
- mappable by the CPU, or only usable by hardware;
- exported/imported as a **dma-buf** file descriptor.

It is **not** a single memory type and it is not automatically a raw CPU pointer.

## The descriptor

```text
NvBufSurface
├── batchSize
├── numFilled
└── surfaceList ──► NvBufSurfaceParams[0]
                     ├── width, height
                     ├── layout: NVBUF_LAYOUT_PITCH or NVBUF_LAYOUT_BLOCK_LINEAR
                     ├── planeParams: width, height, pitch, bytesPerPix
                     └── mappedAddr
                          ├── addr[0]     CPU mapping
                          └── eglImage    EGL mapping
```

The pixels are behind the descriptor. `mappedAddr` is only valid after the corresponding map call.

## CPU access

```cpp
#include <EGL/egl.h>
#include <nvbufsurface.h>
#include <cstdint>

NvBufSurface* surface = nullptr;

NvBufSurfaceCreateParams params{};
params.gpuId = 0;
params.width = 1920;
params.height = 1080;
params.colorFormat = NVBUF_COLOR_FORMAT_RGBA;
params.layout = NVBUF_LAYOUT_PITCH;
params.memType = NVBUF_MEM_DEFAULT;

if (NvBufSurfaceCreate(&surface, 1, &params) != 0)
{
    // allocation failed
}

if (NvBufSurfaceMap(surface, 0, 0, NVBUF_MAP_READ_WRITE) != 0)
{
    // mapping failed
}

if (NvBufSurfaceSyncForCpu(surface, 0, 0) != 0)
{
    // cache sync failed
}

NvBufSurfaceParams const& plane = surface->surfaceList[0];
auto* base = static_cast<std::uint8_t*>(plane.mappedAddr.addr[0]);
int const pitch = static_cast<int>(plane.planeParams.pitch[0]);
int const width = static_cast<int>(plane.planeParams.width[0]);
int const height = static_cast<int>(plane.planeParams.height[0]);

// Address of pixel (x, y):
// base + y * pitch + x * bytesPerPixel

NvBufSurfaceUnMap(surface, 0, 0);
NvBufSurfaceDestroy(surface);
```

Rules:

- `NvBufSurfaceSyncForCpu` before the CPU reads data written by hardware.
- `NvBufSurfaceSyncForDevice` after the CPU writes data that hardware will read.
- `NvBufSurfaceUnMap` when the CPU view is no longer used.
- `NvBufSurfaceDestroy` only for a surface **you created**.

## GPU access with EGL/CUDA

```cpp
#include <cuda_egl_interop.h>
#include <nvbufsurface.h>

if (NvBufSurfaceMapEglImage(surface, 0) != 0)
{
    // EGL mapping failed
}

auto eglImage = static_cast<EGLImageKHR>(surface->surfaceList[0].mappedAddr.eglImage);

cudaGraphicsResource_t resource = nullptr;
if (cudaGraphicsEGLRegisterImage(&resource, eglImage,
                                 cudaGraphicsRegisterFlagsReadOnly) != cudaSuccess)
{
    // registration failed
}

cudaEglFrame frame{};
if (cudaGraphicsResourceGetMappedEglFrame(&frame, resource, 0, 0) != cudaSuccess)
{
    // mapped frame failed
}

if (frame.frameType == cudaEglFrameTypePitch)
{
    auto* pixels = static_cast<std::uint8_t*>(frame.frame.pPitch[0].ptr);
    int const pitch = static_cast<int>(frame.frame.pPitch[0].pitch);
    // A kernel can read pixels directly: pixel = pixels + y * pitch + x * bpp.
}
else
{
    cudaArray_t array = frame.frame.pArray[0];
    // Block-linear: copy out with cudaMemcpy2DFromArray, or convert with VIC.
}

cudaGraphicsUnregisterResource(resource);
NvBufSurfaceUnMapEglImage(surface, 0);
```

The two CUDA frame types are the critical detail:

| NvBufSurface layout | CUDA frame type | Kernel access |
|---|---|---|
| `NVBUF_LAYOUT_PITCH` | `cudaEglFrameTypePitch` | device pointer + pitch; read directly |
| `NVBUF_LAYOUT_BLOCK_LINEAR` | `cudaEglFrameTypeArray` | `cudaArray`; copy out or convert |

Do not cast a block-linear CPU mapping to a row-major image. The project-style check is:

```cpp
if (surface->surfaceList[0].layout != NVBUF_LAYOUT_PITCH)
{
    // CPU mapping is tiled, not rows
}
```

## Memory layouts

### Pitch-linear

```text
base + 0 * pitch    row 0: [px0][px1][px2]...[pxW-1][ padding ]
base + 1 * pitch    row 1: [px0][px1][px2]...[pxW-1][ padding ]
base + 2 * pitch    row 2: ...
```

- `pitch >= width * bytesPerPixel`;
- CPU and CUDA pitch pointers work;
- coalesced if adjacent threads read adjacent pixels.

### Block-linear

```text
Logical image                    Physical storage
┌──────────────────────┐         ┌──────┬──────┬──────┐
│                      │         │ tile │ tile │ tile │
│       1920 x 1080    │   ──►   ├──────┼──────┼──────┤
│                      │         │ tile │ tile │ tile │
└──────────────────────┘         └──────┴──────┴──────┘

A tile is contiguous; rows are not.
```

- hardware-friendly;
- CPU mapping is tiled, not row-major;
- CUDA sees a `cudaArray`, not a pitch pointer;
- VIC can convert it to a pitch-linear surface.

## dma-buf and sharing

A dma-buf is a Linux kernel object that shares a buffer between drivers or processes. NVMM buffers
can be exported/imported as dma-buf file descriptors; the same physical pages then have different
mappings:

```text
physical pages
   ├── CPU mmap        → CPU virtual address
   ├── dma-buf fd      → another driver/process
   ├── IOMMU           → VIC/NVENC/ISP device address
   └── EGLImage        → CUDA virtual address
```

`NvBufSurfaceFromFd` / `NvBufSurfaceImport` are the import side; the producer owns the export. A
dma-buf file descriptor can be reused after the buffer is freed, so a cached registration must be
invalidated when the fd lifetime ends.

## The camera pipeline flow

```text
Camera / Argus
     │
     ▼
NvBufSurface
     ├── CPU: NvBufSurfaceMap + SyncForCpu → inspect/copy
     ├── CUDA: MapEglImage + register → kernel
     ├── VIC: NvBufSurfTransform → new NvBufSurface
     ├── NVENC: NvBufSurface → encoded stream
     └── dma-buf: fd → another process/driver
```

No stage needs to copy the pixels to the CPU. The buffer is passed by handle and mapped into the
consumer’s address space.

## Synchronization and lifetime

| Action | Required call |
|---|---|
| CPU reads after hardware writes | `NvBufSurfaceSyncForCpu` |
| CPU writes before hardware reads | `NvBufSurfaceSyncForDevice` |
| CPU pointer no longer used | `NvBufSurfaceUnMap` |
| CUDA/EGL view no longer used | `cudaGraphicsUnregisterResource`, `NvBufSurfaceUnMapEglImage` |
| Surface you created is done | `NvBufSurfaceDestroy` |

The producer owns the buffer. A consumer must hold a reference/lease for as long as it uses the
surface, and must not call `NvBufSurfaceDestroy` on a buffer it did not create.

## Common mistakes

- Treating `GstBuffer` data as raw pixels. In an NVMM pipeline the data is often the
  `NvBufSurface` descriptor; map it with the API.
- Using `width` instead of `pitch` for the row stride.
- Casting a block-linear CPU mapping to a row-major image.
- Reading CPU data without `SyncForCpu`.
- Writing CPU data without `SyncForDevice`.
- Destroying a surface owned by the camera/decoder pipeline.
- Caching a dma-buf or EGL registration after the fd is reused.
- Assuming a CUDA kernel can read a block-linear surface through a pitch pointer.

## NVIDIA references

- [Jetson Linux API Reference — NvBufSurface](https://docs.nvidia.com/jetson/archives/r39.2/ApiReference/group__ds__aaa.html)
- [CUDA graphics interoperability](https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/graphics-interop.html)
- [Linux kernel dma-buf](https://docs.kernel.org/driver-api/dma-buf.html)

<div class="jo-note">
  NVMM is not “a pointer to DMA memory”. It is a buffer manager plus a descriptor. The pixels stay
  in one allocation; CPU, CUDA, VIC and NVENC each get a different view of it, with their own
  layout, address and synchronization rules.
</div>
