<style>
h1, h2, h3, h4, h5, h6 {
  color: #1976d2 !important;
}
</style>

# Design

The concepts behind this test suite, and *why* it took this shape rather
than raw pytest, its two predecessors, or the off-the-shelf DMTF validators.
This is concept-level only; implementation detail is intentionally left out.

## Contents

| § | Section |
|---|---|
| 1 | [Architecture at a glance](#1-architecture-at-a-glance) |
| 2 | [Three generations](#2-three-generations) |
| 3 | [Why not just the DMTF validators?](#3-why-not-just-the-dmtf-validators) |
| 4 | [Key design concepts](#4-key-design-concepts) |
| 5 | [Feature set](#5-feature-set) |
| 6 | [Trade-offs](#6-trade-offs) |

## 1. Architecture at a glance

One pytest process, one live BMC, one declarative way of describing tests:

```
CLI wrapper        (unknown flags pass straight through to pytest)
      │
      ▼
pytest collection  → a custom hook reads each schema's YAML file
      │              and builds real pytest Collector / Item objects
      ▼
live BMC over HTTPS  (rig connection + platform inventory feed each request;
      │               no mock BMC anywhere)
      ▼
one report         (a row per test step; every request shown as a
                    replayable curl command)
```

**A pytest plugin, not a pytest replacement.** Every declarative construct
becomes a real pytest object — a file becomes a Collector, a test case or
step becomes an Item, a platform declaration becomes a marker. So everything
that already understands pytest (`-m`, `-k`, exit codes, CI) keeps working,
and old hand-written tests could run side by side during migration.

## 2. Three generations

**Each generation fixed the biggest problem of the one before it.**

| | Fixed | Stopped at |
|---|---|---|
| **1. Web test-management tool + in-house keyword runner** | Execution: regression runs became automatic | Maintenance: a language nobody else knows, one test spread over three places, numbered steps, nothing in git |
| **2. Hand-written pytest** | Language and history: mainstream Python, reviewed in git | Scale: boilerplate files per schema, platform support wired through mixin classes, one Pass/Fail per multi-step flow |
| **3. Config-driven (current)** | Scale: tests become data, with pytest still underneath | Cost: a field vocabulary specific to this project |

The lessons carried forward:

- **From generation 1:** an in-house runner ends up maintainable only by its
  author — so keep a mainstream runner (pytest) underneath.
- **From generation 2:** deciding platform support in Python class hierarchies
  doesn't scale — so declare it as data, resolved at collection time.

## 3. Why not just the DMTF validators?

DMTF publishes free, authoritative validators. They are used too — but they
answer a different question.

- **Service Validator** checks that what a service returns is valid Redfish.
  It is **read-only** and treats a **missing property as legal**, because the
  standard marks most properties optional. Vendor conformance is mostly the
  opposite question: the vendor spec says a property *shall* exist.
- **Interop Validator** can say "must exist", but it never actually sends a
  write or invokes an action, and it has no notion of an ordered sequence
  (create → verify → delete, poll a task, reboot and wait).

**The division of labour:** a pure-read resource with no vendor extension is
the validator's job. Anything with a write, an action, a vendor extension, or
a must-exist property belongs to this suite.

## 4. Key design concepts

**Tests are data.** One YAML file per schema. Tests say *what*; the engine
decides *how*. One engine change (curl capture, report filters, a new
safety rule) upgrades every test at once, with no test edits. A Redfish-
literate QA engineer can add coverage without knowing pytest.

**The three-way split.** Every test is assembled from three independent
layers:

1. *which rig* — connection details;
2. *what that hardware has* — a per-platform inventory of resource ids;
3. *what the spec requires* — compliance data generated from the vendor
   spec, with per-platform deltas merged automatically.

That is what lets one test definition run unchanged on every platform, and
makes a new platform a data change rather than a code change.

**Platform tiers.** Applicability is declared against a tier hierarchy
(common → chip generation → customer → platform). A platform added to a tier
picks up every matching test automatically.

**Expected, not discovered.** The inventory says what a platform *should*
have. A device it names but the BMC doesn't return is a failure, not
something a loop silently skips.

**Sequences.** Multi-step flows are ordered lists of steps. Each step is its
own report row; values pass between steps; inserting a step renumbers
nothing; after a failure the rest are skipped instead of producing noise.

**Write, reflect, restore.** A write is followed by a re-read to prove it
took effect, and the original value can be put back — rigs are shared and
must be left as found.

**Safety as an orthogonal gate.** Dangerous action categories (host power,
BMC reset, firmware update, secure-boot keys, …) each need their own
`--allow-<category>` flag, independent of test selection. A marker
expression alone can never trigger a reset.

**Report findings honestly.** When hardware doesn't meet the spec, the test
is written as if it should pass and allowed to fail — never loosened to go
green. And because compliance data comes from the spec, a red row always
raises the question: is the firmware wrong, or the spec?

**No mock BMC.** A mock would only test the mock. The engine itself is
covered by its own self-test suite in a synthetic pytest sandbox, so it can
be changed without hardware.

## 5. Feature set

The engine has 19 features. Each one names a testing pattern that the
hand-written suite used to reinvent in Python, module by module.

| Area | Features |
|---|---|
| **Identity and checks** | 1. a fixed test-id grammar (`<Category>.<HTTP>[.<Property\|Action>]`) · 2. verification checks: status, property patterns, compliance against the spec |
| **Gating** | 3. platform gating by tier · 4. risk gating with a named opt-in per category |
| **Fan-out** | 5. static fan-out over a platform's expected resource ids, nested to any depth · 6. runtime fan-out over ids discovered from a live response |
| **Writes** | 7. restore the original value after a write · 8. non-mutating writes that send back the current value · 9. named request variants, each its own report row · 10. request bodies from files · 11. non-JSON bodies (form-encoded, multipart file upload) |
| **Flows** | 12. sequences of steps · 13. steps that only run inside a sequence · 14. reusable non-test actions (e.g. wait for reboot) · 15. static inputs · 16. retry for flaky resources · 17. setup and teardown at three scopes |
| **Requests** | 18. placeholder substitution that preserves types · 19. per-request headers, auth, cookies, host and environment |

The test files use these features unevenly. Expected-id fan-out appears in
143 of the 243 test cases and named variants in 98; retries in 6.

## 6. Trade-offs

- **The learning curve moved, it didn't vanish.** Raw-pytest knowledge
  transfers anywhere; this engine's field names are project-specific. The
  syntax is plain YAML, though, and AI skills encode the vocabulary.
- **Narrower standard-schema coverage than the Service Validator.** If
  standard-schema conformance were the only requirement, this project would
  not be justified.
- **An escape hatch is kept.** Anything no field can express is a plain
  Python function — used only after checking that no field fits.
