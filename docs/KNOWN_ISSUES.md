# Known issues

## 2026-10-10: the Evaluation Spine CLI fails without `httpx`, even when lineage is off

- What is wrong: `python -m app.evaluation_spine_cli` exits 1 with "unexpected ForgeMath authority failure" on a machine that has no `httpx`. The command has already written `forgemath_lane_evaluation_ref.contract.json` by then. The output file is valid. The exit code says the step failed.
- Root cause: `evaluate_calibration_report_file` writes the output, then runs `from app.lineage.spine_emit import emit_forgemath_lineage` (`app/services/evaluation_spine_authority.py`, line 243). That statement is outside any `try` block. Loading `app.lineage.spine_emit` first loads the package `app/lineage/__init__.py`. The package imports `app.lineage.emitter` at once. The emitter imports `forge_lineage_sdk` at module level (`emitter.py`, line 31). The SDK imports `httpx`. A missing `httpx` raises `ModuleNotFoundError` before `emit_forgemath_lineage` runs. The function is default-off and catches every exception, but the import happens first, so the guard never runs. `evaluate_calibration_report_file_or_raise` then wraps the error in a generic `ForgeMathAuthorityInputError` and hides the cause in the exception chain.
- Why it matters: `CLAUDE.md` and the module docstring say lineage emission is "default-off, and non-blocking". The code does not meet that for a missing optional dependency. A caller that checks the exit code cannot tell a finished authority step from a failed one. Forge_Command's self-healing plan runs this step in a chain (`plans/self_healing_calibration_chain_experiment.md` in Forge_Command, section 11).
- How it was found: an isolated run on 2026-10-10 used an interpreter without `httpx`. Steps before it exited 0. This step exited 1 and left a valid output file. The ForgeMath venv has `httpx`, so the run there would pass.
- Fix: made on 2026-10-10 in #17 (option 1). The import of `app.lineage.spine_emit` is now inside the guard. A missing `httpx` logs a warning and reports `lineage_missing`. The test `test_phase05_file_runner_succeeds_when_the_lineage_transport_cannot_be_imported` fails before the fix and passes after it. Option 2 (a lazy package `__init__`) was not made; other importers of `app.lineage` still load the SDK at once.
- Status: fixed for the Evaluation Spine file runner. Scope: this CLI path only. The FastAPI routes were not checked and stay open.

## 2026-10-02: `doc/system/` holds overlapping part sets

- What is wrong: `doc/system/` has 22 parts. Several share a number prefix (`01-architecture.md`, `01-overview-philosophy.md`; `10-scope.md`, `10-ecosystem-integration.md`), and some describe a frontend and design system that a backend service does not have. `BUILD.sh` assembles all of them into `doc/MATSYSTEM.md`.
- Root cause: unknown. Likely template parts left beside authored parts.
- Fix: not made. Review each part and remove or rewrite the stale ones.
- Status: open.
