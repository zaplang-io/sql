# sql workspace conventions (zaplang-io/sql)

This is an official non-core Zap module. Target: parity with the popular Python
equivalents — [SQLAlchemy](https://docs.sqlalchemy.org/) Core + ORM, with
[Alembic](https://alembic.sqlalchemy.org/) as the model for the migrations layer.
The package ships in three phases: **Core → ORM → migrations** (see the epic).

## Language-first (non-negotiable)
- All language or stdlib bugs discovered while building this module MUST be reported in `zaplang-io/zap` FIRST, and no workaround is permitted. Keep the dependent work blocked or draft and report the dependency; never build the library around a core defect.

## Python parity & signatures
- Match the equivalent `SQLAlchemy` (Core, then ORM) and `Alembic` API surface and naming as closely as the language allows. Where SQLAlchemy's Core and ORM surfaces differ, Core is the Phase 1 reference.
- Where a function/method signature is ambiguous or there is any confusion about argument shape, defaults, or keyword-vs-positional, do NOT just pick one: present the options to the maintainer with concrete examples of what SQLAlchemy/Alembic do, and let the maintainer decide. Record the chosen design in `DECISIONS.md`.

## Security: parameterized queries (non-negotiable)
- Every value that reaches the database MUST travel as a bound parameter. Do NOT add any API that interpolates user data into SQL text. The expression builder's value positions take bound parameters only.
- Identifiers (table/column names) from untrusted input must be validated or quoted by the dialect, never concatenated raw. Follow the upstream [official-package guidelines](https://github.com/zaplang-io/zap/blob/main/docs/OFFICIAL_PACKAGE_GUIDELINES.md): parse driver responses defensively, use overflow-safe length arithmetic, and never echo connection secrets (DSNs, passwords) in error messages.

## Tests
- All test cases live in the `tests/` folder and use the `zap test` methodology (`zap test tests/`).
- Every PR MUST include extensive positive AND negative test cases — treat tests as a walled garden and as intent documentation, not a minimal smoke check. Cover every value type, error paths, malformed driver responses, and — critically — assert that values are bound, never interpolated (include an injection-attempt string that must round-trip as data).
- When fixing a bug, write the failing test first, then fix.

## CI
- CI runs against `ghcr.io/zaplang-io/zap:latest` and MUST continue to pin `zap:latest` until this module reaches a proper major (1.0) version release; only then pin a released toolchain version.

## Issues & PRs
- Keep issues concrete and testable with detailed bodies (repro or intended syntax, Python equivalent, references).
- Multi-issue features get an Epic with trackable sub-issues. Individual issues are 'M' sized; do not split an epic into unnecessary 'S' issues. The epic is phased; only the current phase's sub-issues are filed.
- One issue, one PR. Rebase onto the latest default branch before raising or updating a PR.
- Documentation (README, DECISIONS.md, CHANGELOG) stays current with every PR.

## Attribution
- No agent/model attribution in commits, PR titles/bodies, or issue bodies/titles — maintainer's name only. PR and issue comments may carry a "Reviewed by …" / "… input on …" attribution; nothing else does.
