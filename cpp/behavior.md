---
title: "C++ patterns: behavior"
layout: page
permalink: /cpp/behavior/
toc: true
tags: [cpp, patterns]
description: >-
  Strategy, state, observer, template method, producer-consumer and active object, with the
  concurrency and lifetime rules that make them safe in an always-on camera service.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/cpp/' | relative_url }}">← C++ patterns</a></p>

> Behavior patterns are about **who decides what happens next**. In a camera service the difficult
> part is rarely the algorithm; it is the stop condition, the timeout and the ownership of the
> thread that is still running.

## Strategy

**Problem:** two algorithms share the same input and output, and the choice is a configuration or a
measurement. Inlining both with `if` makes each branch harder to test.

**Shape:** a small interface (or a function) per algorithm; the owner selects once.

```cpp
class IPlanner {
public:
    virtual ~IPlanner() = default;
    virtual std::optional<int> NextTarget() = 0;
    virtual void Record(int position, double score) = 0;
};

class LegacyPlanner final : public IPlanner { /* ... */ };
class CoarseFinePlanner final : public IPlanner { /* ... */ };

std::unique_ptr<IPlanner> MakePlanner(Mode mode) {
    if (mode == Mode::CoarseFine)
        return std::make_unique<CoarseFinePlanner>();
    return std::make_unique<LegacyPlanner>();
}
```

**Use it when** the algorithms are independently testable and the selection is made once.
**Avoid** a Strategy where the "algorithm" is two lines and a lambda would be clearer.

**Trade-off:** a virtual call per step is irrelevant; the cost is the extra type and the test matrix.
Keep the interface minimal: `NextTarget` and `Record` are enough here.

## State

**Problem:** a search or a build has a small set of phases, and only some operations are legal in
each phase. A pile of booleans makes illegal states representable.

**Shape:** one enum and one `Step()` that dispatches; transitions live in one place.

```cpp
enum class Phase { Idle, Moving, Collecting, Done };

class FocusSearch {
public:
    bool Step() {
        switch (m_phase) {
        case Phase::Moving:     OnMoving(); break;
        case Phase::Collecting: OnCollecting(); break;
        case Phase::Done:       return false;
        case Phase::Idle:       return false;
        }
        return m_phase != Phase::Done;
    }
private:
    void OnMoving();
    void OnCollecting();
    Phase m_phase = Phase::Idle;
};
```

**Use it when** the phase count is small and the transitions are the logic. **Avoid** a full
state-machine framework for three states; a table or a switch is easier to review.

**Trade-off:** a switch can grow; keep each phase handler short and make the transition visible.
For larger machines, a transition table with `(state, event) -> state` is easier to test.

## Observer

**Problem:** several sinks (journal, file, test) need the same event, and the producer should not
know any of them.

**Shape:** a narrow sink interface; registration is explicit; callbacks never run while a lock is
held.

```cpp
enum class LogLevel { Debug, Info, Warning, Error };

class ILogSink {
public:
    virtual ~ILogSink() = default;
    virtual void Log(LogLevel level, std::string_view line) = 0;
};

class Logger {
public:
    explicit Logger(ILogSink& sink) : m_sink(sink) {}
    void Info(std::string_view line) { m_sink.Log(LogLevel::Info, line); }
private:
    ILogSink& m_sink;
};
```

**Use it when** the set of consumers is open or a test needs to capture output. **Avoid** an
observer for one known consumer; pass the dependency directly.

**Trade-off:** lifetime is the hard part. A sink that outlives its owner is a use-after-free; a
callback that calls back into the producer can deadlock. Document both.

## Template Method

**Problem:** several sessions share the same skeleton: initialize, process one item, finish. Only
the steps differ.

**Shape:** a non-virtual public `Run()` calls protected virtual steps; the skeleton cannot be
accidentally skipped.

