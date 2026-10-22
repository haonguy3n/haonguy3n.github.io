---
title: "C++ patterns: creation"
layout: page
permalink: /cpp/creation/
toc: true
tags: [cpp, patterns]
description: >-
  Builder, factory, registry, object pool and RAII handles, with the ownership rules that keep
  construction, caching and device memory predictable.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/cpp/' | relative_url }}">← C++ patterns</a></p>

> Creation patterns answer **who makes the object, who owns it, and when is it released**. In an
> embedded service the last question matters more than the first.

## Builder

**Problem:** a plan or a session needs several settings (workspace, precision, batch, cache path)
and invalid combinations must be rejected before the expensive operation starts.

**Shape:** a builder with named steps and one terminal `Build()` that validates and returns a value.

```cpp
class PlanBuilder {
public:
    PlanBuilder& Workspace(std::size_t bytes) { m_workspace = bytes; return *this; }
    PlanBuilder& Precision(Precision precision) { m_precision = precision; return *this; }
    PlanBuilder& Batch(int batch) { m_batch = batch; return *this; }

    std::expected<Plan, std::string> Build() const {
        if (m_batch < 1) return std::unexpected("batch must be positive");
        if (m_workspace < kMinWorkspace) return std::unexpected("workspace too small");
        return Plan::Compile(m_workspace, m_precision, m_batch);
    }
private:
    std::size_t m_workspace = kDefaultWorkspace;
    Precision m_precision = Precision::Fp16;
    int m_batch = 1;
};
```

**Use it when** construction has more than three inputs, defaults matter, or the order of checks is
part of the contract. **Avoid** a builder for a two-field value; use aggregate initialization.

**Trade-off:** the builder is mutable until `Build`; do not share it between threads. Keep the
validation in one place, not in every setter.

## Factory

**Problem:** the concrete type depends on a runtime value (platform, model kind, profile), and the
caller should not include every implementation header.

**Shape:** a function or a small factory that returns the interface, with one switch/table.

```cpp
enum class Kind { Host, Jetson };

std::unique_ptr<IEngine> MakeEngine(Kind kind, const Config& config) {
    switch (kind) {
    case Kind::Jetson: return std::make_unique<TensorRtEngine>(config);
    case Kind::Host:   return std::make_unique<OnnxEngine>(config);
    }
    return nullptr;
}
```

**Use it when** the selection is a small closed set and the construction is non-trivial.
**Avoid** a factory that only calls a constructor; the caller can do that.

**Trade-off:** a factory hides the concrete type from tests; expose a test factory or a fake at the
interface instead of mocking the factory.

## Registry

**Problem:** the set of implementations is open (new machines, new backends) and a string or enum
from configuration selects one.

**Shape:** a map from a stable name to a maker function, filled by each implementation at start-up.

```cpp
using Maker = std::function<std::unique_ptr<IEngine>(const Config&)>;

class EngineRegistry {
public:
    bool Add(std::string name, Maker maker) {
        return m_makers.emplace(std::move(name), std::move(maker)).second;
    }
    std::unique_ptr<IEngine> Create(const std::string& name, const Config& config) const {
        auto it = m_makers.find(name);
        return it == m_makers.end() ? nullptr : it->second(config);
    }
private:
    std::map<std::string, Maker> m_makers;
};
```

**Use it when** new implementations must be added without editing the caller. **Avoid** a registry
when the set is fixed and a `switch` is clearer.

**Trade-off:** a registry is global state if it is a singleton. Fill it in the composition root and
pass it in; keep registration before any worker starts.

## Object Pool

**Problem:** creating a device context or a frame buffer is expensive, and the number of live
objects must be bounded.

**Shape:** a fixed number of objects, an acquire/release pair, and a lease that returns the object
even on early exit.

```cpp
class ContextPool {
public:
    explicit ContextPool(std::size_t size) {
        for (std::size_t i = 0; i < size; ++i)
            m_free.push_back(std::make_unique<Context>());
    }

    std::unique_ptr<Context, std::function<void(Context*)>> Acquire() {
        std::lock_guard lock(m_mutex);
        if (m_free.empty()) return {nullptr, [](Context*) {}};
        Context* context = m_free.back().release();
        m_free.pop_back();
        return {context, [this](Context* p) {
            std::lock_guard lock(m_mutex);
            m_free.emplace_back(p);
        }};
    }
private:
    std::mutex m_mutex;
    std::vector<std::unique_ptr<Context>> m_free;
};
```

**Use it when** construction is expensive and a bounded pool is part of the memory budget.
**Avoid** pooling small values; allocation is not the bottleneck.

**Trade-off:** a lease must be released exactly once; the deleter captures `this`, so the pool must
outlive every lease. Document the shutdown order.

## RAII handle

*A lease is the common name for a borrowed resource; the handle returns it on destruction.*

**Problem:** a resource (engine context, frame, scratch buffer, lock) must be released on every
return path, including an error or a timeout.

**Shape:** the resource is owned by an object whose destructor releases it; a lease is the only way
to use it.

```cpp
class FrameLease {
public:
    FrameLease(Frame* frame, std::function<void(Frame*)> release)
        : m_frame(frame, std::move(release)) {}

    Frame* operator->() const { return m_frame.get(); }
    explicit operator bool() const { return m_frame != nullptr; }
private:
    std::unique_ptr<Frame, std::function<void(Frame*)>> m_frame;
};
```

**Use it when** a resource has a non-trivial release or must outlive a scope. **Avoid** manual
`lock`/`unlock` pairs when a `std::lock_guard` or `std::unique_lock` expresses the scope.

**Trade-off:** a custom deleter costs an allocation and an indirect call. For a hot path, a small
RAII wrapper with a concrete type is cheaper than `std::function`.

<div class="jo-note">
  The creation patterns are not about hiding `new`. They are about making the lifetime one visible
  decision: who creates, who owns, and what happens on the error path.
</div>
