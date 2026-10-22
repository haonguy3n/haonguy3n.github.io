---
title: "C++ patterns: structure"
layout: page
permalink: /cpp/structure/
toc: true
tags: [cpp, patterns]
description: >-
  Interfaces and adapters, adapter, proxy, facade and pimpl, with small C++23 examples and the
  trade-offs that decide when the extra indirection earns its place.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/cpp/' | relative_url }}">← C++ patterns</a></p>

> Structure patterns are about **where the seams are**. In this codebase they sit at the edge of the
> application: a port for a device, an adapter for a legacy class, a facade for a start-up sequence,
> and a pimpl where a public header must stay C++11-clean.

## Interfaces and adapters

*Also called: Ports & Adapters, or Hexagonal architecture.*

**Problem:** the core logic needs a frame, a lens or an engine, but it must be testable on a PC and
must not know whether the backend is TensorRT, a simulator or a fake.

**Shape:** a small interface owned by the core, implementations at the edge, and the concrete object
injected by the composition root.

```cpp
class ISensor {
public:
    virtual ~ISensor() = default;
    virtual std::expected<Frame, std::string> Read() = 0;
};

class CameraSensor final : public ISensor {
public:
    std::expected<Frame, std::string> Read() override { /* CSI/ISP */ }
};

class FakeSensor final : public ISensor {
public:
    std::expected<Frame, std::string> Read() override { return m_next; }
private:
    Frame m_next;
};

void Capture(ISensor& sensor) {
    auto frame = sensor.Read();
    if (!frame) return; // the port says why
    // ...
}
```

**Use it when** the dependency has a real second implementation (test, host, another SoC) or crosses
a hardware boundary. **Avoid** a port around a stable standard-library type or a pure data struct.

**Trade-off:** one virtual call per frame is cheap; one interface per class is not. Keep the
interface at the boundary, not between every internal helper.

## Adapter

**Problem:** an old class has the right behaviour and the wrong signature. You do not want to edit
the old class, and you do not want its header to leak into new code.

**Shape:** a small class that implements the new interface and translates both directions.

```cpp
class LegacyLens {                 // old API, must keep compiling
public:
    bool MoveTo(int position);
    int Current() const;
};

class LensAdapter final : public ILens {
public:
    explicit LensAdapter(LegacyLens& lens) : m_lens(lens) {}

    std::expected<void, std::string> Move(int position) override {
        return m_lens.MoveTo(position) ? std::expected<void, std::string>{}
                                       : std::unexpected("lens refused the move");
    }
    int Position() const override { return m_lens.Current(); }

private:
    LegacyLens& m_lens;
};
```

**Use it when** the legacy class is a real dependency and the new interface is the one the rest of
the code should see. **Avoid** wrapping every legacy call; an adapter earns its place at the seam.

**Trade-off:** translation can hide a semantic difference. Write the difference down (units, error
codes, thread-safety) and test both sides.

## Proxy

**Problem:** a call site holds a pointer that can legitimately be null: the manager was not created
for this profile, or the optional backend is absent. A null check at every call site is repeated and
easy to miss.

**Shape:** a proxy with the same call shape as the real object; calls return a safe "not available"
answer when nothing is held.

```cpp
class ManagerSwitch {
public:
    bool Process(const Input& in, Output& out) const {
        return m_p != nullptr && m_p->Process(in, out);
    }
    explicit operator bool() const { return m_p != nullptr; }
private:
    Manager* m_p = nullptr;
};
```

**Use it when** absence is normal and the caller can handle a false/no-op answer. **Avoid** using a
proxy to hide a programming error that should crash in debug.

**Trade-off:** a false return must be a documented contract; otherwise the proxy turns a bug into a
silent missing result.

## Facade

**Problem:** starting the system needs a fixed order: read the profile, build missing plans, load
engines, start threads, register the service. A caller that repeats that order will eventually get
it wrong.

**Shape:** one small object exposes "start", "run", "stop" and keeps the sequence and ownership
inside.

```cpp
class Runtime {
public:
    std::expected<void, std::string> Start(const Profile& profile) {
        auto plans = m_cache.Prepare(profile);
        if (!plans) return plans;
        m_engine = std::make_unique<Engine>(profile, *plans);
        return m_engine->Start();
    }
    void Stop() { if (m_engine) m_engine->Stop(); }
private:
    PlanCache m_cache;
    std::unique_ptr<Engine> m_engine;
};
```

**Use it when** a sequence has ordering constraints and one owner can enforce them. **Avoid** a
facade that becomes a god object; if it has more than a handful of methods, split the phases.

**Trade-off:** the facade hides flexibility. Keep the lower-level pieces injectable for tests.

## Pimpl: hide the implementation

*The common C++ name is the Pimpl idiom (pointer to implementation).*

**Problem:** a public header is consumed by a C++11 target, but the implementation uses C++23
(`std::expected`, `std::span`, `std::format`) and should not leak those headers to the consumer.

**Shape:** the public class holds a pointer to an incomplete implementation; the destructor and the
copy/move operations are defined in the `.cpp`.

```cpp
// Planner.h — C++11-clean
class Planner final {
public:
    Planner();
    ~Planner();
    Planner(Planner&&) noexcept;
    Planner& operator=(Planner&&) noexcept;
    Planner(const Planner&) = delete;
    Planner& operator=(const Planner&) = delete;

    int Next();
private:
    struct Impl;
    std::unique_ptr<Impl> m_p;
};
```

```cpp
// Planner.cpp — C++23
struct Planner::Impl {
    std::deque<int> pending;
    std::expected<int, std::string> next() { /* ... */ }
};

Planner::Planner() : m_p(std::make_unique<Impl>()) {}
Planner::~Planner() = default;
Planner::Planner(Planner&&) noexcept = default;
Planner& Planner::operator=(Planner&&) noexcept = default;
int Planner::Next() { return m_p->next().value_or(-1); }
```

**Use it when** the ABI/header boundary matters or the implementation is deliberately hidden.
**Avoid** pimpl for a small value type; it costs an allocation and an indirection.

**Trade-off:** the compiler cannot inline through it, and copy semantics need a decision. Define the
rule of five explicitly.

<div class="jo-note">
  The structure patterns here all do one thing: they move a dependency to a seam where it can be
  replaced or tested. If a seam has only one implementation forever, a function or a value type is
  usually the simpler answer.
</div>

**Further reading:** [ArchMan — software architecture and design patterns](https://archman.dev/).
