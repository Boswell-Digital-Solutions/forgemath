# Known issues

## 2026-10-02: `doc/system/` holds overlapping part sets

- What is wrong: `doc/system/` has 22 parts. Several share a number prefix (`01-architecture.md`, `01-overview-philosophy.md`; `10-scope.md`, `10-ecosystem-integration.md`), and some describe a frontend and design system that a backend service does not have. `BUILD.sh` assembles all of them into `doc/MATSYSTEM.md`.
- Root cause: unknown. Likely template parts left beside authored parts.
- Fix: not made. Review each part and remove or rewrite the stale ones.
- Status: open.
