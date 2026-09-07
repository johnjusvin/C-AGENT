# Pipeline Stages

## 1. layout — raw geometry from the PDF

**Purpose:** PyMuPDF reads every text span/block and its (x,y) bounding box, per page. Nothing is interpreted; it's the coordinate map used later to tell code from prose.

**Output — `stages/01_layout.jsonl`** (14.7 MB, one line per page):

```json
{"physical_page":1,"printed_page":1,"width":612.0,"height":792.0,...,
 "blocks":[{"bbox":[72.024,234.246,507.708,326.802],"text":"CIS Ubuntu Linux 24.04\nLTS Benchmark",
   "lines":[{"text":"CIS Ubuntu Linux 24.04","spans":[{"text":"CIS Ubuntu Linux 24.04 ","bbox":[...],
     "font":"ArialMT","size":39.96,"bold":false}]}]}]}
```

**Feeds:** refine uses bbox geometry to decide a monospaced line is a shell command; content locates each control's body by page anchor.

---

## 2. inventory — table of contents + section tree

**Purpose:** Read the Summary Table to list every section/control with page anchors, and validate the count against `benchmark.toml` and the paddle OCR table (a mismatch fails the stage).

**Output — `stages/02_inventory.json`:**

```json
{"sections":[{"id":"1","title":"Initial Setup","kind":"section","summary_page":974,"parent_id":null},
  {"id":"1.1","title":"Filesystem","summary_page":974,"parent_id":"1"},
  {"id":"1.1.1","title":"Configure Filesystem Kernel Modules","summary_page":974,"parent_id":"1.1"},
  {...}], "counts":{"controls":333,...}, "paddle_comparison":{"available":true,"control_count":333,"errors":[]}}
```

**Feeds:** content maps body text onto these ids; mappings needs the id set; finalize uses the sections.

---

## 3. content — control bodies

**Purpose:** For each inventory id, pull description/rationale/impact/audit/remediation/default-value from the body, trusting the body over the Summary Table for titles (title_corrections records every disagreement).

**Output — `stages/03_controls.json`** (333 controls):

```json
{"controls":[{"id":"1.1.1.1","title":"Ensure cramfs kernel module is not available",
  "assessment_status":"automated","profiles":["level_1_server","level_1_workstation"],
  "section_path":["1","1.1","1.1.1"],"cis_controls":[
    {"version":"v7","safeguards":[{"id":"9.2",...}]},{"version":"v8","safeguards":[{"id":"4.8",...}]}],
  "description":{"text":"The cramfs filesystem type is a compressed read-only Linux filesystem ...",
    "blocks":[{"type":"paragraph","text":"...","page":24}]}, ...}],
 "title_starts":333,"title_corrections":0}
```

**Feeds:** finalize (it becomes the controls array of the canonical benchmark).

---

## 4. appendix — CIS Controls membership per implementation group

**Purpose:** Read the appendix tables assigning each control to Implementation Groups (IG1/IG2/IG3).

**Output — `stages/04_appendix_cis.json`:**

```json
{"versions":{"v7":{"groups":{"IG1":["1.1.2.1.1","1.1.2.1.2",...,"5.4.2.2",...], "IG2":[...], "IG3":[...]},
  "counts":{"IG1":141,"IG2":294,"IG3":300,"unmapped":4},"union_count":304}}}
```

**Feeds:** finalize reconciles these memberships with the per-control mapping tables.

---

## 5. mappings — per-control safeguard tables

**Purpose:** Extract each control's own CIS Controls mapping table from the document; flag tables it can't identify.

**Output — `stages/05_cis_safeguards.json`:**

```json
{"table_control_count":333,"unidentified_tables":0,...}   // per-control rows id → v7/v8 safeguard
```

**Feeds:** finalize cross-checks these against the appendix (they can disagree across pages).

---

## 6. finalize — the canonical benchmark (first merge point)

