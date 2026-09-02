# AI Agent Architecture — COMPAILANCE-AUTOMATION

## Overview

6 AI agents working together, orchestrated by a central brain. Built on LangGraph + Claude API.

---

## Agent Count & Roles

| # | Agent | Stages Owned | Tools | Role |
|---|---|---|---|---|
| 1 | **Orchestrator** | — | 6 | Main brain: understands intent, routes, tracks state |
| 2 | **Extraction** | 1-6 (discover→finalize) | 9 | PDF parsing, text extraction, canonical JSON |
| 3 | **Authoring** | 7-9 (refine→commands) | 21 | Commands, parsers, validation rules, fixes |
| 4 | **Verification** | 10-12 (probe→validation) | 15 | Probes, evidence matching, rule evaluation |
| 5 | **Remediation** | 13-16 (fix→bundle) | 19 | Fixes, bundles, deployable policies, frontend |
| 6 | **QA** | Cross-cutting | 14 | Enforces 16 evidence gates, completeness checks |

---

## 1. Orchestrator Agent (Main Brain)

**Role**: Understands user intent, routes to specialist agents, tracks progress, answers questions

**Tools**:

| Tool | Source | Purpose |
|---|---|---|
| `list_standards` | `config.available_benchmarks()` | List all configured standards |
| `get_standard_config` | `config.load_benchmark()` | Load a standard's full config |
| `check_pipeline_status` | `pipeline.stage_status()` | Check what's stale |
| `run_pipeline_stage` | `pipeline.run_pipeline()` | Execute a pipeline stage |
| `read_manifest` | `pipeline._load_manifest()` | Read pipeline state |
| `answer_question` | RAG over docs + library | Answer about any control/rule/gate |

---

## 2. Extraction Agent (PDF → Structured Data)

**Role**: Handles stages 1-6 — turns a raw PDF into canonical JSON

**Stages**: `discover → layout → inventory → content → appendix → mappings → finalize`

**Tools**:

| Tool | Source | Purpose |
|---|---|---|
| `discover_pdf` | `discover.run_discover()` | Propose benchmark.toml from new PDF |
| `extract_layout` | `extract.run_layout_extraction()` | PyMuPDF text extraction |
| `parse_inventory` | `extract.parse_summary_inventory()` | Parse Summary Table |
| `parse_controls` | `extract.parse_controls()` | Extract per-control content |
| `parse_appendix` | `cis_mapping.parse_appendix_membership()` | Parse IG appendices |
| `parse_mappings` | `cis_mapping.parse_control_tables()` | Parse CIS mapping tables |
| `finalize_benchmark` | `pipeline` (finalize stage) | Merge + validate + render |
| `validate_counts` | `pipeline.validate_inventory()` | Check counts match document |
| `validate_final` | `pipeline.validate_final()` | Full validation of canonical output |

---

## 3. Authoring Agent (Commands + Rules + Fixes)

**Role**: Handles stages 7-9 — builds command library, validation rules, and fix library per control

**Stages**: `refine → split → risk → research → commands → fix`

**Tools**:

| Tool | Source | Purpose |
|---|---|---|
| `refine_benchmark` | `refine.refine_benchmark()` | Fix wrapped paths, code blocks |
| `split_controls` | `split.run_split()` | Split into per-control JSON |
| `score_risk` | `risk.run_risk()` | Risk score each control |
| `lookup_research` | `research.run_research()` | Content-addressed research cache |
| `build_commands` | `commands.run_bare_commands()` | Build command + parser + contract |
| `validate_commands` | `commands_validate.validate_entries()` | Lint commands |
| `build_fix` | `fix.run_fix()` | Generate remediation scripts |
| `validate_fix` | `fix.validate_fix()` | Lint fix scripts |
| `author_validation_rule` | (LLM generates rule JSON) | Author new validation rules |
| `check_validation` | `tools/check_validation.py` | Lint rules against contracts |
| `gate_validation` | `tools/gate_validation_bare.py` | Gate: honesty invariants |
| `plan_validation` | `tools/plan_validation.py` | Generate authoring packets |
| `check_fact_names` | `tools/check_fact_names.py` | Fact name vocabulary check |
| `contract_diff` | `tools/contract_diff.py` | Independent contract opinion |
| `draft_registry_commands` | `tools/draft_windows_registry_commands.py` | Draft Windows registry commands |
| `draft_registry_rules` | `tools/draft_windows_registry_rules.py` | Draft Windows registry rules |
| `draft_secedit` | `tools/draft_windows_secedit.py` | Draft secedit commands |
| `draft_auditpol` | `tools/draft_windows_auditpol.py` | Draft auditpol commands |
| `bind_observability` | `tools/bind_observability.py` | Bind observability to rules |
| `bind_registry_type` | `tools/bind_registry_value_kind.py` | Bind value type assertions |
| `generate_fix_family` | `tools/generate_family.py` | Generate fix recipe families |

---

## 4. Verification Agent (Empirical Testing)

**Role**: Handles stages 10-12 — runs probes, matches evidence, evaluates rules

**Stages**: `probe → evidence → validation`

**Tools**:

