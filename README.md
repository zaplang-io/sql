# sql

SQL toolkit, ORM, and schema migrations for [Zap](https://github.com/zaplang-io/zap) — an official **non-core** package.

- Tracking issue: [zaplang-io/zap#323](https://github.com/zaplang-io/zap/issues/323)
- Contributor & agent conventions: [AGENTS.md](AGENTS.md)
- Design decisions: [DECISIONS.md](DECISIONS.md)

This is the entry point to the Zap database ecosystem. It is large and strategic, so it ships in **three phases**. The [Tier-1 epic](https://github.com/zaplang-io/sql/issues/2) tracks the whole plan; the first-slice sub-issues ([#3](https://github.com/zaplang-io/sql/issues/3)–[#6](https://github.com/zaplang-io/sql/issues/6)) scope **Phase 1 (Core) only**. No library code has landed yet.

## Goals

A typed Zap SQL toolkit whose public API mirrors Python's [**SQLAlchemy**](https://docs.sqlalchemy.org/) (Core + ORM) as closely as Zap's static typing allows, with [**Alembic**](https://alembic.sqlalchemy.org/) as the model for the migrations layer. API-parity with the popular Python library is a project rule — see [AGENTS.md](AGENTS.md).

Core is pure-Zap and dialect-neutral; it talks to databases through per-engine drivers (a DBAPI-style driver abstraction, with the Postgres driver as the first concrete backend). **Every value that reaches the database travels as a bound parameter** — the toolkit never builds SQL by string interpolation of user data. That is the package's central security property; see the trust boundary below and in [DECISIONS.md](DECISIONS.md).

## Phasing

The charter (SQLAlchemy Core + ORM + Alembic-style migrations) is too large for one slice, so it is delivered in order:

- **Phase 1 — Core** *(first slice; sub-issues [#3](https://github.com/zaplang-io/sql/issues/3)–[#6](https://github.com/zaplang-io/sql/issues/6))*
  - Engine / connection + a DBAPI-style driver abstraction.
  - SQL expression language / query builder — `select` / `insert` / `update` / `delete`, `where`, joins, ordering.
  - Result handling — typed rows, column access, type coercion.
- **Phase 2 — ORM** *(named here, not yet filed)*
  - Declarative models, `Session` / unit-of-work, identity map, relationships and loading strategies.
- **Phase 3 — Migrations** *(named here, not yet filed)*
  - Alembic-style migration scripts, autogenerate from model diffs, up/down revisions, a version table.

Phase 2 and Phase 3 sub-issues are **deferred** — enumerated in the epic, filed only when Phase 1 is in hand.

## Planned public surface (Phase 1 — Core)

The exact signatures (keyword vs positional, defaults, return shapes) are **not final**; this sketch is pinned in step with epic [#2](https://github.com/zaplang-io/sql/issues/2) / surface sub-issue [#3](https://github.com/zaplang-io/sql/issues/3). The SQLAlchemy-Core equivalent is named for each piece.

```zap
import sql

# Engine + connection (SQLAlchemy: create_engine / Engine.connect)
engine = sql.create_engine("postgresql://user@host/db")

# Describe tables (SQLAlchemy: Table / Column / MetaData)
users = sql.table("users",
    sql.column("id", sql.Integer),
    sql.column("name", sql.Text),
    sql.column("age", sql.Integer),
)

# Build a query — Core expression language, not ORM
# (SQLAlchemy: select(...).where(...).order_by(...))
query = (sql.select(users.c.id, users.c.name)
    .where(users.c.age >= 18)
    .order_by(users.c.name))

# Execute with BOUND PARAMETERS — never string interpolation
with engine.connect() as conn:
    result = conn.execute(query)
    for row in result:
        print(row["id"], row["name"])   # typed column access

# Writes (SQLAlchemy: insert()/update()/delete())
conn.execute(sql.insert(users).values(name="Ada", age=36))
```

`age >= 18` builds a bound-parameter predicate; the literal `18` is sent as a
parameter, not spliced into the SQL text. Identifiers (table and column names)
are quoted by the dialect. Result rows expose columns by name and by position,
coerced to Zap types through the column's declared SQL type.

## Trust boundary

Query **structure** (which tables, columns, and operators) is author-controlled;
query **values** are untrusted and always travel as bound parameters, so user
data can never alter SQL structure (no injection). Identifiers that originate
from untrusted input must be validated or quoted by the dialect, never
concatenated raw. Rows returned by the driver are parsed defensively into typed
values. See [DECISIONS.md](DECISIONS.md) and the upstream
[official-package guidelines](https://github.com/zaplang-io/zap/blob/main/docs/OFFICIAL_PACKAGE_GUIDELINES.md).

## Build and test

Run the native `zap test` suite against a temporary consumer that installs this
checkout as a package:

```
sh tools/test.sh
```

While the package is at the scaffolding stage (no `.zap` sources yet) the harness
reports that and exits green; it runs the full fmt-check + consumer-install +
`zap test` flow automatically once `src/` and `tests/` contain code.

GitHub Actions runs the suite on pull requests and pushes inside the
`ghcr.io/zaplang-io/zap:latest` container. CI stays pinned to `zap:latest`
until this module reaches a 1.0 release.

## Development

Read **[AGENTS.md](AGENTS.md)** before contributing — it covers the
SQLAlchemy/Alembic-parity rule, the parameterized-query security rule, the
one-issue-one-PR flow, how design decisions are recorded in
**[DECISIONS.md](DECISIONS.md)**, and the rule for reporting core Zap
language/stdlib defects upstream in `zaplang-io/zap` (rather than working around
them here).