**Purpose:** Merge sections + controls + appendix + mappings into `benchmark.json`, render `benchmark.md`, and validate everything against `benchmark.schema.json`. Errors here fail the run.

**Output — `benchmark.json`:**

```json
{"benchmark":{...},"profiles":[{"id":"level_1_server","name":"Level 1 - Server"}, ...],
 "sections":[...],
 "controls":[{"id":"1.1.1.1","title":"Ensure cramfs kernel module is not available",
   "description":{"text":"...cramfs is a compressed read-only Linux filesystem...",
     "blocks":[{"type":"paragraph","text":"...","page":24}]}, ...}]}
```

Accompanied by `benchmark.md`, `benchmark.schema.json`, `validation_report.json` (status pass).

**Feeds:** refine.

---

## 7. refine — cleaned, repair-word-wrap, code-aware corpus

**Purpose:** Using layout, repair line-wrap-corrupted shell blocks, rejoin split lines, tag nodes as text/heading/list/note/code. It hashes `benchmark.json` before/after to prove refinement never mutates the canonical file. One of the real bugs it fixed: a wrapped registry value whose tail was in got demoted to prose and cut out of a code block.

**Output — `benchmark.refined.json`** (note the canonical `{"text":...,"blocks":[...]}` shape becomes the AST-style content):

```json
"description":{"content":[{"type":"text","text":"The cramfs filesystem type is a compressed read-only
   Linux filesystem embedded in small footprint systems. ...[now joined into one line]"}]}
```

`refinement_report.json` reports: `status:"pass"`, `nodes":{"code":1546,"heading":450,"list":563,"note":262,"text":2446}`, `canonical_sha256_before==after`.

**Feeds:** split.

---

## 8. split — one file per control (pure rearrangement)

**Purpose:** Explode into `stages/06_split/<id>.json`. Carries two digests: `content_sha256` (integrity/reassembly key) and `research_sha256` (text only, id excluded → renumbered-but-unchanged controls are research cache hits).

**Output — `stages/06_split/1.1.1.1.json`:**

```json
{"schema_version":"...","index":0,"control_id":"1.1.1.1",
 "content_sha256":"daf3284cf6ceb8f84575639593ec50872846559a8a7258affae8daf6c4cca845",
 "research_sha256":"f31fe37ad8dde804123161c4e7443fe6a334aa948d89c874d89c7fc176f69ccb",
 "control":{...verbatim control...}}
```

**Feeds:** everything downstream is organized per control.

---

## 9. risk — score every control with the ported ISIS risk engine

**Purpose:** Apply `cis_risk_v1` (weights + severity bands + dimensions) once, emit `risk_model.json` and per-control scores that reference it by name.

**Output — `stages/07_risk/1.1.1.1.json`:**

```json
{"control_id":"1.1.1.1","severity":"medium","score":5.52,
 "dimensions":{"breach_impact":6,"exploitability":6,"privilege_impact":5,"exposure_scope":6,
   "forensic_impact":4,"compliance_impact":6,"ransomware_impact":5},
 "concepts":[{"concept":"container_supply_chain_security","matched_patterns":["\\bimage\\b"]}, ...]}
```

`risk_model.json` once:

```json
{"model":"cis_risk_v1",...,"weights":{"exploitability":0.22,"breach_impact":0.2,...}}
```

**Feeds:** commands (attaches score/severity to each entry).

---

## 10. research — pure cache lookup of authored records

**Purpose:** Resolve `research_sha256` → `library/research/<sha>.json`, never calling a model in the pipeline. A miss blocks that control; in `library_mode="bare"` the stage is skipped outright.

**Output — `stages/08_research/_index.json`** (or a recorded "skipped" entry in bare mode). Note the 24.04 artifact has no `08_research/` dir — it's bare mode, so `manifest.json` records:

```json
{"controls":0,"complete":0,"blocked":0,"status":"skipped", "reason":"[library].mode = 'bare'..."}
```

