# Hardening Pipeline: Phase and Stage Map

This project turns a CIS benchmark PDF into reviewable, executable compliance
artifacts: read-only audit commands, typed evidence, validation rules,
remediation recipes, and deployable per-profile policies.

There are **eight lifecycle phases** for people working on a benchmark. The
pipeline implements the work through **sixteen executable stages**. Stages are
resumable: each records its inputs, output, version, and completion state in
the benchmark manifest; a changed input or upstream artifact rebuilds only the
affected downstream work.

## Lifecycle phases and their stages

| Phase | Goal | Pipeline stages |
|---|---|---|
| 1. Discover | Understand a new CIS document and define its configuration. | Pre-pipeline `discover` command |
| 2. Extract | Turn the PDF into an accurate, canonical benchmark. | `layout`, `inventory`, `content`, `appendix`, `mappings`, `finalize`, `refine`, `split` |
| 3. Enrich | Attach risk context and, when required, research-backed contracts. | `risk`, `research` |
| 4. Command library | Build safe collection entries that emit typed facts. | `commands` |
| 5. Capture and evidence | Prepare collection and connect commands to committed host evidence. | `probe`, `evidence` |
| 6. Validation | Admit rules that can evaluate the collected facts safely. | `validation` |
| 7. Remediation | Build separate, privileged fix artifacts. | `fix` |
| 8. Release and landing | Assemble a complete deliverable and prove it is ready to ship. | `bundle`, then policy builders and `land_standard.py` |

`discover`, policy builders, and `land_standard.py` are workflow tools rather
than members of the sixteen-stage `STAGES` tuple.

## The sixteen executable stages

| # | Stage | Lifecycle phase | Purpose | How it helps the next process |
|---:|---|---|---|---|
| 1 | `layout` | 2. Extract | Reads the PDF's native text layer and preserves positional/layout information. | Gives `content` the reliable page text and structure needed to reconstruct controls and code blocks. |
| 2 | `inventory` | 2. Extract | Reads the Summary Table to derive the authoritative control IDs, sections, statuses, and expected counts. | Gives `content` and later gates the complete control universe; prevents a partial extraction from looking complete. |
| 3 | `content` | 2. Extract | Extracts each control's title, recommendation body, audit procedure, remediation text, and hierarchy. | Supplies the control material that `finalize` turns into the canonical benchmark. |
| 4 | `appendix` | 2. Extract | Parses CIS Implementation Group appendix membership. | Adds profile/IG applicability that `finalize` reconciles into each control. |
| 5 | `mappings` | 2. Extract | Parses control mapping tables found in the PDF. | Supplies additional mapping evidence for reconciliation in `finalize`. |
| 6 | `finalize` | 2. Extract | Merges inventory, content, appendix, and mappings into schema-validated `benchmark.json` and Markdown. | Produces the canonical document representation that `refine` can safely improve without changing meaning. |
| 7 | `refine` | 2. Extract | Repairs layout-derived issues such as wrapped command lines and code/prose boundaries. | Produces a cleaner, stable benchmark representation for per-control splitting. |
| 8 | `split` | 2. Extract | Writes one normalized JSON unit per control plus an index. | Creates the per-control work units consumed independently by risk, research, commands, validation, and fixes. |
| 9 | `risk` | 3. Enrich | Calculates or attaches risk metadata for every control. | Gives command entries and release artifacts consistent risk context. |
| 10 | `research` | 3. Enrich | Looks up committed, content-addressed research records for research-mode standards; it is skipped in normal `bare` mode. | Provides independently declared collection/validation contracts to `commands` when CIS text alone was insufficient. |
| 11 | `commands` | 4. Command library | Assembles per-control, agent-executable read-only commands, parser output contracts, profiles, risk, and execution settings. | Gives `probe`, `evidence`, `validation`, and `fix` the exact collection interface they must reference. |
| 12 | `probe` | 5. Capture and evidence | Generates collection/probe artifacts and operator handoff material. | Lets an operator run the correct commands on the target host and commit an authoritative capture. |
| 13 | `evidence` | 5. Capture and evidence | Associates each command with the committed authoritative capture and records whether evidence is conformant. | Makes evidence coverage explicit for reviewers and the final bundle, rather than assuming a command was tested. |
| 14 | `validation` | 6. Validation | Assembles authored validation rules and checks every referenced fact and field against the command's evidence contract. | Ensures rules can safely score collected facts as `pass`, `fail`, `error`, or `manual`, and supplies the evaluation layer for release. |
| 15 | `fix` | 7. Remediation | Assembles remediation recipes derived from the command/validation context. | Provides optional, separately privileged fix material for deployment and the release bundle. |
| 16 | `bundle` | 8. Release and landing | Merges per-control artifacts, writes coverage and control-ledger data, and rejects incomplete required stages by default. | Produces the reviewable release input for policy/frontend/database builders; `land_standard.py` then verifies the complete shipping state. |

## Important handoff rules

- **Commands observe; rules decide.** Collection commands emit typed facts, not
  compliance verdicts. Validation is the separate decision layer.
- **Silence is not proof.** An absent result must be explicitly observable;
  unreadable or failed collection is an `error`, not a `fail` or a `pass`.
- **Collection is read-only.** The audit library never changes a host. Fixes
  are generated and deployed through the separate remediation path.
- **Per-control work can be blocked without stopping all work.** The final
  `bundle` stage is where incomplete required coverage becomes a release blocker.
- **`library/` is committed source; `output/` is rebuildable.** The library
  holds authored commands, rules, evidence, and policies. Output holds
  generated stage artifacts and the manifest.

## Typical commands

```powershell
# Inspect configured standards
python -m benchmark_pipeline standards

# Check which stages are stale before rebuilding
python -m benchmark_pipeline run --standard <key> --version <version> --check-only

# Run the full sixteen-stage pipeline
python -m benchmark_pipeline run --standard <key> --version <version>

# Run a bounded segment while developing
python -m benchmark_pipeline run --standard <key> --version <version> `
  --from-stage commands --to-stage bundle

# Final completeness gate
python tools/land_standard.py --standard <key> --version <version>
```
