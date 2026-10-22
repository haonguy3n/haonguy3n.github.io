---
title: "C++ patterns: data and flow"
layout: page
permalink: /cpp/data/
toc: true
tags: [cpp, patterns]
description: >-
  Value types, stores and caches, rule objects, pipelines, do-nothing implementations and one place
  that builds the objects: the patterns that keep data and decisions in one place.
---

{% include jetson-orin-style.html %}

<p class="jo-back"><a href="{{ '/cpp/' | relative_url }}">← C++ patterns</a></p>

> Data and flow patterns are about **what is a value, what is a policy, and where the data goes
> next**. They remove the "same condition in five files" problem.

## Value type

*Also called: Value Object. Two values with the same fields are equal and interchangeable.*

**Problem:** a key or a configuration is passed as a bag of strings and integers; equality, hashing
and validation are repeated (and sometimes disagree).

**Shape:** an immutable type with a constructor that validates and value equality.

```cpp
struct PlanKey {
    std::string model;
    int batch = 1;
    Precision precision = Precision::Fp16;

    auto operator<=>(const PlanKey&) const = default;
};

struct PlanKeyHash {
    std::size_t operator()(const PlanKey& key) const {
        return std::hash<std::string>{}(key.model)
             ^ (std::hash<int>{}(key.batch) << 1)
             ^ (std::hash<int>{}(static_cast<int>(key.precision)) << 2);
    }
};
```

**Use it when** the same tuple appears in several interfaces or as a cache key. **Avoid** a value
object for a temporary pair used once.

**Trade-off:** a value object is copied; keep it small or pass it by `const&`. Equality must include
every field that changes the result.

## Store / cache

*Also called: Repository / Cache. The store hides the file layout; the cache hides the lookup policy.*

**Problem:** the code needs an object from disk or a plan from a cache, but the callers should not
know the file layout, hashing or eviction rules.

**Shape:** one interface with `Find` and `Store`; the implementation owns the policy.

```cpp
class PlanCache {
public:
    std::expected<Plan, std::string> Find(const PlanKey& key) const {
        auto it = m_entries.find(key);
        if (it == m_entries.end()) return std::unexpected("not cached");
        return it->second;
    }

    void Store(const PlanKey& key, Plan plan) {
        m_entries.insert_or_assign(key, std::move(plan));
    }
private:
    std::unordered_map<PlanKey, Plan, PlanKeyHash> m_entries;
};
```

**Use it when** the storage policy is likely to change or the cache is a test seam. **Avoid** a
repository over a single `ifstream` used in one function.

**Trade-off:** a cache must define staleness, ownership and concurrency. A cache that never evicts
is a leak with a nicer name.

## Rule object

*Also called: Specification. It is a named predicate that can be tested and logged.*

**Problem:** a decision ("is this model used?") is a set of rules that must be testable, composable
and explainable in a log.

**Shape:** a named predicate object; composition with `and`, `or`, `not`.

```cpp
class IRule {
public:
    virtual ~IRule() = default;
    virtual bool Matches(const Profile& profile) const = 0;
    virtual std::string Describe() const = 0;
};

class KeyIsSet final : public IRule {
public:
    KeyIsSet(std::string section, std::string key)
        : m_section(std::move(section)), m_key(std::move(key)) {}
    bool Matches(const Profile& profile) const override {
        return profile.Int(m_section, m_key) != 0;
    }
    std::string Describe() const override { return m_section + "/" + m_key + " != 0"; }
private:
    std::string m_section;
    std::string m_key;
};
```

**Use it when** the same rule is used to decide and to log why. **Avoid** a specification hierarchy
for one `if`; a function is enough.

**Trade-off:** a predicate object per rule can be many small types; a table of rules with a name is
often simpler. The important part is one source of truth, not the class count.

## Pipeline

*A pipeline is a chain of filters; each filter takes a value and returns the next value or an error.*

**Problem:** a frame passes through a fixed sequence of steps (decode, resize, normalize, infer),
and each step can fail or drop the frame.

**Shape:** a small `Filter` interface and a graph that owns the filters and the hand-over.

```cpp
class Filter {
public:
    virtual ~Filter() = default;
    virtual std::expected<Frame, std::string> Process(Frame frame) = 0;
};

class Pipeline {
public:
    void Add(std::unique_ptr<Filter> filter) { m_filters.push_back(std::move(filter)); }

    std::expected<Frame, std::string> Run(Frame frame) {
        for (auto& filter : m_filters) {
            auto next = filter->Process(std::move(frame));
            if (!next) return next;
            frame = std::move(*next);
        }
        return frame;
    }
private:
    std::vector<std::unique_ptr<Filter>> m_filters;
};
```

**Use it when** the steps are ordered and each step is independently testable. **Avoid** a pipeline
for two steps that will never be reordered; a function call is clearer.

**Trade-off:** moving frames through a vector of filters can add copies if the type is not
move-aware. Use a frame view/lease (see [Idioms]({{ '/cpp/idioms/' | relative_url }})) and define
the failure policy: stop, skip or drop.

## Do-nothing implementation

*Also called: Null Object. It is useful when absence is normal; it must not hide a configuration error.*

**Problem:** "no logger", "no viewer" or "this manager was skipped" is a normal state, and every
caller should not need a null check.

**Shape:** an implementation that does nothing, or a proxy that returns the neutral answer.

```cpp
class NullLog final : public ILogSink {
public:
    void Log(LogLevel, std::string_view) override {}
};
```

**Use it when** absence is normal and a no-op is semantically correct. **Avoid** a null object that
hides a configuration mistake which should be an error.

**Trade-off:** a no-op can make a missing dependency silent. Log once at construction, not per call.

## One place that builds the objects

*Also called: dependency injection, with a composition root. The graph is built once at the edge.*

**Problem:** a class creates its own dependencies, so it cannot be tested and its lifetime is
hidden.

**Shape:** dependencies are constructor parameters; the process's `main` constructs the graph once.

```cpp
class App {
public:
    App(ISensor& sensor, IEngine& engine, ILogSink& log)
        : m_sensor(sensor), m_engine(engine), m_log(log) {}

    std::expected<void, std::string> Run() {
        auto frame = m_sensor.Read();
        if (!frame) return std::unexpected("capture failed");
        m_log.Log(LogLevel::Info, "frame");
        return m_engine.Run(*frame);
    }
private:
    ISensor& m_sensor;
    IEngine& m_engine;
    ILogSink& m_log;
};

int main() {
    CameraSensor sensor;
    TensorRtEngine engine;
    JournalLog log;
    App app(sensor, engine, log);
    return app.Run() ? 0 : 1;
}
```

**Use it when** an object has more than one dependency or a test needs a fake. **Avoid** a
service-locator/global registry; it hides the graph and makes teardown order unclear.

**Trade-off:** the composition root knows every concrete type. That is its job; keep it at the
edge and keep the core dependent only on the interfaces.

<div class="jo-note">
  A value object removes a repeated tuple. A repository removes a repeated file layout. A
  specification removes a repeated rule. A pipeline removes a repeated sequence. If two of these
  are in the same class, split the class.
</div>
