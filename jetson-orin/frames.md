---
title: Camera frames on Jetson Orin
layout: page
permalink: /jetson-orin/frames/
toc: true
description: >-
  A sensor-to-GPU frame path on Jetson Orin: NVMM, dmabuf, GStreamer caps, NvBufSurface views, GPU
  preprocessing, lifetime and the copies that hide in plain sight.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/jetson-orin/' | relative_url }}">← Jetson Orin notes</a></p>


> The goal is simple to state and easy to lose: the sensor buffer should reach the GPU and the model
> without a CPU copy of the pixels. The hard part is not the first step; it is proving that every consumer
> on the path still accepts the same buffer.

## 1. The path I want

<div class="jo-flow">
  <div class="jo-step"><strong>Sensor</strong><span>MIPI CSI-2</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>ISP / VIC</strong><span>scale, debayer, convert</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>NVMM</strong><span>dmabuf-backed buffer</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>GStreamer</strong><span>memory:NVMM caps</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>View</strong><span>pointer, format, pitch</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>GPU</strong><span>preprocess + inference</span></div>
</div>

After the view, different consumers want different things:

<div class="jo-flow">
  <div class="jo-step"><strong>GPU preprocess</strong><span>NV12 → RGB, resize, normalize</span></div>
  <div class="jo-step"><strong>TensorRT</strong><span>bind the device buffer</span></div>
  <div class="jo-step"><strong>Encoder</strong><span>NVENC / RTSP</span></div>
  <div class="jo-step"><strong>JPEG</strong><span>snapshot / event</span></div>
  <div class="jo-step"><strong>CPU fallback</strong><span>only when the fast path cannot serve</span></div>
</div>

<p class="jo-caption">The branch is the design decision: every consumer either accepts the view or asks for a copy.</p>

## 2. What NVMM actually means

NVMM is NVIDIA's name for a memory buffer that can be shared by the camera, the ISP, the video processor,
the GPU and the encoder. It is backed by a dmabuf (or a platform buffer handle); the CPU may be able to map
it, but mapping it is not the normal path.

- In GStreamer, the caps say `video/x-raw(memory:NVMM)`. Without the `(memory:NVMM)` part, the element is
  allowed to negotiate system memory, and your “zero-copy” pipeline quietly becomes a download.
- `nvvidconv` is the VIC scaler/converter on Jetson. It can keep its output in NVMM; `videoconvert` is a CPU
  element and will not.
- An `appsink` can receive an NVMM buffer, but the caps you ask for decide whether it stays there. Ask for
  the format and memory type you actually want, and check the negotiated caps at runtime.
- `NvBufSurface` describes the buffer: width, height, pitch, plane count, bytes per pixel and the device/CPU
  addresses. The pitch is not always the width; rows can be padded.

NVMM is not the same as pinned host memory, and it is not automatically a CUDA pointer. Depending on the
JetPack release, the buffer is imported into CUDA through the platform API (dmabuf import, an
`NvBufSurface` device address, or a graphics resource). Check the API for the version you are on; do not
assume that a `void*` from a CPU map is device-accessible.

## 3. Where copies hide

| Stage | What you want | A common accident |
|---|---|---|
| Sensor → ISP | no CPU pass | a pipeline that negotiates system memory |
| ISP → NVMM | stay in NVMM | `videoconvert`, or a cap without `memory:NVMM` |
| NVMM → view | pointer + pitch | `gst_buffer_map` then `memcpy` into a packed array |
| View → GPU | device import | `cudaMemcpy` from a CPU mapping |
| Preprocess | one GPU pass | `cv::cvtColor` + `cv::resize` + `cv::Mat` copy |
| Postprocess | small CPU result | copying the full frame back to the CPU |
| Encode/JPEG | hardware path | converting to system memory first |

A useful discipline is to write a **copy budget** for one frame before writing code:

```text
sensor -> ISP:        0 CPU copies
ISP -> NVMM:          0
NVMM -> GPU:          0
preprocess:           1 GPU pass (color + resize + normalize fused)
inference:            0 pixel copies
postprocess:          small metadata only
encode/JPEG:          hardware path, 0 CPU copies
```

Then verify the budget with counters and traces, not with the pipeline diagram.

## 4. The zero-copy contract

A frame view is a small struct, not an image:

```cpp
struct FrameView {
    const void* data;   // plane 0
    int width;
    int height;
    int pitch;          // bytes per row, may be > width * bytesPerPixel
    int format;         // NV12, RGBA, GRAY8, ...
    int planes;
};
```

The contract around it matters as much as the pointer:

