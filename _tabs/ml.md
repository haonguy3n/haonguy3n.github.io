---
title: Machine Learning on the Edge
layout: page
icon: fas fa-brain
order: 5
toc: true
description: >-
  A practical field guide for embedded engineers: how a model goes from ONNX to a CPU, GPU or NPU, what its
  inputs and outputs really are, and how to verify a precision change without breaking the product.
---

{% raw %}

> I learn a new SoC by following one frame from the sensor to the answer and naming every box it passes through.
> This page is that path for a camera that reads licence plates. No calculus, no training run required — C++,
> a terminal and curiosity are enough. Terms in **bold** are in the [glossary](#12-glossary).

## How to use this page

- **10 minutes:** sections [1](#1-the-big-picture), [3](#3-inputs-the-models-abi) and [4](#4-outputs-numbers-into-answers).
- **Porting to a new board:** [6](#6-precision-fp32-tf32-fp16-int8), [7](#7-hardware-cpu-gpu-dla-npu), [8](#8-memory-and-data-movement), [9](#9-porting-recipe).
- **Testing yourself:** the [exercises](#10-exercises) and the [quiz](#11-quiz).

## 1. The big picture {#1-the-big-picture}

A model is a **graph of mathematical operations** (convolutions, additions, activations) plus millions of
**weights** — numbers learned from data. The graph lives in an `.onnx` file. To run it, a device needs a compiler,
a precision and a memory path. Everything else is plumbing.

| Idea | C++ analogy | In practice |
|---|---|---|
| `model.onnx` | source code | portable between machines |
| compiler | `gcc`/`clang` | TensorRT, TFLite, RKNN, OpenVINO, HailoRT… |
| engine / plan / HEF | executable | tied to the device, runtime version and shapes |
| runtime | loader | loads the engine, allocates memory, runs it |
| config (`*TRTConfig.ini`) | ABI | input size, layout, value range, thresholds |

The same ONNX compiles for a CPU, a Jetson GPU and an i.MX NPU — *if* the compiler supports the operations and
the precision. That "if" is the whole porting story.

What a compiler does for you:

1. parses the graph and checks the operators;
2. **fuses** layers (`Conv + BatchNorm + ReLU` becomes one kernel);
3. picks a **kernel** (and its tiling/vectorisation) per layer from a library — this is why two builds of the
   same model can give slightly different bits;
4. plans memory (weights, activations, workspace);
5. applies the chosen **precision** (FP32, FP16, INT8…).

## 2. One frame through a plate camera

```text
sensor frame (1440×1080, 8-bit, padded rows)
   │  whole frame resized to 352×288, one channel
   ▼
plate detector (a small YOLO)  ──▶  [1, 2475, 6]
   │  2475 candidate boxes; keep confidence ≥ 2 %, then NMS at 20 % IoU
   ▼
a few boxes  ──▶  crop each plate
   │  resize to 256×64, scale values to −1…1
   ▼
OCR (SVTR)  ──▶  [1, 25, 39]  ──▶  25 time steps × 39 characters  ──▶  "51F-12345"
   │
   ├── state classifier   112×64×1  ──▶  [1, 67]   ──▶  "AR"
   ├── colour classifier  150×150×3 ──▶  [1, 10]   ──▶  "white"
   └── make/model         224×224×3 ──▶  four heads
```

Two numbers that do **not** mean what they look like:

- `2475` are **candidate boxes**, not plates. Almost all die at the confidence threshold and NMS.
- `25` are **time steps**, not characters. **CTC** decoding collapses repeats and blanks, so 25 steps can
  produce a 7-character plate.

## 3. Inputs: the model's ABI {#3-inputs-the-models-abi}

The camera gives bytes (`uint8`, 0–255). The model wants floats in a fixed shape. The config defines four things:

| Knob | Values | Meaning |
|---|---|---|
| `WIDTH_INPUT` × `HEIGHT_INPUT` × `CHANNEL_INPUT` | e.g. 352×288×1 | resize target and channel count |
| `INPUT_FORMAT` | `NHWC` / `NCHW` | pixel-interleaved vs channel planes |
| `STANDARDIZE_INPUT_TYPE` | `none` / `127` / `255` / `std` | value mapping, below |
| `SWAP_RB_CHANNEL`, `PADDING_TYPE`, `SKIP_MATCH_CHANNEL` | 0/1 | channel order, stretch vs warp, gray↔RGB |

Value mappings:

| `std` key | Formula | Typical range |
|---|---|---|
| `none` | `v` | 0…255 |
| `127` | `v / 127.5 − 1` | −1…1 |
| `255` | `v / 255` | 0…1 |
| `std` | `(v − mean) / max(stddev, 1/√N)`, computed **per image** | roughly −2…2 |

Get this wrong and the network sees noise. It is the first thing to check when "the model is bad".

<div class="mlg-card" markdown="0">
  <p class="mlg-title">Tensor shape &amp; size calculator</p>
  <div class="mlg-grid">
    <label>Width <input id="mlg-w" type="number" min="1" value="352" /></label>
    <label>Height <input id="mlg-h" type="number" min="1" value="288" /></label>
    <label>Channels <input id="mlg-c" type="number" min="1" value="1" /></label>
    <label>Batch <input id="mlg-b" type="number" min="1" value="1" /></label>
    <label>Layout
      <select id="mlg-layout"><option>NHWC</option><option>NCHW</option></select>
    </label>
    <label>Type
      <select id="mlg-type">
        <option value="4" selected>FP32</option>
        <option value="2">FP16</option>
        <option value="1">INT8</option>
      </select>
    </label>
  </div>
  <p class="mlg-presets">
    <button type="button" class="mlg-btn" data-w="352" data-h="288" data-c="1" data-b="1" data-layout="NHWC">plate detector</button>
    <button type="button" class="mlg-btn" data-w="256" data-h="64" data-c="1" data-b="1" data-layout="NCHW">plate reader</button>
    <button type="button" class="mlg-btn" data-w="112" data-h="64" data-c="1" data-b="1" data-layout="NCHW">state</button>
    <button type="button" class="mlg-btn" data-w="150" data-h="150" data-c="3" data-b="1" data-layout="NHWC">colour</button>
    <button type="button" class="mlg-btn" data-w="416" data-h="320" data-c="3" data-b="1" data-layout="NCHW">car finder</button>
  </p>
  <p class="mlg-out" id="mlg-out">—</p>
  <p class="mlg-hint" id="mlg-hint">—</p>
</div>

Rules that save hours:

- `NHWC` is `[N, H, W, C]`; `NCHW` is `[N, C, H, W]`. Same numbers, different memory order.
- `B` (batch) is how many items go through together; on a camera it is usually 1.
- A model trained with one mapping cannot be fed another (`std` vs `none` is not a small difference).
- A gray model fed RGB (or the reverse) needs the channel conversion the config asks for.

## 4. Outputs: numbers into answers {#4-outputs-numbers-into-answers}

A model almost never outputs the answer directly. It outputs **numbers**, and a decoder turns them into one.

| Task | Output shape | What it is | Decoder → answer |
|---|---|---|---|
| classifier | `[B, C]` | **logits**, one per class | softmax → argmax → label file |
| multi-head | `[B,115]`, `[B,1234]`, … | several heads | argmax on each |
| detector | `[B, N, 6]` | N candidates: box, objectness, class | confidence → **NMS** → boxes |
| OCR | `[B, T, C]` | T steps × C characters | argmax per step → **CTC** → string |
| embedding | `[B, 128]` | a vector, not a class | distance between vectors |
| score | `[B, 1]` | one value | scale |
| landmark | `[B, 28, 72, 72]` | heat maps | argmax per map → points |

- **Logits vs probabilities.** Logits are raw scores. **Softmax** turns them into probabilities that sum to 1.
  **Argmax** picks the largest — and softmax is monotonic, so the argmax is the same before and after. That is why
  a decision-level check compares the argmax, not every float.
- **NMS (non-maximum suppression).** Many candidate boxes overlap. Sort by score, keep the best, drop boxes whose
  **IoU** (intersection over union) with a kept box is above the threshold. `IoU = intersection area / union area`.
- **CTC.** The OCR sees 25 steps for a 5–8 character plate. Decoding picks the best character per step, then removes
  repeated characters and a special blank, then maps indices through the charset file.

The **output contract** is more than the shape: output names and order (`OUTPUT_NAMES`), and whether a tensor is
already softmax (`softmax:0`) or raw logits. A decoder that applies softmax twice still gets the same argmax but
wrong confidences.

## 5. Training vs inference

- **Training** adjusts the weights: show an example, compare the output with the label, compute the **loss**, and
  nudge every weight in the direction that lowers it (**gradient descent**, with **backpropagation** for the
  gradients). It happens in **epochs** and **batches**, on a GPU, usually with PyTorch.
- **The output of training is the weights**, saved as a checkpoint and exported to ONNX — not "a colour" or "a plate".
- **Inference** just evaluates the function. That is what the camera does.
- **Overfitting** is memorising the training set; that is why a **validation** set and a **test** set exist.
- **Distribution shift** is why a model can be 98 % accurate in the lab and fail on your bench: IR vs colour,
  day vs night, rain, angle, plate material. Your data is not the training data.

## 6. Precision: FP32, TF32, FP16, INT8 {#6-precision-fp32-tf32-fp16-int8}

| Format | Bits (sign/exp/mantissa) | Range | Notes |
|---|---|---|---|
| FP32 | 1/8/23 | huge | the reference |
| TF32 | 1/8/10 | huge | TensorRT's **default** for fp32 layers on newer GPUs |
| FP16 | 1/5/10 | ±65 504 | half the bytes; tensor cores |
| BF16 | 1/8/7 | huge | less precision, more range |
| INT8 | 8-bit integer | −128…127 + scale | 4× smaller; needs calibration |

**FP16.** 10 mantissa bits ≈ 3 decimal digits. The gap between representable numbers (**ULP**) grows with the
magnitude; values above 65 504 become `Inf`, and `Inf − Inf` becomes `NaN`. Setting the FP16 flag is a *permission*:
TensorRT keeps FP32 for layers where FP16 hurts. Engine inputs and outputs usually stay FP32, so your code does not
change.

**INT8.** Each tensor gets a **scale** (and often a zero-point): `q = round(x / scale)`, back `x ≈ q · scale`.
Choosing the scale from real data is **calibration**. Values outside the calibrated range **clip**. If calibration
alone loses too much accuracy, **quantization-aware training (QAT)** simulates the rounding during training.

**Hardware chooses the precision.** On a plain CPU, FP16 is usually *slower* (it converts to FP32); INT8 helps only
with the right instructions. On a GPU/NPU, FP16/INT8 are the fast path. Never assume; measure.

**Verify decisions, not just floats.** Compare the argmax, the boxes after NMS, the OCR string. Report
non-finite values as hard failures, errors relative to the magnitude, and the class/character decisions. Two builds
are never bitwise identical: kernel selection changes too. Open-source tools such as
[Polygraphy](https://github.com/NVIDIA/TensorRT/tree/main/tools/Polygraphy) do the same comparison on a PC.

## 7. Hardware: CPU, GPU, DLA, NPU {#7-hardware-cpu-gpu-dla-npu}

| Class | Examples | Precision | Toolchain | Strength | Limit |
|---|---|---|---|---|---|
| CPU | x86, Arm SBC | FP32, INT8 | ONNX Runtime, OpenVINO, Arm NN | flexible, everywhere | big models are slow |
| GPU | Jetson Orin | FP32/TF32/FP16/INT8 | TensorRT | fast, fused kernels | power, shared memory |
| DLA | Jetson Orin | FP16/INT8 | TensorRT (offload) | low power per frame | limited ops, static shapes |
| NPU | NXP i.MX 8M Plus | INT8 | NXP eIQ / TFLite | efficient | op subset, quantisation |
| microNPU | NXP i.MX 93 (Ethos-U65) | INT8 | Vela + TFLite Micro | tiny, always-on | very small models |
| Edge TPU | Coral | INT8 | TFLite | cheap | strict op list |
| NPU | Hailo-8 | INT8 | HailoRT / Dataflow Compiler | high throughput | whole-graph compile |
| NPU | Rockchip RK3588 | INT8/FP16 | RKNN toolkit | price/performance | vendor op list |
| NPU | Intel (OpenVINO) | FP16/INT8 | OpenVINO | CPU/iGPU/NPU | Intel only |

`TOPS` is a peak INT8 number, not a frame rate: real throughput depends on memory, operators, batch and fallbacks.
"i.MX" is not one target — 8M Plus, 93 and 95 have different NPUs and different compilers.

What changes when you port:

1. **Supported operators** — unsupported ops fail the compile or fall back to the CPU (a copy and a latency spike).
2. **Precision** — INT8 requires calibration and an accuracy check on camera-like data.
3. **Shapes** — NPUs often want static shapes; batch 1, fixed height/width.
4. **Memory** — NVMM on Jetson, dma-buf on i.MX, plain RAM on a CPU board. Zero-copy needs an adapter.
5. **Post-processing** — keep NMS, CTC and softmax on the CPU; they are cheap, variable, and poorly supported on NPUs.

## 8. Memory and data movement {#8-memory-and-data-movement}

- **Weights** are the file size: a 14 M-parameter model is ~58 MB in FP32, ~29 MB in FP16, ~14 MB in INT8.
- **Activations** are the scratch tensors during a run; they grow with batch and resolution.
- **Bandwidth** often wins over compute: moving 4× fewer bytes is why INT8 helps even when the maths is fast enough.
- **Zero-copy**: Jetson shares DRAM between CPU and GPU (NVMM); i.MX uses dma-buf; a CPU board copies. Every added
  copy costs milliseconds at 15 fps.
- **Batch** trades latency for throughput; it also multiplies activations.

Measure per stage — capture, convert, preprocess, inference, postprocess, encode. The neural network is often not
the biggest line.

## 9. Porting recipe {#9-porting-recipe}

1. **Freeze a reference.** Run the ONNX in FP32 on the CPU over a fixed, camera-like set; store outputs *and*
   decoded decisions.
2. **Compile** for the target: TRT plan, TFLite/eIQ, RKNN, HEF…
3. **Check operators**; move unsupported ones out of the graph.
4. **Quantise** if required: calibration set, per-channel, PTQ then QAT if needed.
5. **Compare decisions**, not raw floats: confusion matrix, boxes by IoU, exact OCR string.
6. **Measure**: latency per stage, CPU load, RAM, power, and the **fallback count**.
7. **Keep a safety net**: a sampled reference check and a permanent CPU path if the accelerator fails.

## 10. Exercises {#10-exercises}

<details markdown="0">
<summary>1. How big is the input tensor?</summary>
<p>A detector takes 352×288, one channel. Compute the values and the bytes at FP32, FP16 and INT8.</p>
<p><strong>Answer:</strong> 352 × 288 = 101,376 values. FP32 = 405,504 B (396 KiB); FP16 = 198 KiB; INT8 = 99 KiB.
The input is usually the biggest tensor in the pipeline, which is why its layout and copies matter.</p>
</details>

<details markdown="0">
<summary>2. Why are 2475 rows not 2475 plates?</summary>
<p>The detector emits one row per candidate location, not one per object. Almost all candidates are background;
the confidence threshold removes most, and NMS removes the overlapping rest. The decoder, not the network, produces
the final list.</p>
</details>

<details markdown="0">
<summary>3. OCR outputs [1, 25, 39]. Why 25 and 39?</summary>
<p>25 is the number of time steps the sequence head produces; 39 is the number of characters in the charset
(including a blank). Decoding picks one character per step, collapses repeats and blanks, then maps the indices
through the charset file. 25 steps can produce a 7-character plate.</p>
</details>

<details markdown="0">
<summary>4. Which precision would you pick?</summary>
<p>CPU-only board: FP32 (or INT8 with calibration). Jetson: FP16 first, INT8 when the accuracy check passes.
i.MX 8M Plus NPU: INT8, because that is what the NPU accelerates. Always compare decisions against the FP32
reference.</p>
</details>

## 11. Quiz {#11-quiz}

<div class="mlg-quiz" id="mlg-quiz" markdown="0">

<div class="mlg-q" id="mlg-q1">
<p><strong>1.</strong> What can you move between two different devices unchanged?</p>
<label><input type="radio" name="q1" value="a" /> the TensorRT engine</label>
<label><input type="radio" name="q1" value="b" /> the ONNX graph</label>
<label><input type="radio" name="q1" value="c" /> the plan cache</label>
<p class="mlg-exp"><strong>Answer: b.</strong> The ONNX graph is portable; an engine/plan is compiled for one runtime,
GPU and shape set.</p>
</div>

<div class="mlg-q" id="mlg-q2">
<p><strong>2.</strong> A FP16 activation reaches 70,000. What happens?</p>
<label><input type="radio" name="q2" value="a" /> it rounds to 65,504</label>
<label><input type="radio" name="q2" value="b" /> it becomes Infinity, and may later become NaN</label>
<label><input type="radio" name="q2" value="c" /> it clamps to 255</label>
<p class="mlg-exp"><strong>Answer: b.</strong> FP16's largest finite value is 65,504; overflow gives Inf, and
arithmetic on Inf can produce NaN. That is why non-finite outputs are a hard failure in a precision check.</p>
</div>

<div class="mlg-q" id="mlg-q3">
<p><strong>3.</strong> What does INT8 need that FP16 does not?</p>
<label><input type="radio" name="q3" value="a" /> a GPU</label>
<label><input type="radio" name="q3" value="b" /> calibration data and scales</label>
<label><input type="radio" name="q3" value="c" /> a larger batch</label>
<p class="mlg-exp"><strong>Answer: b.</strong> Integers need a scale (and often a zero-point) per tensor or channel,
chosen from representative data. Clipping and rounding are the accuracy risks.</p>
</div>

<div class="mlg-q" id="mlg-q4">
<p><strong>4.</strong> OCR output is <code>[1, 25, 39]</code>. What is the 25?</p>
<label><input type="radio" name="q4" value="a" /> time steps</label>
<label><input type="radio" name="q4" value="b" /> characters</label>
<label><input type="radio" name="q4" value="c" /> classes</label>
<p class="mlg-exp"><strong>Answer: a.</strong> 25 sequence steps, each with 39 character scores; CTC turns them into
a string.</p>
</div>

<div class="mlg-q" id="mlg-q5">
<p><strong>5.</strong> On a plain CPU, FP16 usually…</p>
<label><input type="radio" name="q5" value="a" /> doubles the speed</label>
<label><input type="radio" name="q5" value="b" /> fails to run</label>
<label><input type="radio" name="q5" value="c" /> is not faster, because it converts to FP32</label>
<p class="mlg-exp"><strong>Answer: c.</strong> Most CPUs have no native FP16 arithmetic path; the conversion costs
time. INT8 can help with the right instructions.</p>
</div>

<p><button type="button" class="mlg-btn" id="mlg-check">Check answers</button></p>
<p class="mlg-out" id="mlg-score"></p>
</div>

## 12. Glossary {#12-glossary}

<dl class="mlg-dl" markdown="0">
<dt>Activation</dt><dd>A tensor produced between layers; the runtime scratch memory.</dd>
<dt>Batch</dt><dd>How many items are processed together (the first tensor dimension).</dd>
<dt>Calibration</dt><dd>Choosing integer scales from representative data for quantisation.</dd>
<dt>Checkpoint</dt><dd>The saved weights after training, before export to ONNX.</dd>
<dt>CTC</dt><dd>Connectionist Temporal Classification: decodes a sequence of character scores into a string.</dd>
<dt>DLA</dt><dd>Deep Learning Accelerator: a fixed-function CNN block on some Jetson chips.</dd>
<dt>Embedding</dt><dd>A vector that represents an object; compared by distance, not by argmax.</dd>
<dt>Engine / plan</dt><dd>The compiled model for one device, runtime and shape set.</dd>
<dt>Epoch</dt><dd>One full pass over the training data.</dd>
<dt>Fusion</dt><dd>Merging several operations into one kernel.</dd>
<dt>INT8</dt><dd>8-bit integer arithmetic with per-tensor or per-channel scales.</dd>
<dt>IoU</dt><dd>Intersection over union: how much two boxes overlap.</dd>
<dt>Kernel</dt><dd>One device's implementation of an operation.</dd>
<dt>Logits</dt><dd>Raw class scores before softmax.</dd>
<dt>Loss</dt><dd>A number measuring how wrong a prediction is during training.</dd>
<dt>NMS</dt><dd>Non-maximum suppression: removes overlapping duplicate boxes.</dd>
<dt>NPU</dt><dd>Neural processing unit: a fixed-function accelerator for convolutions and matrix maths.</dd>
<dt>ONNX</dt><dd>An open graph format for models; the portable source.</dd>
<dt>Operator</dt><dd>One node of the graph, such as Conv, Add, Relu, Softmax.</dd>
<dt>Overfitting</dt><dd>Memorising the training set instead of generalising.</dd>
<dt>PTQ / QAT</dt><dd>Post-training quantisation / quantisation-aware training.</dd>
<dt>Softmax</dt><dd>Turns logits into probabilities that sum to 1.</dd>
<dt>Tensor</dt><dd>An n-dimensional array: shape, type, layout.</dd>
<dt>TOPS</dt><dd>Peak trillions of operations per second; a marketing ceiling, not a frame rate.</dd>
<dt>ULP</dt><dd>Unit in the last place: the gap between two representable numbers at a given magnitude.</dd>
<dt>Weights</dt><dd>The numbers learned by training; most of the model file.</dd>
<dt>Zero-copy</dt><dd>The accelerator reads the camera buffer directly, without a memcpy.</dd>
</dl>

## 13. Reading

- [Dive into Deep Learning (d2l.ai)](https://d2l.ai/) — chapters 1–3 are enough to start.
- [Stanford CS231n: convolutional networks](https://cs231n.github.io/convolutional-networks/) — the best diagrams
  of what a convolution actually does.
- [Netron](https://netron.app/) — open an ONNX in the browser and look at the graph.
- [ONNX](https://onnx.ai/) — the format and the operator set.
- [TensorRT: accuracy considerations](https://docs.nvidia.com/deeplearning/tensorrt/latest/inference-library/accuracy-considerations.html) —
  FP16 overflow, ULP, sensitive operations.
- [TensorRT: best practices](https://docs.nvidia.com/deeplearning/tensorrt/latest/performance/best-practices.html).
- [TFLite: post-training quantisation](https://www.tensorflow.org/lite/performance/post_training_quantization).
- [Polygraphy](https://github.com/NVIDIA/TensorRT/tree/main/tools/Polygraphy) — compare two runtimes or precisions
  on the same input.
- [f1tenth lab 8](https://github.com/f1tenth-cmu/f1tenth_lab8) — a course lab that times FP16 vs FP32 correctly
  (inference only, at least 100 runs).

{% endraw %}

<style>
.mlg-card { border: 1px solid rgba(128,128,128,.35); border-radius: .6rem; padding: 1rem 1.1rem; margin: 1.4rem 0; background: rgba(128,128,128,.06); }
.mlg-title { margin: 0 0 .7rem; font-weight: 600; }
.mlg-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(110px, 1fr)); gap: .7rem; }
.mlg-grid label { display: flex; flex-direction: column; gap: .25rem; font-size: .85rem; }
.mlg-grid input, .mlg-grid select { padding: .35rem .45rem; border: 1px solid rgba(128,128,128,.45); border-radius: .35rem; background: transparent; color: inherit; }
.mlg-presets { display: flex; flex-wrap: wrap; gap: .4rem; margin: .8rem 0 .2rem; }
.mlg-btn { padding: .3rem .6rem; border: 1px solid rgba(128,128,128,.45); border-radius: .35rem; background: transparent; color: inherit; cursor: pointer; font-size: .85rem; }
.mlg-btn:hover { border-color: var(--link-color, #2a7ae2); }
.mlg-out { font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; font-size: .92rem; margin: .9rem 0 .2rem; }
.mlg-hint { font-size: .82rem; opacity: .8; margin: 0; }
.mlg-quiz .mlg-q { margin: 1rem 0; padding: .1rem 0; }
.mlg-quiz label { display: block; margin: .15rem 0 .15rem .2rem; }
.mlg-exp { display: none; margin: .45rem 0 0 .2rem; padding-left: .6rem; border-left: 3px solid rgba(128,128,128,.45); font-size: .92rem; }
.mlg-ok { border-left: 3px solid #2da44e; padding-left: .6rem; }
.mlg-bad { border-left: 3px solid #f85149; padding-left: .6rem; }
.mlg-dl dt { font-weight: 600; margin-top: .6rem; }
.mlg-dl dd { margin: .1rem 0 .1rem 1.2rem; }
</style>

<script>
(function () {
  var ids = ['mlg-w', 'mlg-h', 'mlg-c', 'mlg-b', 'mlg-layout', 'mlg-type'];
  var el = {};
  ids.forEach(function (id) { el[id] = document.getElementById(id); });
  var out = document.getElementById('mlg-out');
  var hint = document.getElementById('mlg-hint');
  if (!out || !hint) { return; }

  function human(bytes) {
    if (bytes >= 1048576) { return (bytes / 1048576).toFixed(2) + ' MiB'; }
    if (bytes >= 1024) { return (bytes / 1024).toFixed(1) + ' KiB'; }
    return bytes + ' B';
  }
  function update() {
    var w = Math.max(1, parseInt(el['mlg-w'].value, 10) || 1);
    var h = Math.max(1, parseInt(el['mlg-h'].value, 10) || 1);
    var c = Math.max(1, parseInt(el['mlg-c'].value, 10) || 1);
    var b = Math.max(1, parseInt(el['mlg-b'].value, 10) || 1);
    var layout = el['mlg-layout'].value;
    var bpp = parseInt(el['mlg-type'].value, 10) || 4;
    var typeName = el['mlg-type'].options[el['mlg-type'].selectedIndex].text;
    var shape = layout === 'NCHW' ? [b, c, h, w] : [b, h, w, c];
    var values = w * h * c * b;
    var item = w * h * c;
    out.textContent = 'shape [' + shape.join(', ') + ']  ·  ' + values.toLocaleString('en-US') +
      ' values  ·  ' + human(values * bpp) + ' at ' + typeName;
    hint.textContent = 'per item: ' + item.toLocaleString('en-US') + ' values  ·  FP32 ' + human(item * 4) +
      '  ·  FP16 ' + human(item * 2) + '  ·  INT8 ' + human(item);
  }
  ids.forEach(function (id) {
    el[id].addEventListener('input', update);
    el[id].addEventListener('change', update);
  });
  Array.prototype.forEach.call(document.querySelectorAll('.mlg-preset'), function (btn) {
    btn.addEventListener('click', function () {
      el['mlg-w'].value = btn.dataset.w;
      el['mlg-h'].value = btn.dataset.h;
      el['mlg-c'].value = btn.dataset.c;
      el['mlg-b'].value = btn.dataset.b;
      el['mlg-layout'].value = btn.dataset.layout;
      update();
    });
  });
  update();
})();

(function () {
  var answers = { q1: 'b', q2: 'b', q3: 'b', q4: 'a', q5: 'c' };
  var check = document.getElementById('mlg-check');
  var score = document.getElementById('mlg-score');
  if (!check || !score) { return; }
  check.addEventListener('click', function () {
    var right = 0;
    Object.keys(answers).forEach(function (name) {
      var block = document.getElementById('mlg-' + name);
      var picked = document.querySelector('input[name="' + name + '"]:checked');
      var exp = block.querySelector('.mlg-exp');
      block.classList.remove('mlg-ok', 'mlg-bad');
      if (picked && picked.value === answers[name]) { right += 1; block.classList.add('mlg-ok'); }
      else { block.classList.add('mlg-bad'); }
      if (exp) { exp.style.display = 'block'; }
    });
    score.textContent = right + ' / 5 — ' + (right === 5 ? 'solid.' : 'read the explanation and try again.');
  });
})();
</script>
