# Benchmark Pipeline Steps

| Step | File / command |
|---|---|
| 1. Discover | `python -m benchmark_pipeline discover --pdf standards/<key>/<version>/source/pdf/<file>.pdf` |
| 2. Extract | `python -m benchmark_pipeline run --standard <key> --version <version> --to-stage risk` |
| 3. Author commands | `edit library/<key>/<version>/bare_commands/<id>.json` + parser in `src/benchmark_pipeline/bare_parsers/` |
| 4. Capture | `vmrun revertToSnapshot ...` then `python tools/bare_probe.py ...` + `python tools/materialise_facts.py --standard <key> --version <version>` |
| 5. Contract | `python -m benchmark_pipeline run ... --from-stage commands --to-stage commands --force` |
| 6. Validation | `python tools/plan_validation.py ...`, `check_validation.py`, `gate_validation_bare.py` |
| 7. Value gate | `tools/gate_validation_bare.py`, `tools/check_fact_names.py` |
| 8. Fix + bundle | `python tools/build_fix_commands.py` then the four `build_*_policy` / `build_*_benchmark` / `build_card_data` / `explorer` / `master_db` tools |
| 9. Landed | `python tools/land_standard.py --standard <key> --version <version>` |
