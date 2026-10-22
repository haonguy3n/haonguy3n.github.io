---
title: Coding rules and PR checklist
layout: page
icon: fas fa-code
order: 7
permalink: /engineering/
toc: true
tags: [engineering, process]
description: >-
  The short set of coding rules I use on embedded C++ work, and a copy-paste PR checklist
  template: one concern, explicit ownership, tested failures, measured results and a rollback.
---

{% include jetson-orin-style.html %}

> These are the rules I keep coming back to on embedded C++ work. They are deliberately short:
> a rule that cannot be checked in review or by a script is a preference, not a rule. They are
> generic and contain no employer, customer, product or private-code details.

## The rules

1. **One concern per change.** A commit and a PR should answer one question. Do not mix a
   reformat, a rename and a behaviour change; the diff becomes unreviewable and the rollback
   becomes all-or-nothing.
2. **New code in new files.** Legacy code keeps compiling until a phase deliberately touches it.
   If a legacy file must change, make the seam as small as possible and document why it is there.
3. **Ownership is explicit.** Use RAII and one owner per resource. Avoid raw `new`/`delete`,
   hidden global owners and shared ownership without a reason. The destructor order is part of
   the design, especially for threads and device buffers.
4. **Errors are values at the boundary.** Prefer a status/`std::expected`-style result to an
   exception crossing a C ABI, a thread or a device callback. Fail closed, and log enough context
   to identify the object that failed.
5. **One job per function and class.** Keep functions short enough to read on one screen, name
   units in the identifier, and name constants. A magic number in a condition is a bug waiting
   for a second caller.
6. **Threads have one home.** Use one threading primitive/library, never detach a thread that
   owns state, and make stop/join order explicit. Do not call unknown code while holding a lock.
7. **Pin behaviour before rewriting.** Add a test or a golden input for the current behaviour,
   then change the implementation. Pure logic runs on the PC; hardware and timing checks run on
   the device.
8. **The check gate is part of the change.** Format, lint, rule checks and tests must be clean
   before review. If a file is excluded from a check, say so in the PR; an unexplained exclusion
   is a future surprise.
9. **Dependencies are decisions.** Check licence, size, maintenance and update path before adding
   one. Do not vendor a copy or download at build time; put the dependency in the image/build
   system where it can be reproduced.
10. **Measure, do not claim.** “Faster”, “zero copy” and “lower memory” need a before/after
    measurement on the same input and hardware. State the unit, the sample and what was not
    measured.
11. **Security is a requirement.** No secrets in the repository, no credentials in logs, validate
    untrusted input and keep the privilege boundary small. A debug path must not become a
    production bypass.
12. **Docs are part of the change.** Update the flow diagram, the failure modes and the test
    instructions. A future reader should not need the author to understand the change.

## A PR in one picture

<div class="jo-flow">
  <div class="jo-step"><strong>Issue</strong><span>one problem, one owner</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Branch</strong><span>small, focused commits</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>PC checks</strong><span>format, lint, tests</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>PR</strong><span>what, why, verification</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Review</strong><span>design and failure modes</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Device</strong><span>the check that only hardware can do</span></div>
  <div class="jo-arrow">→</div>
  <div class="jo-step"><strong>Merge</strong><span>known rollback</span></div>
</div>

<p class="jo-caption">The device step is not optional just because the PC tests are green. It is the reason the change exists.</p>

## PR checklist template

Copy this into the PR description. Delete the prompts that do not apply; do not delete the
verification and rollback sections.

```markdown
## What changed

<!-- One paragraph. User-visible behaviour, not a file list. -->

## Why

<!-- The problem, the context, the issue link. What happens if we do nothing? -->

## How

<!-- The design, the alternative rejected, and the legacy seam if any. -->

## Verification

- [ ] Build: <command / CI job>
- [ ] PC tests: <command, result>
- [ ] Device test: <what was run, result>
- [ ] Error / negative case: <what was forced, result>
- [ ] Performance or memory: <before -> after, same input and device>
- [ ] Not verified: <what is still open, and why>

## Risk and rollback

<!-- What can break, who is affected, how to revert, whether data/config migrates. -->

## Checklist

- [ ] One concern; the diff is as small as it can be
- [ ] Legacy code untouched, or the seam is documented
- [ ] Ownership, lifetime and error handling are explicit
- [ ] Threads stop and join; no detached owner
- [ ] A test covers the change and a failure case
- [ ] Format / lint / rule checks are clean
- [ ] No secrets, no credential in logs
- [ ] No new dependency without a licence and maintenance decision
- [ ] Logs are actionable and do not leak sensitive data
- [ ] Docs, diagram or runbook updated
- [ ] The rollback has been thought through
```

The code shapes behind these rules are collected in [C++ patterns]({{ '/cpp/' | relative_url }}):
ownership, seams, state, builders, leases and the idioms that keep them safe.

## For the reviewer

- **Does it do one thing?** If the description needs “and”, ask for a split.
- **Is the failure path tested?** The happy path is the easy half.
- **Is the legacy seam explicit?** A behaviour change hidden in a cleanup is a review failure.
- **Is the claim measured?** Ask for the input, the hardware and the before/after.
- **Can it be rolled back?** If not, the risk section is incomplete.
- **Is the check gate green for the right reason?** An exclusion needs a reason.

## Definition of done

A change is done when the code, the test, the documentation and the verification agree with each
other, and when the person on call can undo it without guessing. “Works on my machine” is a
starting point, not a definition.

<div class="jo-note">
  These rules are intentionally generic. Adapt the wording and the check commands to your team;
  keep the parts that make a change reviewable, testable and reversible.
</div>