**Feeds:** commands.

---

## 11. commands — GRC_agent command-library entries

**Purpose:** Adapter that turns the research/CIS text into wire-shape command entries: read-only bash/PowerShell scripts, declared facts, evidence contract, risk, profile applicability, and automation_verdict.

**Output — `stages/09_commands/1.1.1.1.json`:**

```json
{"schema_version":"cis_bare_command_entry/1","control_id":"1.1.1.1",
 "automation_verdict":{"verdict":"automated","review_required":false,"reason":"Parser derived facts
    from real output on the authoritative capture.","inputs":{"assessment_status":"automated","facts_observed":7}},
 "bare_commands":[{"name":"module_directory_scan",
    "description":"CIS step 1... walks every /usr/lib/modules/**/kernel/fs tree... Emits search_path<TAB>resolved<TAB>module_dir<TAB>dir_exists<TAB>dir_non_empty. Backend derives module_available_path rows and module_available.",
    "command_text_sha256":"34faafa366d80e9cf8d9e13bedb3664b2ec06824fe8531d61e25739855cc9fa2"},
   {"name":"loaded_modules","description":"Raw lsmod... sets module_loaded.loaded only on an exact first-column match..."}, ...]}
```

**Feeds:** probe, evidence, validation, fix — it's the contract they all bind against.

---

## 12. probe — Handoff #1: the bundle carried to the VM

**Purpose:** Emit the self-contained probe bundle (4 files, stdlib-only on the guest). `manifest.json` digests are written last so a half-copied bundle is detectable. Nothing on the VM judges anything.

**Output — `output/.../handoff/`:**

- `HANDOFF.md`, `run_probe.py` (byte-copy of `tools/run_probe.py`),
- `probe_bundle.json`:

```json
{"schema_version":"cis_probe_bundle/1","benchmark":...,"control_count":333,"entries":[...all command entries verbose...]}
```

- `manifest.json`:

```json
{"files":{"probe_bundle.json":"947380d8...","run_probe.py":"154d8d7f..."},"stage_versions":{"commands":"1","probe":"1"}}
```

**Feeds:** running `run_probe.py` on the guest produces the tarball that returns as a capture:

```
library/cis_ubuntu_linux_24_04_lts/2.0.0/evidence/ubuntuserver2404-root-20260731T122034Z/
  bare_capture.json   (1.2 MB: per-control raw output)
  _env.json           ({"hostname":"ubuntu2404","os_release":"PRETTY_NAME=\"Ubuntu 24.04 LTS\"",
                        "uname":"Linux ubuntu2404 6.8.0-124-generic...","euid":"0",
                        "captured_at":"2026-07-31T12:20:38Z","commands":722,"controls":333})
```

Inside `bare_capture.json["1.1.1.1"]["loaded_modules"]["lines"]`:

```json
["Module Size Used by","cpuid 12288 0","tls 155648 0",...]
```

— note cramfs is absent, which is exactly what the rule keys on.

---

## 13. evidence — does the capture conform to the declared contract?

**Purpose:** Validate every committed capture's raw output against the current command entries' evidence contracts. Distinguishes `unavailable` (explained by a probe_error) from `missing_silently` (script defect) and `stale` (script digest changed).

**Output — `stages/11_evidence/1.1.1.1.json`:**

```json
{"control_id":"1.1.1.1","capture":"library/.../ubuntuserver2404-root-20260731T122034Z",
 "facts":["modprobe_config","module_available","module_available_path","module_builtin","module_loaded"],
 "probe_errors":[],"observability":{...}}
```

**Feeds:** the observability matrix rule authors rely on; a control with no conformant capture is blocked for bundle.

---

## 14. validation — bind-linted validation rules

**Purpose:** Assemble the authored rules (`library/<version>/validation/<id>.json`) into per-control units and bind-lint every referenced fact against the command's declared contract — a rule binding an undeclared fact is blocked, never silently passed.

