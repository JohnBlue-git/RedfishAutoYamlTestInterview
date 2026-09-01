# Insyde Redfish Automated Test — Case Study

> This is a documentation-only excerpt of a live-hardware conformance-testing
> framework built at Insyde Software. Source code, test definitions, live
> credentials, and internal issue tracking have all been removed; what's left
> is the design story, the AI-assisted engineering process behind it, and how
> the tool is used day to day. Commands below illustrate the CLI's shape —
> the entrypoint script itself isn't included in this excerpt.

## What it is

Automated conformance testing for a hardware vendor's schema, property, and
action extensions to **Redfish** — the industry-standard REST API server
hardware exposes for power, thermal, storage, and firmware-update control —
on OpenBMC-based server-management firmware. Every test runs against **real,
live hardware** over HTTPS; there is no mock or simulated BMC anywhere in the
suite, across two hardware generations spanning multiple CPU and GPU vendor
platform configurations.

## Why it exists

DMTF, the standards body behind Redfish, publishes free, open-source
validators — the **Service Validator** and the **Interop Validator**. Both
are used here too, but neither can answer the question this project exists
to answer, for two structural reasons:

- The Service Validator is **GET-only** and treats a **missing property as
  legal** — the standard schema marks almost everything optional, so it can
  confirm *"is what you returned well-formed?"* but never *"did you return it
  at all?"*. Vendor conformance is largely the second question: the vendor's
  own spec says a property **shall** exist, and a platform quietly omitting
  it is a real, recurring failure mode the validator simply cannot see.
- The Interop Validator can express "this must exist," but its model stops at
  the property — it can say a property is writable without saying what a
  write must send, return, or reflect, and it has no way to express an
  *ordered sequence* of operations (create → verify → delete, poll a
  long-running task, reboot-and-wait).

Full comparison: [`DESIGN.md`](DESIGN.md#why-not-just-the-dmtf-validators).

## Scale

| | |
|---|---|
| Schemas covered | 41 resource types, ~75 test definition files |
| Tests collected per run | ~690 |
| Hand-written test modules fully retired in favor of the current engine | ~40, across ~18 schemas |
| Engine's own self-tests (run without any live hardware) | ~215 |
| Destructive-action categories requiring explicit opt-in | 7 |
| Platform coverage | 2 hardware generations, 10 concrete platform configurations |

## How it's used

A single CLI wraps pytest; every flag or argument it doesn't recognize passes
straight through, so the full native pytest vocabulary (`-m`, `-k`, `-v`, …)
always stacks on top of it.

```bash
# Full suite against the configured rig, terminal output
python3 run_tests.py

# Same, plus a self-contained HTML report
python3 run_tests.py --html

# One schema, filtered to one platform
python3 run_tests.py -m "<platform> and Chassis"

# A destructive action stays off by default no matter what's selected -
# it needs its own explicit, named opt-in
python3 run_tests.py --allow-bmc-reset tests/Manager/
```

That last pattern is a deliberate safety choice, not an afterthought: a
handful of action categories (host power control, a full BMC reset, a BIOS
password change, a UEFI secure-boot key change, …) each require their own
`--allow-<category>` flag, independent of test selection entirely — so a
marker expression alone can never trigger a factory reset. More in
[`DESIGN.md`](DESIGN.md).

## Read next

- **[`PROJECT.html`](https://htmlpreview.github.io/?https://github.com/JohnBlue-git/RedfishAutoYamlTestInterview/blob/main/PROJECT.html)** — the whole case study as a single
  visual page: overview, architecture, and the AI-workflow narrative together (open in a browser).
- **[`DESIGN.md`](DESIGN.md)** — the architecture, the three
  generations this suite evolved through, why the off-the-shelf validators
  fall short, and the design concepts that came out of it (prose form).
- **[`AI_WORKFLOW.md`](AI_WORKFLOW.md)** — how this was actually
  built working with an AI pair: the process, what it caught, and what it
  cost (prose form).