```cpp
class Session {
public:
    virtual ~Session() = default;

    void Run() {
        if (!Init()) return;
        while (ProcessOne()) {}
        Finish();
    }
protected:
    virtual bool Init() = 0;
    virtual bool ProcessOne() = 0;
    virtual void Finish() = 0;
};

class ClassifierSession final : public Session {
protected:
    bool Init() override { /* load engine */ return true; }
    bool ProcessOne() override { /* pop one item */ return true; }
    void Finish() override { /* release */ }
};
```

**Use it when** the order is fixed and the variation is a handful of steps. **Avoid** it when the
steps need to be composed dynamically; use Strategy or a pipeline instead.

**Trade-off:** inheritance couples the steps to the base. Keep the base small and do not call
virtuals from the destructor.

## Producer–Consumer

**Problem:** a capture thread must not block on a slow consumer, and a consumer must stop promptly
when the service stops.

**Shape:** a bounded queue with one mutex, one condition variable and an explicit stop flag.

```cpp
template <class T>
class BoundedQueue {
public:
    explicit BoundedQueue(std::size_t capacity) : m_capacity(capacity) {}

    bool Push(T value) {
        std::lock_guard lock(m_mutex);
        if (m_stopped || m_items.size() >= m_capacity) return false;
        m_items.push_back(std::move(value));
        m_ready.notify_one();
        return true;
    }

    std::optional<T> Pop() {
        std::unique_lock lock(m_mutex);
        m_ready.wait(lock, [this] { return m_stopped || !m_items.empty(); });
        if (m_items.empty()) return std::nullopt;
        T value = std::move(m_items.front());
        m_items.pop_front();
        return value;
    }

    void Stop() {
        std::lock_guard lock(m_mutex);
        m_stopped = true;
        m_ready.notify_all();
    }
private:
    std::mutex m_mutex;
    std::condition_variable m_ready;
    std::deque<T> m_items;
    std::size_t m_capacity;
    bool m_stopped = false;
};
```

**Use it when** the producer and consumer have different rates and a drop/backpressure policy is
acceptable. **Avoid** an unbounded queue: it turns a slow consumer into an out-of-memory failure.

**Trade-off:** a mutex and a condition variable are simple and predictable; a lock-free ring is
faster only after measurement. Always define what happens when the queue is full.

## Active Object / Command

**Problem:** a device call or a model run must happen on one worker thread, but callers arrive from
several threads and must not block on the device.

**Shape:** a queue of commands owned by a worker; `Submit` enqueues, `Stop` drains and joins.

```cpp
struct Command {
    virtual ~Command() = default;
    virtual void Execute() = 0;
};

class Worker {
public:
    void Submit(std::unique_ptr<Command> command) {
        std::lock_guard lock(m_mutex);
        if (m_stopping) return;
        m_queue.push_back(std::move(command));
        m_ready.notify_one();
    }

    void Stop() {
        {
            std::lock_guard lock(m_mutex);
            m_stopping = true;
            m_ready.notify_all();
        }
        if (m_thread.joinable()) m_thread.join();
    }
private:
    void Run() {
        for (;;) {
            std::unique_ptr<Command> command;
            {
                std::unique_lock lock(m_mutex);
                m_ready.wait(lock, [this] { return m_stopping || !m_queue.empty(); });
                if (m_queue.empty()) return;
                command = std::move(m_queue.front());
                m_queue.pop_front();
            }
            command->Execute();
        }
    }
    std::mutex m_mutex;
    std::condition_variable m_ready;
    std::deque<std::unique_ptr<Command>> m_queue;
    bool m_stopping = false;
    std::thread m_thread {[this] { Run(); }};
};
```

**Use it when** the work must be serialized on one thread and the caller must not wait.
**Avoid** an active object for a call that is already fast and thread-safe; the queue only adds
latency.

**Trade-off:** every command needs a lifetime decision (who owns the result, what happens on stop).
The destructor must stop and join before members are destroyed; the member order matters.

<div class="jo-note">
  The concurrency patterns above all end with the same question: who owns the thread, and what does
  stop mean? If that answer is not in the type, the pattern is not finished.
</div>