| Tool | Source | Purpose |
|---|---|---|
| `emit_probe` | `probe.run_probe_stage()` | Generate probe scripts + handoff |
| `run_vm_probe` | `tools/vm_probe.py` | Run probe on VMware guest |
| `run_bare_probe` | `tools/bare_probe.py` | Take root-privilege bare capture |
| `run_windows_probe` | `tools/windows_probe.py` | Windows probe runner |
| `run_native_probe` | `tools/native_probe.py` | Native plugin probe |
| `run_emit` | `tools/run_emit.py` | Run emit prelude |
| `ingest_evidence` | `evidence.run_evidence()` | Match captures to scripts |
| `validate_evidence` | `evidence.validate_evidence()` | Validate evidence stage |
| `evaluate_rules` | `validation.run_validation()` | Evaluate validation rules |
| `validate_validation` | `validation.validate_validation()` | Lint validation stage |
| `eval_single_rule` | `tools/eval_validation.py` | Evaluate one rule against facts |
| `materialise_facts` | `tools/materialise_facts.py` | Materialise facts from captures |
| `derive_env` | `tools/derive_env_from_capture.py` | Derive environment from capture |
| `report_capture` | `tools/report_capture.py` | Report on capture contents |
| `parse_bare` | `tools/parse_bare.py` | Verdict-equivalence gate (research mode) |

---

## 5. Remediation Agent (Fix + Bundle + Deploy)

**Role**: Handles stages 13-16 — builds fixes, bundles release, generates deployable policies

**Stages**: `fix → bundle` + policy assembly

**Tools**:

| Tool | Source | Purpose |
|---|---|---|
| `run_fix` | `fix.run_fix()` | Generate remediation scripts |
| `run_bundle` | `bundle.run_bundle()` | Merge all per-control artifacts |
| `build_bare_policy` | `tools/build_bare_policy.py` | Build shell/exec agent envelope |
| `build_native_policy` | `tools/build_native_policy.py` | Build native-plugin agent envelope |
| `build_fix_policy` | `tools/build_fix_policy.py` | Build fix agent envelope |
| `build_agent_policy` | `tools/build_agent_policy.py` | Build agent policy |
| `build_final_benchmark` | `tools/build_final_benchmark.py` | Build final benchmark artifact |
| `build_card_data` | `tools/build_card_data.py` | Build per-control card data |
| `build_explorer` | `tools/build_control_explorer.py` | Build offline HTML explorer |
| `build_master_db` | `tools/build_master_db.py` | Build normalized MSSQL database |
| `check_master_db` | `tools/check_master_db.py` | Validate master database |
| `build_remediation_bundle` | `tools/build_remediation_bundle.py` | Bundle remediation scripts |
| `build_remediation_script` | `tools/build_remediation_script.py` | Build individual remediation script |
| `fix_coverage` | `tools/fix_coverage.py` | Measure fix coverage |
| `fix_feasibility` | `tools/fix_feasibility.py` | Assess fix feasibility |
| `fix_breakers` | `tools/fix_breakers.py` | Identify fix breakers |
| `gate_fix` | `tools/gate_fix.py` | Gate: safety + rollback check |
| `score_bundles` | `tools/score_bundles.py` | Score release bundles |

---

## 6. QA Agent (Cross-Cutting Quality Gates)

**Role**: Enforces the 16 evidence gates, validates correctness across all stages

**Stages**: None (cross-cutting — called by any agent before claiming done)

**Tools**:

| Tool | Source | Purpose |
|---|---|---|
| `land_standard` | `tools/land_standard.py` | Definition of done |
| `check_validation` | `tools/check_validation.py` | Lint rules against contracts |
| `gate_validation_bare` | `tools/gate_validation_bare.py` | Honesty invariants gate |
| `check_fact_names` | `tools/check_fact_names.py` | Fact name vocabulary |
| `gate_native` | `tools/gate_native.py` | Native plugin parity gate |
| `gate_fix` | `tools/gate_fix.py` | Fix safety gate |
| `parse_bare` | `tools/parse_bare.py` | Verdict-equivalence gate |
| `validate_agent_output` | `tools/validate_agent_output.py` | Validate agent output |
| `write_validation_report` | `tools/write_validation_report.py` | Write final report |
| `test_read_only` | `tests/test_command_library_is_read_only.py` | Collector read-only check |
| `test_no_silence` | `tests/test_no_pass_on_silence.py` | No pass on silence |
| `test_empty_capture` | `tests/test_no_pass_on_an_empty_capture.py` | Empty capture check |
| `test_type_correctness` | `tests/test_rule_value_types.py` | Typed literal check |

---

## Interaction Flow

```
User: "Onboard CIS Ubuntu 24.04 v2.0.0"
        │
        ▼
   ORCHESTRATOR
        │
        ├─→ EXTRACTION: "Run discover + extract"
        │       └─→ returns: benchmark.json, 333 controls
        │
        ├─→ AUTHORING: "Build commands for all 333"
        │       └─→ returns: commands, rules, fixes
        │
        ├─→ VERIFICATION: "Validate against capture"
        │       └─→ returns: 320 pass, 8 error, 5 manual
        │
        ├─→ QA: "Run all gates"
        │       └─→ returns: LANDED ✓
        │
        └─→ REMEDIATION: "Build deployable policies"
                └─→ returns: agent envelopes, frontend, DB
```

---

## Tech Stack

| Component | Choice |
|---|---|
| LLM | Claude API (Anthropic) |
| Agent Framework | LangGraph (LangChain) |
| Observability | LangSmith |
| Memory | SQLite / PostgreSQL |
| RAG | ChromaDB or Qdrant |
| PDF Parsing | PyMuPDF (existing) |
| Frontend | Streamlit or Gradio |
| Runtime | Python 3.12+ |
| Deployment | Docker + systemd |

---

## Tool Count Summary

| Agent | Tool Count | Unique |
|---|---|---|
| Orchestrator | 6 | 6 |
| Extraction | 9 | 9 |
| Authoring | 21 | 21 |
| Verification | 15 | 15 |
| Remediation | 19 | 19 |
| QA Agent | 14 | 14 |
| **Total** | **84** | **~75 unique** |
