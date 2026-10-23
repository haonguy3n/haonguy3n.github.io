---
title: CUDA .cu files on Jetson Orin
layout: page
permalink: /jetson-orin/cuda/
toc: true
tags: [jetson]
description: >-
  What a .cu file is, how nvcc turns a CUDA kernel into GPU code, a public vectorAdd example,
  the thread and memory hierarchy, and the build and error-checking rules for Jetson Orin.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/jetson-orin/' | relative_url }}">← Jetson Orin notes</a></p>

> A `.cu` file is CUDA C++ source compiled by `nvcc`. It contains code that runs on the GPU —
> kernels — plus the host-side code that launches them. The file extension is a build instruction:
> `nvcc` reads it, the ordinary C++ compiler does not.

<div class="jo-flow">
  <div class="jo-step"><strong>Host</strong><span>allocate, copy or map, launch</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Grid</strong><span>blocks scheduled on the GPU</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Threads</strong><span>one element each</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Result</strong><span>synchronize, verify</span></div>
</div>

## What `.cu` means

| Part | Compiled by | Runs on |
|---|---|---|
| `__global__ void kernel(...)` | `nvcc` → PTX/cubin | GPU |
| `kernel<<<grid, block>>>(...)` | host side of `nvcc` | CPU launches GPU work |
| ordinary C++ functions | host compiler | CPU |
| `__device__` functions | `nvcc` | GPU, called from a kernel |

A `.cpp` file can call CUDA runtime functions, but it cannot define a `__global__` kernel or use
the `<<<...>>>` launch syntax unless it is compiled as CUDA. That is why kernels live in `.cu`.

## A public `vectorAdd` example

This is the standard NVIDIA example shape: one thread per element, a bounds check, and a grid
large enough to cover the array.

```cpp
#include <cstddef>
#include <cstdio>
#include <cstdlib>
#include <cuda_runtime.h>

#define CUDA_CHECK(call)                                                        \
    do                                                                          \
    {                                                                           \
        cudaError_t error = (call);                                             \
        if (error != cudaSuccess)                                               \
        {                                                                       \
            std::fprintf(stderr, "%s:%d: %s\n", __FILE__, __LINE__,             \
                         cudaGetErrorString(error));                            \
            std::exit(EXIT_FAILURE);                                            \
        }                                                                       \
    } while (0)

__global__ void vectorAdd(const float* a, const float* b, float* c, int n)
{
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n)
    {
        c[i] = a[i] + b[i];
    }
}

int main()
{
    constexpr int n = 1 << 20;
    constexpr std::size_t bytes = n * sizeof(float);

    float* a = nullptr;
    float* b = nullptr;
    float* c = nullptr;

    // Unified memory: the same allocation is addressable by the CPU and the GPU.
    CUDA_CHECK(cudaMallocManaged(&a, bytes));
    CUDA_CHECK(cudaMallocManaged(&b, bytes));
    CUDA_CHECK(cudaMallocManaged(&c, bytes));

    for (int i = 0; i < n; ++i)
    {
        a[i] = 1.0f;
        b[i] = 2.0f;
    }

    constexpr int threads = 256;
    int const blocks = (n + threads - 1) / threads;

    vectorAdd<<<blocks, threads>>>(a, b, c, n);

    // Check the launch, then wait for the GPU.
    CUDA_CHECK(cudaGetLastError());
    CUDA_CHECK(cudaDeviceSynchronize());

    for (int i = 0; i < n; ++i)
    {
        if (c[i] != 3.0f)
        {
            std::fprintf(stderr, "mismatch at %d: %f\n", i, c[i]);
            return EXIT_FAILURE;
        }
    }

    CUDA_CHECK(cudaFree(a));
    CUDA_CHECK(cudaFree(b));
    CUDA_CHECK(cudaFree(c));

    std::puts("vectorAdd: ok");
    return EXIT_SUCCESS;
}
```

Save it as `vector_add.cu`:

```sh
nvcc -O2 -arch=sm_87 vector_add.cu -o vector_add
./vector_add
```

`sm_87` is the compute capability of the Orin GPU family. If you leave out `-arch`, `nvcc` targets
a default; for a deployable binary, name the architecture explicitly.

## The thread hierarchy

