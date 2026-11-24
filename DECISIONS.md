# sql package decisions

Design decisions for the Zap `sql` library, keyed to issues. Each records a
choice that could not be read straight off Python's
[SQLAlchemy](https://docs.sqlalchemy.org/) (Core + ORM) or
[Alembic](https://alembic.sqlalchemy.org/), with the reference it follows or
departs from. Signature and surface ambiguities are resolved with the maintainer
(with Python examples) before being recorded here.

## Parity target: SQLAlchemy (Core + ORM) + Alembic

The package tracks Python's [**SQLAlchemy**](https://docs.sqlalchemy.org/) as its
primary API reference — **Core** (engine, connection, SQL expression language) in
Phase 1, the **ORM** (declarative models, `Session`, relationships) in Phase 2 —
and [**Alembic**](https://alembic.sqlalchemy.org/) as the model for the Phase 3
migrations layer. Where SQLAlchemy's Core and ORM surfaces differ, the Core
surface is the Phase 1 reference. Deviations are recorded below, keyed to the
issue that introduces them.

## Phasing (owned by epic [#1](https://github.com/zaplang-io/sql/issues/1))

The charter is delivered in three phases — **Core** → **ORM** → **migrations**.
The first slice is Phase 1 (Core) only: engine/connection + driver abstraction
([#3](https://github.com/zaplang-io/sql/issues/3)), the SQL expression/query
builder ([#4](https://github.com/zaplang-io/sql/issues/4)), and result handling
([#5](https://github.com/zaplang-io/sql/issues/5)), all pinned by a docs-first
surface spec ([#2](https://github.com/zaplang-io/sql/issues/2)). ORM and
migrations are named in the epic but not filed until Phase 1 is in hand.

## Forced divergences from Python (static typing)

Zap is statically typed, so the surface cannot match SQLAlchemy 1:1. The
following are decided by the language and recorded now; a user-facing
"differences from SQLAlchemy" note lands with the surface
([#2](https://github.com/zaplang-io/sql/issues/2)):

- **No untyped `Any` for cell values.** A result cell is modeled over a closed
  `SqlValue` sum type (null / integer / float / text / blob / bool / the date-time
  forms / decimal) rather than Python's dynamic object. Typed access returns
  `SqlValue`; there is no bare `Any`.
- **Parameterized values only — no string interpolation.** Every value sent to
  the database is a bound parameter. The toolkit exposes no API that splices user
  data into SQL text. This is a security invariant, not a convenience default;
  see the trust boundary. SQLAlchemy achieves the same by default, but in Zap it
  is enforced by the type of the expression builder (a value position takes a
  bound parameter, never a raw SQL fragment).
- **Typed driver interface.** Python's drivers follow the dynamic
  [PEP 249 DBAPI](https://peps.python.org/pep-0249/); SQLAlchemy wraps them in
  dialects. Zap models the driver as a typed interface (connect, prepare, execute,
  fetch, parameter binding, type mapping) that concrete backends implement. The
  Postgres driver is the first backend.
- **Date-time / numeric mapping.** SQL date-time columns map onto Zap's `time`
  stdlib types, and SQL numeric/decimal onto Zap's numeric model (64-bit signed
  integers, IEEE-754 binary64 floats, an explicit decimal type), recorded rather
  than hidden. The exact mapping is pinned in
  [#2](https://github.com/zaplang-io/sql/issues/2) and
  [#5](https://github.com/zaplang-io/sql/issues/5).
- **Deferred keyword-arg ergonomics.** SQLAlchemy's deep keyword surface
  (`create_engine(...)` pool/echo options, `select(...).options(...)`, etc.) is
  trimmed to the minimum in the first slice; which options become keyword args
  later is a separate decision.

## Open decisions (owned by the surface sub-issue [#2](https://github.com/zaplang-io/sql/issues/2))

- **How column operators build the predicate tree.** SQLAlchemy writes
  `users.c.age >= 18` and relies on Python operator overloading so `>=` returns a
  SQL expression object rather than a bool. The first design question is whether
  Zap's operator surface lets a `Column` type overload comparison operators to
  return an expression node (the ergonomic, parity-preserving form), or whether
  the builder must use method calls (`users.c.age.ge(18)`). **Recommended:**
  pursue operator overloading for parity; fall back to methods only if the
  language cannot express it. If it cannot, that is a core language gap — it is
  filed in `zaplang-io/zap`, linked here, and the query-builder sub-issue
  ([#4](https://github.com/zaplang-io/sql/issues/4)) blocks on it. No workaround
  is built around the gap (language-first rule).
- **Result-row access shape.** By name (`row["id"]`), by position (`row[0]`),
  and/or by a typed accessor. SQLAlchemy offers mapping-style and tuple-style
  access on `Row`. #2 confirms which Zap can express (indexed get overload on a
  user `Row` type) and #5 implements it; an un-expressible form becomes a core
  gap filed upstream.
- **Connection lifecycle / context manager.** Whether `engine.connect()` returns
  a value usable in a `with` block (scoped close), matching SQLAlchemy's
  `with engine.connect() as conn:`. #2/#3 confirm against the language's
  context-manager support.

## Trust boundary

Query **structure** (tables, columns, operators, clause shape) is
author-controlled; query **values** are untrusted and always travel as bound
parameters, so user data can never change SQL structure (no injection). This is
the package's central security property. Identifiers (table/column names) that
originate from untrusted input must be validated or quoted by the dialect, never
concatenated raw into SQL text. Rows returned by a driver are parsed defensively
into typed `SqlValue`s with overflow-safe length/size handling, per the upstream
[official-package guidelines](https://github.com/zaplang-io/zap/blob/main/docs/OFFICIAL_PACKAGE_GUIDELINES.md).
Driver-level connection secrets (DSNs, passwords) are treated as sensitive and
never echoed in error messages.
