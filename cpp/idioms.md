---
title: "C++ idioms"
layout: page
permalink: /cpp/idioms/
toc: true
tags: [cpp, patterns]
description: >-
  The C++ rules behind the patterns: RAII, Rule of Zero/Five, std::expected, value semantics, views,
  type erasure and zero-copy frame views.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/cpp/' | relative_url }}">← C++ patterns</a></p>

> Patterns are the shapes; idioms are the details that make them safe. These are the C++ rules I
> apply before adding a pattern.

## Rule of Zero / Five

**Rule of Zero:** if a class does not manage a resource directly, do not write a destructor, copy or
move operation. Let the members do it.

```cpp
struct Config {
    std::string profile;
    int batch = 1;
    std::vector<int> shapes;
    // no destructor, no copy, no move: the compiler does the right thing
};
```

**Rule of Five:** if a class owns a resource (raw pointer, fd, device handle), define the destructor,
copy constructor, copy assignment, move constructor and move assignment — or delete the copies and
default the moves.

```cpp
class Device {
public:
    explicit Device(Handle handle) : m_handle(handle) {}
    ~Device() { if (m_handle) Close(m_handle); }
    Device(const Device&) = delete;
    Device& operator=(const Device&) = delete;
    Device(Device&& other) noexcept : m_handle(std::exchange(other.m_handle, {})) {}
    Device& operator=(Device&& other) noexcept {
        if (this != &other) {
            if (m_handle) Close(m_handle);
            m_handle = std::exchange(other.m_handle, {});
        }
        return *this;
    }
private:
    Handle m_handle = {};
};
```

**Why:** the compiler's copy is a shallow copy of the handle. Two destructors then close it twice.
The explicit rule is a correctness rule, not style.

## RAII

**Rule:** every resource is owned by an object; the destructor releases it; the resource is acquired
in the constructor (or a named factory) and never leaked on an error path.

```cpp
void Run() {
    std::lock_guard lock(m_mutex);          // unlocks on return or throw
    auto frame = m_pool.Acquire();          // returns the lease on destruction
    auto plan = m_cache.Find(key);          // value, no manual free
    // ...
}
```

**Why:** early returns are normal in error handling. Manual `unlock`/`free` on every path is a bug
waiting for the next branch.

**Pitfall:** a destructor that runs while a worker thread still uses the object. Stop and join
before the members are destroyed, or hold the resource in a longer-lived owner.

## `std::expected` for errors

**Rule:** a function that can fail returns a value or a reason; exceptions stay at the process
boundary and never cross a C ABI, a thread entry or a device callback.

```cpp
std::expected<Frame, std::string> ReadFrame() {
    if (!m_device) return std::unexpected("device is not open");
    Frame frame;
    if (!m_device->Read(frame)) return std::unexpected("read failed");
    return frame;
}

void Use() {
    auto frame = ReadFrame();
    if (!frame) {
        Log(LogLevel::Error, frame.error());
        return;
    }
    Process(*frame);
}
```

**Why:** the caller cannot ignore the failure silently (unlike an error code), and the function has
no hidden control flow. In a real-time service, a predictable error path is easier to test than an
exception unwinding through a callback.

**Pitfall:** `std::expected` is C++23. A C++11 public header needs a pimpl or a status type; do not
leak `std::expected` into a C++11 consumer.

## Value semantics and `const`

**Rule:** pass small values by value, larger values by `const&`, and mark methods `const` when they
do not change the object. Return values, not output parameters, when the value is the result.

```cpp
PlanKey MakeKey(const Profile& profile, int batch) {
    return PlanKey{profile.Name(), batch, profile.Precision()};
}

std::size_t Count(const std::vector<Frame>& frames) {
    return frames.size();
}
```

**Why:** value semantics make ownership obvious; `const` makes the thread-safety review smaller.
An output parameter usually means the function has two results and no one knows which one failed.

**Pitfall:** a `const` method that mutates through a pointer member is still a mutation. Use
`mutable` only for a cache with a documented lock.

## Views: `std::span` and `std::string_view`

**Rule:** a view is a non-owning reference with a lifetime shorter than the owner. Use it for
read-only parameters; never store it beyond the call unless the owner is guaranteed.

```cpp
void Parse(std::string_view line, std::span<const std::byte> payload);
```

**Why:** avoids copying a buffer or a string at every call.

**Pitfall:** the owner can die while the view lives. A frame view with a pointer and a pitch is the
same contract: the lease must outlive every consumer.

## Type erasure and `std::variant`

**Rule:** use `std::variant` for a **closed** set of types; use a small interface (or type erasure)
for an **open** set. Do not use `std::any` for either.

```cpp
using Result = std::variant<Classification, Detection, std::string>;

std::string Describe(const Result& result) {
    return std::visit([](const auto& value) { return ToString(value); }, result);
}
```

**Why:** `std::variant` makes the closed set explicit and forces every visitor to handle every
alternative. An interface is better when a new implementation is expected without editing the
caller.

## Zero-copy frame view

**Rule:** a frame view is a pointer, a size, a pitch and a lifetime. The producer owns the buffer;
the consumer holds a lease; the GPU path uses the same memory.

```cpp
struct FrameView {
    const std::byte* data = nullptr;
    int width = 0;
    int height = 0;
    int pitch = 0;
    PixelFormat format = PixelFormat::Gray8;
};

class FrameLease {
public:
    FrameView View() const { return m_view; }
    // release returns the buffer to the pool
private:
    FrameView m_view;
};
```

**Why:** a `cv::Mat` or a `std::vector` copy is easy to write and expensive on every frame. The view
plus the lease makes the copy budget explicit.

**Pitfall:** pitch is not width; a plane may be padded. The buffer must stay alive and synchronised
until the GPU work finishes.

## No exceptions across boundaries

**Rule:** catch at the boundary where a C API, a thread entry or a callback returns; convert to a
status. Do not let an exception unwind through a `noexcept` destructor or a C function.

```cpp
extern "C" int ApiCall() noexcept {
    try {
        return Run();
    } catch (const std::exception&) {
        return -1;
    }
}
```

**Why:** the caller cannot see the exception; the ABI may not support unwinding; a destructor that
throws terminates the process.

## What not to abstract

- A single implementation behind an interface with no test or second backend: use the concrete type.
- A two-line algorithm behind a Strategy: use a function.
- A value object with two fields used once: use a struct.
- A factory that only calls a constructor: call the constructor.
- A registry for a fixed set: use a `switch`.

The patterns are there to remove a real duplication or make a seam testable. If neither is true, the
simpler code wins.

<div class="jo-note">
  These idioms are the base layer under every page: ownership, lifetime, errors and views. If those
  are right, the pattern is usually small; if they are wrong, the pattern only hides the bug.
</div>

**Further reading:** [ArchMan](https://archman.dev/) for the general architecture catalogue, and
[Coding rules and PR checklist]({{ '/engineering/' | relative_url }}) for the review side.
