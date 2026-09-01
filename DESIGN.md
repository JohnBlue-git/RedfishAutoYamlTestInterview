<style>
h1, h2, h3, h4, h5, h6 {
  color: #1976d2 !important;
}
</style>

# Design

Architecture of this test suite, and *why* it exists in this form rather than
raw pytest or the off-the-shelf DMTF validators.

This is a condensed, case-study version of a larger internal design set.
Implementation-level detail — source modules, the full YAML field reference,
ticket-by-ticket history — has been left out on purpose. What's kept is the
*why*.

## Contents

| § | Section |
|---|---|
| 1 | [Architecture at a glance](#architecture-at-a-glance) |
| 2 | [Three generations of this test suite](#three-generations-of-this-test-suite) |
| 3 | [Why not just the DMTF validators?](#why-not-just-the-dmtf-validators) |
| 4 | [Design concepts worth naming](#design-concepts-worth-naming) |
| 5 | [Migration outcome](#migration-outcome) |

## Architecture at a glance

One pytest process, one live BMC, one declarative way of feeding it test cases:

```
CLI entrypoint   (unrecognized flags/args pass straight through to pytest)
        │
        ▼
pytest collection
        │
        ▼
a schema's test-definition file
(a custom collection hook turns this into a real pytest Collector/Item tree —
 see "A pytest plugin, not a pytest replacement" below)
        │
        ▼
live BMC over HTTPS
(per-rig connection config + per-platform resource-topology data feed every
 request; no mock/fixture BMC anywhere)
        │
        ▼
pass/fail per test, one report (terminal, or a self-contained HTML file)
```

**A pytest plugin, not a pytest replacement.** The engine never invents its
own execution model — every declarative construct becomes a real pytest
object, reached through one of pytest's own hooks:

| Declarative construct | Becomes | pytest hook |
|---|---|---|
| a schema's test file | a Collector | `pytest_collect_file` |
| a test case / a payload / a sequence step | an Item | `item.runtest()` |
| a platform-applicability declaration | a marker | `item.setup()` |
| a setup/teardown declaration | a fixture-like hook | `item.setup()` / `item.teardown()` |
| a verification declaration | an assertion | raised inside `runtest()` |

That matters because everything that already understands pytest — `-m`,
`-k`, `--collect-only`, exit codes, CI integration — keeps working
unmodified; the declarative layer sits *on top of* pytest's own model
instead of beside it.

## Three generations of this test suite

Conformance testing here went through three forms, each replacing the one
before because the previous shape broke down at scale.

### 1. TestLink — manual, prose-based

Coverage lived in a web-based test-management tool: prose test cases with
expected results, executed by a human who recorded Pass/Fail per case. It was
good at what a management tool is good at — requirement traceability, and
letting anyone author a case without writing code. It broke down for two
structural reasons: **every regression run was human effort**, and **a
linear list of prose steps cannot express a conditional** ("only on this
platform, verify X") without either duplicating the case per branch — which
drifts independently over time — or pushing the condition into prose, which
becomes unverifiable: nothing confirms the tester evaluated it correctly, and
the recorded Pass covers an unknown path.

That is the core reason this moved to real code at all: **conditional logic
belongs somewhere a machine evaluates it.**

### 2. Hand-written pytest — real code, real branching

Every schema got a hand-written test module — real execution, real
`if`/`class` branching, strictly more expressive than anything before or
after it, since plain code can do whatever a test needs. What broke down was
*consistency at scale*, across roughly 40 modules:

- **Platform applicability was decided at design time and wired through
  class inheritance.** Adding a platform meant editing classes; a new
  *combination* meant a new mixin. Fifteen of these accumulated — and by the
  time the suite finished migrating off this style, not one was still in use.
- **Hardware-topology data** (which sensors, fans, or devices a given
  platform has) **was scattered** across dozens of small per-schema helper
  modules instead of living in one place.
- **Test-discovery conventions blocked reuse** — a test step could never
  simply be called by another test, only wrapped in a fixture or a separate
  helper module.
- **Reporting was one lump per test function** — a create → verify → delete
  flow reported as a single Pass/Fail, with no visibility into which step
  actually failed.

### 3. Config-driven declarative tests — the current, and only, form

Every test today is a declarative file, collected by a small custom pytest
engine rather than pytest's native class/function discovery. A test case
declares an id, a request path, expected-response checks, optional
platform/risk gating, and — for a write operation — one or more payloads. A
**Group** sequences several test cases as an ordered set of steps, threading
data from one step's response into the next.

| | TestLink | Hand-written pytest | Config-driven (current) |
|---|---|---|---|
| Who can author | anyone — prose | code-fluent only | anyone who knows the API + the field vocabulary |
| Execution | manual | automated | automated |
| Conditional logic | none (see above) | native `if`/`match` | declarative (platform tiers, per-tier deltas) |
| Pass/fail granularity | per case, often several branches at once | per test function | per case, per payload, **and per sequence step** |
| Requirement traceability | strongest feature | none | partial |
| Learning curve | low to author, tedious to execute | the host language itself | this engine's own vocabulary — a real, stated cost (below) |

**The cost, stated plainly.** This engine does not remove the learning curve
of the generation before it — it *relocates* it. Raw-code knowledge transfers
to any project in that language; this engine's field vocabulary is
repo-specific. That trade was made deliberately: the vocabulary is smaller
and far more uniform than the code it replaced, it's declarative enough for a
non-programmer to contribute real coverage, and the same declaration drives
execution, platform gating, and reporting at once.

## Why not just the DMTF validators?

The most reasonable objection to this project's existence: DMTF already
publishes free, open-source, authoritative validators. Why write anything?

- **Redfish Service Validator** — crawls a live service from the root and
  checks every resource it finds against the official standard schema
  bundle. *Is what this service returns well-formed?*
- **Redfish Interop Validator** — checks a service against a declared
  Interoperability Profile: which resources/properties are required or
  recommended for a given use case. *Does it meet a stated set of
  requirements?*

Both are used here too, alongside this suite rather than instead of it —
because neither can do this job:

| | DMTF Service Validator | DMTF Interop Validator | This suite |
|---|---|---|---|
| Standard-schema conformance | ✅ automatic, whole tree | partial — only what the profile lists | hand-curated — more work, less breadth |
| Property must-exist requirement | ✗ absent ≠ failure | ✅ can declare it mandatory | ✅ the default assumption |
| Vendor extension properties | ✗ not covered by the standard schema at all | possible, hand-authored in a second dialect | ✅ same file, same mechanism as everything else |
| Per-platform-tier variation | ✗ | awkward — a separate profile per platform | ✅ automatic, merged deltas |
| Hardware-topology expectations | ✗ validates only what it finds | partial | ✅ names the expected inventory per platform |
| Write operations (PATCH/POST/PUT/DELETE) | ✗ read-only by design | states a capability, never sends a write | ✅ sends the write, verifies the result, can restore the original value |
| Actions invoked | ✗ | ✗ | ✅ |
| Ordered multi-step flows | ✗ | ✗ no sequencing concept | ✅ create → verify → delete, poll a task, reboot-and-wait |
| Destructive-action safety | n/a — never writes | n/a | ✅ explicit opt-in flag per risk category, independent of test selection |

**Where the validators are honestly better, and this project pays for it:**
breadth. The Service Validator checks *every* resource the service exposes,
automatically, from the official bundle; this suite's compliance data is
hand-maintained and narrower — and if a maintained file's properties are ever
left mismarked, it can silently validate nothing at all. *(This exact failure
mode was caught more than once in practice — see the AI-workflow doc's "what
this caught."*) If standard-schema conformance were the only requirement,
this project would not be justified.

The two are complementary, and the rule that falls out of it is simple:
**a pure-GET resource with no vendor extension is the validator's job;
anything with a write path, an invoked action, a vendor extension, or a
must-exist property is this suite's.**

## Design concepts worth naming

**The three-way split.** Every test's behavior is assembled from three
independent layers that never overlap: *which rig* (connection info — IP,
credentials, per-rig endpoints), *what that hardware actually has* (a
per-platform inventory of resource ids), and *what the spec requires of it*
(the compliance data, with per-platform-tier deltas merged automatically,
most-specific wins). Keeping these three separate is what lets the same test
definition run unmodified against ten different physical machines.

**Safety is a first-class, orthogonal gate.** A small number of action
categories — host power control, firmware update, a full BMC reset, a BIOS
password change, a UEFI secure-boot key change, and similar — are dangerous
enough on real hardware that none run by default under any test-selection
expression. Each needs its own explicit command-line opt-in, deliberately
independent of which tests were otherwise selected, so a marker expression
alone can never trigger a factory reset.

**A finding is reported, not hidden.** When live hardware genuinely doesn't
do what the spec requires, the house style is to write the test as though it
should pass, let it fail honestly, and record the finding rather than
quietly loosening the check. A green report that hides a real firmware gap
is worse than a red one that names it.

## Migration outcome

The hand-written generation is fully retired — no hand-written test module
remains anywhere in the suite; every test that runs today is config-driven.
The migration happened incrementally, schema by schema, running both styles
side by side in the same test session throughout, rather than as one
disruptive cutover.
