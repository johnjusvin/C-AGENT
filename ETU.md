```markdown
# Benchmark Processing Pipeline

1. **layout** — Creates the empty folder skeleton for that benchmark inside `standards/` and `output/`. No content yet, just the directory structure.

2. **inventory** — Scans the PDF and lists every section/chapter with its page number. A map of "what's in the book."

3. **content** — Extracts the actual control text from the PDF (titles, requirements, levels L1/L2, profiles) for every section. The raw material of the benchmark.

4. **appendix** — Pulls the tech references: `DEFINE`/`CONFIGURE` service rules, firewall port requirements, etc. The "implementation details" part of the PDF.

5. **mappings** — Builds the two mapping tables:
   - Section rule → related controls
   - Language → words that translate to rule behavior

   This is how the benchmark language becomes machine directives.

6. **finalize** — After **all stages above**, merges everything from the sections into their final form:
   - Clean rule text
   - Enriched titles
   - Profile tiers (`Server`/`Workstation`)
   - "STIG entire benchmark" flag
   - Benchmark ID

7. **refine** — Filters the finalized controls:
   - Keeps only scored/automated controls applicable per profile
   - Drops duplicate sections
   - Drops profile tags that don't apply

   Produces the final "what we actually build commands for" list.

8. **split** — Takes the final control list and cuts each control into its own JSON artifact. One file per control, e.g. `1.1.1.json`. This is the per-control **control definition** everything downstream works from.

9. **risk** — Computes each control's risk factor (`IR`, `3`), a deterministic number from the rule text and severity. Purely math, no agents, no AI.

10. **research** — **Only for "research mode" standards** such as Ubuntu 22.04. For each control, asks sub-agents to find:
    - The exact command that audits that setting
    - The full evidence the benchmark requires

    Runs with a memory and writes the result as a `library/research/` record.

    Bare/no-mode standards skip this stage.

11. **commands** — After the authored library is committed (`library/.../bare_commands/*.json`), this stage runs the correct rule per-control and **records the output** into:

    `output/.../stages/09_commands/`

    In research mode:
    - Builds the command from the research record's script
    - Binds the emitter prelude
    - Produces the artifact
    - Lints it as a "pinned block"

12. **probe** — Wraps the commands into a self-contained handoff bundle:

    - `probe_bundle.json`
    - `run_probe.py`
    - `HANDOFF.md`
    - `manifest.json`

    A human can carry this bundle to the VM, run it read-only, and bring back real evidence.

13. **evidence** — Takes the VM's raw capture and checks each control's output against its contract:
    - Digest match
    - Script exit `0`
    - No timeout/truncation
    - Typed JSON fields
    - No silently missing facts

    Certified = **complete**. Anything doubtful = **blocked**.

    Produces:
    - Per-control evidence units
    - Observability matrix

14. **validation** — Scores each control as:
    - `PASS`
    - `FAIL`
    - `ERROR`
    - `SKIP` / `N/A`

    Uses the certified facts from stage 13 against each control's rule.

    **Never guesses:** "couldn't look" is `ERROR`, not `FAIL`.

    Emits:
    - Validation ledger
    - Per-control verdict records

15. **fix** — Assembles authored remediation scripts into per-control fix units.

    Each fix includes:
    - Dry-run
    - Rollback
    - Backup

    This is the **fix library** — packaged and safe, but **never executed by the pipeline**.

16. **bundle** — The grand finale. Merges:
    - Command library
    - Validation
    - Fixes
    - Policies
    - Frontend
    - MSSQL master DB

    Into the final, deployable artifacts, such as:

    `fix_policy/pulse_level_1_server_fix.json`

    Everything the GRC platform can consume as one deliverable.

---

## One-Sentence Summary

> **Extract the PDF → cut it into controls → decide risk → (research) → build commands → run them read-only on a real VM → certify evidence → score pass/fail → package safe fixes → ship the whole thing as one bundle.**
```
