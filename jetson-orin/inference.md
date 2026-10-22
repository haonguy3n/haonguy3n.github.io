---
title: Inference on Jetson Orin
layout: page
permalink: /jetson-orin/inference/
toc: true
description: >-
  From ONNX to a TensorRT plan on Jetson Orin: build versus load, precision validation, memory, plan
  caching and the checklist I use before trusting a model on a device.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/jetson-orin/' | relative_url }}">← Jetson Orin notes</a></p>

# Inference on Jetson Orin: from ONNX to a plan

> The short version: an ONNX file is source code, a TensorRT plan is a binary for one device and one set of
> shapes and precision, and the builder is a compiler that can use several times the memory of the binary it
> produces. Build it once; load it many times; validate what the product decides.

## 1. The path

<div class="jo-flow">
  <div class="jo-step"><strong>ONNX</strong><span>graph, operators, weights</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Parser</strong><span>import and check the graph</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Builder</strong><span>fuse, choose kernels, time tactics</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Plan</strong><span>serialized engine</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Runtime</strong><span>load, allocate, execute</span></div>
</div>

The same ONNX can produce different plans on two devices, or on the same device after a TensorRT upgrade.
The plan is tied to the model content, the runtime version, the GPU architecture, the input shapes, the batch
size and the precision. That is why a cache key has to include all of them, not just the model file name.

## 2. Build is not load

| | Build | Load |
|---|---|---|
| When | model, runtime, GPU, shape or precision changed | every normal start |
| Work | parse, fuse, select and time kernels, serialize | read the plan, allocate weights and a context |
| Peak memory | parser + network + workspace + weights + plan | plan + weights + context |
| Time | seconds to minutes | milliseconds |
| Failure mode | OOM, unsupported operator, no tactic | missing/stale plan, incompatible runtime |

The builder is allowed to be greedy. It keeps the parsed network, candidate kernels, timing data and the
serialized output alive at the same time. A plan that is a few megabytes can come from a build that needs
hundreds of megabytes or more. On a Jetson that memory is shared with the camera pipeline, the encoder and
the rest of the application, so a build is a system-wide event, not just an inference event.

Practical rules:

- Build **one model at a time** and do not keep a second builder or engine alive beside it.
- Destroy the builder and the network before loading the finished engine; do not overlap build and load
  peaks unless you have measured that you can afford it.
- Put the build on a start-up path that is allowed to take minutes, and keep the normal path load-only.
- On glibc, freeing a large build does not always return the pages to the OS immediately; an explicit
  `malloc_trim` after the build can help. It is a host-heap fix, not a TensorRT API. The device-side
  allocations and the CUDA context have their own lifetimes.

<div class="jo-note">
  If the build peak is the problem and the process must stay up, the strongest fix is to build in a
  short-lived child process and let it exit. The OS then reclaims the host heap, the device allocations and
  the context together. The application only loads the plan.
</div>

## 3. Precision is a product decision

FP32 is the reference. FP16 is usually the first useful step on Orin; INT8 can be much faster but needs a
calibration set and a careful accuracy check. A plan built for one precision cannot be mixed with another.

Tensor-level checks are necessary but not sufficient:

- compare the same input through the reference and the candidate;
- report max absolute difference, max relative difference and the distribution, not only the mean;
- for a classifier, compare the top-1 class and the margin;
- for a detector, compare boxes **after** NMS, not the raw tensor;
- for a sequence/OCR model, compare the decoded string, not only the per-step tensor;
- for a heatmap, compare the peak position, not only the argmax index;
- run real frames from the product's scenes, including the difficult ones: low light, motion, IR, rain,
  reflective plates, extreme angles.

A result is not “within noise” just because the maximum relative difference is small. If the decision
changes, it is a product change. Keep a short exception list for models that must stay in a wider precision,
and write down the evidence next to the exception.

<div class="jo-flow jo-flow--col">
  <div class="jo-step"><strong>FP32 reference</strong><span>same input, same preprocessing, same postprocessing</span></div>
  <div class="jo-arrow">↓</div>
  <div class="jo-step"><strong>Candidate (FP16 / INT8)</strong><span>same plan key, different precision</span></div>
  <div class="jo-arrow">↓</div>
  <div class="jo-step"><strong>Compare tensors and decisions</strong><span>max diff, top-1, boxes after NMS, decoded text</span></div>
  <div class="jo-arrow">↓</div>
  <div class="jo-step"><strong>Accept or add an exception</strong><span>the exception is a measured decision</span></div>
</div>

## 4. The plan cache is part of the product

A plan cache is only useful if a stale plan can never be loaded silently. The key should include:

- a content hash of the ONNX file, not only its path or modification time;
- the TensorRT version and the CUDA/driver generation;
- the GPU architecture or device tag;
- the precision, batch size and optimization profile;
- the model's input/output shapes and preprocessing contract.

Write the plan atomically (temporary file, flush, rename), write the stamp last, and sweep old plans on
start-up. A missing or mismatched stamp means “build”, never “try it and see”. The first start on a new image
is allowed to be slow; every later start should be a load.

## 5. Only build what the pipeline can call

A model that no reachable call path uses is still expensive if it is created: it loads weights, allocates a
context or two, starts worker threads and may build a plan on first start. Before adding a model, ask:

- which profile/configuration selects this model?
- which call site reaches it, under which condition?
- what happens if the condition is false: is the manager still created?
- how many sessions/contexts does it need, and how many callers can actually run at once?

Skipping an unused model is usually a larger and safer win than optimizing its kernel. The same applies to
extra sessions: a pool of two is only useful if two callers can genuinely overlap.

## 6. A checklist before trusting a model

1. Build on a clean plan cache and record the peak RSS and the time.
2. Start again with the cache present: it must be load-only, with no builder allocation.
3. Compare the new precision against the FP32 reference on a fixed golden set and on real frames.
4. Check the decision-level metric the product uses, not only the tensor metric.
5. Measure latency with the real batch size, preprocessing and postprocessing included.
6. Run a long soak: memory must be flat, the plan must not be rebuilt and the result must not drift.
7. Check thermal behaviour: a fanless Orin can be fast for one minute and slow for the next ten.
8. Keep the previous plan and a fallback path until the new one has passed on the device.

## Public references

- [NVIDIA TensorRT documentation](https://docs.nvidia.com/deeplearning/tensorrt/)
- [NVIDIA Jetson Linux Developer Guide](https://docs.nvidia.com/jetson/archives/)
- [ONNX](https://onnx.ai/)
- [NVIDIA Jetson benchmarks and power modes](https://developer.nvidia.com/embedded/jetson-benchmarks)