```text
Grid
├── Block 0
│   ├── Warp 0  (32 threads)
│   ├── Warp 1
│   └── ...
├── Block 1
│   └── ...
└── ...

A thread knows:
  threadIdx.x/y/z   position inside its block
  blockIdx.x/y/z    position of its block
  blockDim.x/y/z    block size
  gridDim.x/y/z     number of blocks
```

For a 1D array:

```cpp
int i = blockIdx.x * blockDim.x + threadIdx.x;
if (i < n) { ... }
```

A warp is 32 threads. Threads in a warp execute together; divergence (different branches in the
same warp) serializes those branches. For memory, adjacent threads reading adjacent floats is
coalesced and much faster than a strided pattern.

## The memory hierarchy

```text
GPU
├── registers          per thread, fastest
├── shared memory      per block, explicit, small
├── L1/L2 cache        hardware-managed
└── global memory      large, all threads, highest latency
```

| Memory | Scope | Lifetime | Typical use |
|---|---|---|---|
| register | thread | kernel | temporaries |
| shared | block | kernel | tiles, reductions |
| global | grid | allocation | input/output tensors |
| local | thread | kernel | spills |
| constant | grid | application | read-only constants |

On Jetson Orin, CPU and GPU share the same physical DRAM. `cudaMallocManaged` gives one allocation
that both can address; it is still important to synchronize before the CPU reads a GPU result.

## Build with CMake

```cmake
cmake_minimum_required(VERSION 3.25)
project(vector_add LANGUAGES CXX CUDA)

set(CMAKE_CUDA_ARCHITECTURES 87)
add_executable(vector_add vector_add.cu)
```

Configure and build:

```sh
cmake -S . -B build -G Ninja
cmake --build build
```

`enable_language(CUDA)` is implicit in `project(... LANGUAGES CUDA)`.

## Streams and errors

A kernel launch is asynchronous. `cudaGetLastError()` catches a bad launch configuration; it does
not wait for the kernel. To read the result on the CPU, synchronize:

```cpp
cudaStream_t stream = nullptr;
CUDA_CHECK(cudaStreamCreate(&stream));

vectorAdd<<<blocks, threads, 0, stream>>>(a, b, c, n);
CUDA_CHECK(cudaGetLastError());
CUDA_CHECK(cudaStreamSynchronize(stream));

CUDA_CHECK(cudaStreamDestroy(stream));
```

In production, synchronize as little as possible. Use one stream for dependent work and events to
coordinate stages. A `cudaDeviceSynchronize()` after every kernel is a debugging tool, not a
performance plan.

## Performance rules

- One thread per output element is the default; it is often enough.
- Adjacent threads should touch adjacent memory (coalescing).
- Use shared memory when several threads reuse the same values.
- Keep branches within a warp uniform when possible.
- Prefer `const` and `__restrict__` for input/output pointers so the compiler can optimize.
- Measure with Nsight Systems and Nsight Compute; do not guess.
- Check occupancy only after the kernel is correct; a simple kernel with good coalescing often wins.
- For image work, process a tile per block and respect the row pitch — `width` is not `pitch`.

## What `.cu` is not

- It is not where TensorRT builds or runs an engine. TensorRT has a host C++ API.
- It is not automatically a zero-copy path. A kernel reading a CPU pointer is still a copy or a
  mapping problem.
- It is not a separate application. A `.cu` file is usually one translation unit in a larger binary.

For the memory side of a camera pipeline — `NvBufSurface`, EGL/CUDA interop and dma-buf — see
[NVMM and NvBufSurface]({{ '/jetson-orin/nvmm/' | relative_url }}).

## NVIDIA references

- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-programming-guide/)
- [CUDA C++ Programming Guide — compute capabilities](https://docs.nvidia.com/cuda/cuda-programming-guide/05-appendices/compute-capabilities.html)
- [Even Easier Introduction to CUDA](https://developer.nvidia.com/blog/even-easier-introduction-cuda/)
- [CUDA graphics interoperability](https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/graphics-interop.html)

<div class="jo-note">
  The `.cu` extension is not cosmetic. It selects the CUDA compiler and makes the kernel launch
  syntax legal. Everything else — grid size, coalescing, streams, synchronization — is the actual
  GPU programming.
</div>
