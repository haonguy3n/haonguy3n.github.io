---
title: Jetson Orin
layout: page
icon: fas fa-microchip
order: 6
toc: true
tags: [jetson]
description: >-
  Field notes on NVIDIA Jetson Orin for edge vision: TensorRT inference, precision and memory, and a
  camera frame path that stays in NVMM from the sensor to the GPU.
---

{% include jetson-orin-style.html %}


> These are generic engineering notes from building edge vision systems on Orin-class devices.
> They deliberately leave out product details, customer information, model names and private code.
> The patterns are what transfer; the confidential parts stay out of this site.

A Jetson is a shared-memory computer. The CPU, GPU, camera ISP and video encoder all read and write the same
LPDDR. That is where its efficiency comes from, and it is the source of most surprises: there is no separate
VRAM to watch, and a buffer that is described as “on the GPU” can still be copied three times before the model
sees it.

<div class="jo-flow">
  <div class="jo-step"><strong>Model</strong><span>ONNX graph and weights</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Plan</strong><span>TensorRT engine for this device</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Frame</strong><span>NVMM buffer, no CPU copy</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Answer</strong><span>boxes, classes, text</span></div>
</div>

## Trails

- [CUDA `.cu` files on Jetson Orin]({{ '/jetson-orin/cuda/' | relative_url }}) — kernels, grids,
  warps, the memory hierarchy, a public vectorAdd example and the build/error rules.
- [NVMM and NvBufSurface on Jetson Orin]({{ '/jetson-orin/nvmm/' | relative_url }}) — one
  DMA-capable buffer, the CPU/CUDA/dma-buf views, pitch vs block-linear, and synchronization.
- [Delta updates on Jetson Orin]({{ '/jetson-orin/delta-updates/' | relative_url }}) — read-only root,
  raw ext4, A/B slots, SWUpdate/RAUC, and how to generate X → Y safely.
- [Inference on Jetson Orin]({{ '/jetson-orin/inference/' | relative_url }}) — what a TensorRT build
  actually does, why build memory is much larger than the finished plan, how to validate FP16/INT8, and how to
  keep a plan cache honest.
- [Camera frames on Jetson Orin]({{ '/jetson-orin/frames/' | relative_url }}) — the zero-copy contract, where
  copies hide, how to keep a frame as a view, and what to measure before believing “zero copy”.

For the longer, hardware-agnostic background, see [Machine Learning on the Edge]({{ '/ml/' | relative_url }}):
the field guide from ONNX to a CPU, GPU or NPU.

The process side is in [Coding rules and PR checklist]({{ '/engineering/' | relative_url }}) — small
changes, explicit ownership, tests that include a failure, measured claims and a rollback.

## The rules I keep coming back to

1. **Build once, load many times.** A plan is a binary; the builder is a compiler. Treat the build as a
   one-time, memory-hungry step and the load as the normal path.
2. **Follow the frame, not the label.** “GPU path”, “zero copy” and “hardware encoder” are claims about a
   whole chain, not about one element. One system-memory cap or one `cv::Mat` can undo all of it.
3. **Validate the decision, not only the tensor.** A tiny numerical difference is fine until it changes a
   class, a box or a decoded string. Test the output the product actually consumes.
4. **Measure copies and memory at the boundary.** `tegrastats`, `nsys`, `/proc` and a deliberately broken
   pipeline tell you more than an architecture diagram.
5. **Keep a fallback.** A slightly slower path that still produces a frame beats a perfect zero-copy path
   that deadlocks when a consumer is slow.

## What this section is not

- Not a product guide, a benchmark report or a reproduction of a private design.
- No customer names, proprietary model names, private file paths, credentials or performance figures.
- The public references at the bottom of each note are the places to check for the exact API on your JetPack
  version.

<div class="jo-note">
  The first note is about the model side; the second is about the pixels. They meet at one question:
  <em>what is the cheapest path from a sensor buffer to a TensorRT input?</em>
</div>

<p class="jo-caption">Tag: <a href="{{ '/tags/jetson/' | relative_url }}">#jetson</a></p>
