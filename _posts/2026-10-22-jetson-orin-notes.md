---
title: "Jetson Orin notes: inference and the frame path"
date: 2026-10-22 14:00:00 +0700
categories: [Embedded Linux]
tags: [jetson, machine-learning]
description: >-
  New public notes on TensorRT inference and zero-copy camera frames on Jetson Orin, kept generic
  and free of project-specific details.
---

The new [Jetson Orin notes](/jetson-orin/) are live: a public place for the parts of edge-vision
work that are worth keeping after a project ends.

- [Inference on Jetson Orin](/jetson-orin/inference/) — ONNX to a TensorRT plan, build versus load,
  build memory, precision validation, plan-cache keys and the checklist before trusting a model on a
  device.
- [Camera frames on Jetson Orin](/jetson-orin/frames/) — sensor to NVMM to GPU, the copy budget, the
  zero-copy contract, GPU preprocessing, lifetime and backpressure.

The notes are deliberately generic: no product or customer names, no private model names, no code
from a private repository and no performance figures. The patterns and the public references are the
point.
