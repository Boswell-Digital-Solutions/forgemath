# Known issues

## 2026-10-30: `FORGEMATH_HOST` and `FORGEMATH_PORT` do not set the bind address (KI-MAT-20261030-001)

- Classification: documentation drift. This is not a security vulnerability.
- How it was found: the BDS-XMACH-CALLER-BOUNDARY-001 WP00 inventory (forge PR #325). Re-verified on 2026-10-30 by source reading on `origin/main` `0a808a4`. No service was run.
- Documentation claim: `doc/system/05-configuration.md:8-9` lists `FORGEMATH_HOST` (default `127.0.0.1`) and `FORGEMATH_PORT` (default `8006`) as environment variables, read by `app/config.py`. It does not say that the service binds to them, and it does not say that it does not. A reader takes them as the bind settings.
- What is observed: `app/config.py:10-26` reads the two values and `validate_config` checks them (host not empty, port an integer from 1 to 65535). `app/main.py:11-22` calls `validate_config` at startup and does nothing else with them. No in-repo launcher passes them to uvicorn: there is no `__main__` block, Dockerfile, Procfile or script that does. The README command (`README.md:60`) passes `--host 127.0.0.1 --port 8006` as literal values. The bind address is therefore the uvicorn command line, or uvicorn's default (`127.0.0.1:8000`) when no flags are given. A change to `FORGEMATH_PORT` alone does not move the port. The README values equal the defaults today, so the documented default port is reached only by that command.
- Proposed correction (doc only): in `doc/system/05-configuration.md`, state that `FORGEMATH_HOST` and `FORGEMATH_PORT` are validated only, and that the bind address comes from the uvicorn `--host` and `--port` flags (`README.md:60`). Then run `bash doc/system/BUILD.sh`. A change that makes the service bind to these values is a code change and needs separate authority. This entry does not change `doc/system`.
- ID note: forgemath has no row in the forge `doc/PREFIX_REGISTRY.md`. This ID uses the repository's own designation `MAT` (`doc/MATSYSTEM.md`).
- Status: Open — recorded, unresolved until the correction is verified.

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
