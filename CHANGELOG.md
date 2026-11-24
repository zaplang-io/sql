# Changelog

## Unreleased

- Initialize the package scaffold: manifest, README with the pinned Phase 1
  (Core) public surface and the Core → ORM → migrations phasing, `DECISIONS.md`,
  `AGENTS.md`, the `zap test` harness, and CI pinned to
  `ghcr.io/zaplang-io/zap:latest`. No library code yet — the engine/driver, SQL
  expression builder, and result handling are tracked in the Tier-1 epic
  [#2](https://github.com/zaplang-io/sql/issues/2) and its Phase 1 first-slice
  sub-issues [#3](https://github.com/zaplang-io/sql/issues/3)–[#6](https://github.com/zaplang-io/sql/issues/6).
  ORM and migrations are deferred to later phases.
