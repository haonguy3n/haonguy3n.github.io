---
title: C++ Patterns
layout: page
icon: fas fa-cubes
order: 8
toc: true
tags: [cpp, patterns]
description: >-
  The C++ patterns used in this project, with small examples: interfaces and adapters, strategy, state,
  builder, proxy, RAII handles, pools, caches, rule objects, pipelines and the idioms around them.
---

{% include jetson-orin-style.html %}

> Patterns are a shared vocabulary, not a checklist. The heading is the common name; the formal
> pattern name is in parentheses. Each note says what problem it solves, where it fits in this
> codebase, a small C++23 example, and when the pattern is the wrong answer. The examples are simplified and generic; no private code, paths or customer details.

<div class="jo-flow">
  <div class="jo-step"><strong>Rule</strong><span>small change, one owner, test the failure</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Pattern</strong><span>name the shape that removes the duplication</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Example</strong><span>C++23, small enough to read</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Test</strong><span>the edge case is the proof</span></div>
</div>

## The map

| Group | Patterns | Page |
|---|---|---|
| Structure | Interfaces and adapters (Ports & Adapters), Adapter, Proxy, Facade, Pimpl | [Structure]({{ '/cpp/structure/' | relative_url }}) |
| Behavior | Strategy, State, Observer, Template Method, Bounded queue between threads, Worker thread with a command queue | [Behavior]({{ '/cpp/behavior/' | relative_url }}) |
| Creation | Builder, Factory, Registry, Object Pool, RAII handle | [Creation]({{ '/cpp/creation/' | relative_url }}) |
| Data and flow | Value type, Store / cache, Rule object, Pipeline, Do-nothing implementation, One place that builds the objects | [Data and flow]({{ '/cpp/data/' | relative_url }}) |
| C++ idioms | Rule of Zero/Five, `std::expected`, value semantics, views, type erasure, zero-copy | [Idioms]({{ '/cpp/idioms/' | relative_url }}) |

## How to use this

- **Start with the problem, not the pattern.** If a virtual interface has one implementation and no
  test seam, you probably want a function, not a Strategy.
- **Prefer composition at the boundary.** Ports at the edge of the app, concrete classes inside.
- **A pattern is justified by a second user or a test.** A pattern with one caller is usually
  indirection.
- **Keep the ownership visible.** The pattern should make who frees what clearer, not hide it.

The process rules that go with these patterns are in [Coding rules and PR checklist]({{ '/engineering/' | relative_url }}).
For a broader catalogue of architecture and design patterns, see [ArchMan](https://archman.dev/);
the examples here are adapted to this codebase and its constraints.

<div class="jo-note">
  The common thread is not "use more patterns". It is: put the volatile part behind a seam, keep
  the ownership explicit, and make the failure path testable.
</div>