**Output — `stages/12_validation/1.1.1.1.json`:**

```json
{"control_id":"1.1.1.1","assessment":"automated","binds":["modprobe_config","module_available","module_loaded"],
 "comment":"Compliant when the module is not available at all ... OR ... not loaded AND modprobe config both
   blacklists it and installs it to /bin/false or /bin/true. inject a null never satisfies eq true...",
 "rules":[{"op":"or","of":[{"op":"field","fact":"module_available","field":"available","cmp":"eq","value":false},
   {"op":"and","of":[{"op":"field","fact":"module_loaded","field":"loaded","cmp":"eq","value":false},
                     {"op":"field","fact":"modprobe_config","field":"blacklisted","cmp":"eq","value":true}, ...]}]}]}
```

**Feeds:** fix (fix target state is derived from the rule) and bundle.

---

## 15. fix — remediation units with backup/rollback

**Purpose:** Assemble authored fix units; the `_manifest.json` partitions all 333 controls into auto/parameterized/manual/declined, each with a reason. Nothing ever runs here.

**Output — `stages/13_fix/1.1.1.1.json`:**

```json
{"control_id":"1.1.1.1","coverage":{"status":"auto","auto_fixable":true,"families":["module"],"reason":""},
 "safety_class":"approved_risk:kernel","enabled":true,"phases":["fix","rollback"],
 "commands":[{"name":"cis-1.1.1.1:fix","phase":"fix",
   "description":"Remediate 1.1.1.1... Backs up prior state under /var/backups/grc-fix/1.1.1.1;
      GRC_FIX_DRYRUN=1 prints intent and exits before mutating.",
   "command_text":"set -euo pipefail\nCID=\"1.1.1.1\"\nBK=\"/var/backups/grc-fix/1.1.1.1\"\nDRY=\"${GRC_FIX_DRYRUN:-0}\"...\n# --- backup pre-fix state..."}]}
```

Contrast the manual partition (`library/.../fix_commands/_manifest.json`):

```json
{"control_id":"1.1.1.11","status":"manual","auto_fixable":false,"reason":"rule contains a `manual` op — verdict can never be `pass`, cannot VM-gate","fix_unit":null}
```

**Feeds:** bundle.

---

## 16. bundle — the only merge point & release gate

**Purpose:** Merge every per-control artifact into the library, report coverage per stage, and refuse to emit while any control is blocked (unless `allow_partial`). No new derivation — assembly only.

**Output — `output/.../bundle/`:**

- `library.json`:

```json
{"summary":{"benchmark":{"catalog_key":"cis_ubuntu_linux_24_04_lts","version":"2.0.0"},
 "expected_controls":333,"bundled_controls":333,"coverage":{...},"authoritative_ratio":1.0}, "controls":[...]}
```

- `bundle_report.json`:

```json
{"coverage":{"split":{"complete":333,"skipped":0,"blocked":0,"expected":333},
  "risk":{"complete":333,...},"commands":{...},"validation":{...},"evidence":{...},
  "fix":{"complete":206,"skipped":127,...}},
 "authoritative_ratio":1.0,"blocked":{},"skipped":{"fix":["1.1.1.11","1.1.2.1.1",...]}}
```

- `control_ledger.json` — the per-control breadcrumb trail:

```json
{"1.1.1.1":{"split":{"status":"complete","research_sha256":"f31fe37a..."},
            "risk":{"status":"complete","severity":"medium","score":5.52},
            "commands":{"status":"complete","verdict":"automated"},
            "probe":{"status":"complete"},
            "evidence":{"status":"complete","capture":"library/.../ubuntuserver2404-root-20260731T122034Z"},
            "validation":{"status":"complete","assessment":"automated"},
            "fix":{"status":"complete","fix_status":"auto","safety_class":"approved_risk:kernel"}}}
```
