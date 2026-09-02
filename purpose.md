# Security Benchmark to Machine-Executable Hardening Policies

## Core Idea

The project turns a security benchmark PDF (like a CIS hardening document) into **machine-executable hardening policies** — things an agent can actually run on a server to check and fix security.

Each step builds on the previous one.

## Step 1 — Discover (`discover`)

- **Purpose:** Point it at a PDF; it reads it and proposes a config file (`benchmark.toml`) describing the document — how many controls, sections, etc.
- **How it helps:** Sets up the whole project. You verify/correct the numbers, then it's the foundation everything else reads.
- **Next step:** Install that config, then extract.

## Step 2 — Extract (`run --to-stage risk`)

- **Purpose:** Actually parses the PDF into individual **per-control** units (each security recommendation becomes its own record). Also scores risk.
- **How it helps:** Turns a big unstructured PDF into structured, per-control data the pipeline can work on one at a time.
- **Next step:** Now that controls exist, you write a command per control.

## Step 3 — Author Command Library

- **Purpose:** For each control, you write a shell command that observes the host (e.g. reads a config file), plus a parser that turns the raw output into typed facts.
- **How it helps:** This is the **"detective" stage** — the project now knows how to check each security rule.
- **Next step:** Run those commands on a real machine to capture evidence.

## Step 4 — Capture

- **Purpose:** Runs the commands on an actual test server (as root) and saves what they return as evidence files.
- **How it helps:** Gives real data to prove the commands actually work.
- **Next step:** Check that the commands' stated "facts" match the captured data.

## Step 5 — Contract (`contract_diff`)

- **Purpose:** Compares what your parsers say a control needs vs. what CIS's own tooling says — a second opinion.
- **How it helps:** Catches controls that collect none of the evidence CIS actually asks for.
- **Next step:** Write validation rules (pass/fail logic) per control.

## Step 6 — Validation Rules

- **Purpose:** Author the actual pass/fail rules — **"if X then this control passes, else fails."**
- **How it helps:** This is how a report can say a host is compliant or not.
- **Next step:** Prove those rules actually work against the real captured data.

## Step 7 — Value Gate

- **Purpose:** Automated checks that every rule binds to a fact the parser really emitted, and no **"pass"** rests on empty data.
- **How it helps:** Stops silent wrong answers — e.g. a rule passing because it never actually looked at anything.
- **Next step:** Write the fix (remediation) commands.

## Step 8 — Fix Library + Build Policies

- **Purpose:** Writes commands that remediate (fix) non-compliant hosts, then packages everything (audit + fix) into deployable JSON policy envelopes per profile.
- **How it helps:** This is the **actual product** — the shippable artifact an agent runs in production.
- **Next step:** Validate the whole standard is complete.

## Step 9 — Landed (`land_standard.py`)

- **Purpose:** The **"definition of done"** — it checks every stage completed and refuses to say `LANDED` if anything is missing.
- **How it helps:** Prevents you from thinking the work is finished when it isn't (a past recurring failure).
- **Next step:** Nothing — the standard is shipped.

## The Chain in One Line

```text
PDF
  → discover (1)
  → extract (2)
  → commands (3)
  → capture (4)
  → contract (5)
  → validation rules (6)
  → prove rules (7)
  → fix + bundle (8)
  → landed (9)