- **Pitch and planes.** NV12 has a Y plane and an interleaved UV plane; RGBA is one plane. A view that
  ignores pitch works on the test image and corrupts the edge of the real frame.
- **Lifetime.** The view is only valid while the buffer is held. A `GstBuffer` reference, a dmabuf fd or a
  frame lease must outlive the GPU work. Do not unmap a buffer while a kernel is still reading it.
- **Synchronisation.** If the CPU wrote the buffer, sync it for the device before the GPU reads it; if the
  GPU wrote it and the CPU will read it, sync it back. On some paths the buffer is cache-coherent, on others
  it is not; know which one you have.
- **Format contract.** NV12 and RGBA have different color matrices and ranges. VIC, OpenCV and CUDA may not
  agree on the YUV-to-RGB conversion. Compare a known color target, not only the gray image.
- **Backpressure.** A bounded pool with a drop policy is part of zero copy. If a slow consumer holds every
  buffer, the camera has nowhere to write; the producer must never block the sensor forever.

## 5. GPU preprocessing

The usual pipeline is NV12 (or RGBA) → RGB → resize → normalize to the model's input tensor. Do it in one
CUDA kernel when possible, and write the result into the TensorRT input binding:

1. import the NV12 surface as a CUDA-accessible buffer (or map it through the platform API);
2. run one kernel that reads the Y and UV planes with the real pitch, converts to RGB, resizes to the model
   input and applies the model's scale/mean;
3. write directly into the input tensor's device memory;
4. enqueue TensorRT on the same stream;
5. synchronise once, at the boundary where the result is needed.

That is one GPU pass over the pixels. The CPU only handles the crop rectangle and the metadata. If the model
expects a packed RGB tensor, the kernel can write that layout; you do not need a CPU-visible image.

Not everything belongs on the GPU. A tiny postprocessing step on the CPU can be cheaper than a kernel launch
and a synchronisation. The rule is to keep **pixels** on the GPU and let **metadata** go back to the CPU.

## 6. Lifetime, pools and the slow consumer

The camera has a fixed number of buffers. If a consumer holds one, it is not available to the producer.
That is why the buffer pool and the lease policy are part of the frame path, not an implementation detail.

- Keep a small pool (enough for the pipeline depth, not “as many as possible”).
- Give each consumer a bounded number of leases; when it is full, drop the frame and count it.
- Never let a slow socket, disk write or JPEG encoder block the capture thread.
- For a second process, a shared-memory ring with a lease protocol avoids a socket copy; for a second
  consumer in the same process, a view plus a reference is enough.
- At shutdown, drain the pool and join the consumers before unmapping anything.

## 7. What to measure

| Question | Tool | What to look for |
|---|---|---|
| Is the frame still in NVMM? | GStreamer debug / negotiated caps | `memory:NVMM`, the expected format |
| Where is the CPU spending time? | `top -H`, `perf`, `tegrastats` | a `camera`/`vidconv` thread doing pixel work |
| Is there a copy? | `nsys`, `strace`, custom counters | `memcpy`, `cudaMemcpy`, unexpected CPU maps |
| Is the GPU the bottleneck? | `nsys`, TensorRT timing | kernels, gaps, synchronization |
| Is memory stable? | `tegrastats`, `/proc/<pid>/status` | RSS growing per frame or per reconnect |
| Is the path healthy? | frame counters, drop counters, PTS | drops, stale frames, buffer starvation |

Measure with the real consumers connected: RTSP, the web view, JPEG snapshots and inference at the same
time. A pipeline is only zero-copy for the combination you actually run.

## 8. Checklist

- [ ] The caps contain `memory:NVMM` for every buffer that should stay on the device.
- [ ] No `videoconvert` or another CPU converter on the hot path.
- [ ] The view uses the real pitch and plane layout.
- [ ] The GPU path imports the buffer instead of copying a CPU mapping.
- [ ] Preprocessing is fused into one GPU pass where it matters.
- [ ] The buffer is held for the whole GPU/encoder operation and released exactly once.
- [ ] A slow consumer drops frames and does not stall capture.
- [ ] The color conversion matches the product's golden image.
- [ ] A 30-minute soak keeps RSS, frame rate and latency flat.
- [ ] The fallback path is tested, not just written.

## Public references

- [NVIDIA Jetson Linux Developer Guide](https://docs.nvidia.com/jetson/archives/)
- [GStreamer documentation](https://gstreamer.freedesktop.org/documentation/)
- [NVIDIA TensorRT documentation](https://docs.nvidia.com/deeplearning/tensorrt/)
- [NVIDIA Jetson benchmarks and power modes](https://developer.nvidia.com/embedded/jetson-benchmarks)
