# Database Correctness and Scale

Data conventions must preserve integrity under concurrency and realistic volume. Keep the model normalised by default: persisted duplication creates synchronization, staleness, migration, and repair obligations.

## Do

- Use transactions for multi-write and read-modify-write operations that must preserve an invariant.
- Use appropriate constraints, foreign keys, uniqueness, and update/deletion rules.
- Inspect query shape, cardinality, and access patterns, especially loops and hot paths.
- Use existing indexes and add required indexes with new lookup, join, ordering, uniqueness, or filtering patterns.
- Check query plans and selectivity where volume matters.
- Keep derived calculations and response shaping in application/API code by default.
- Ask about unclear consistency, isolation, migration, scale, or fundamental schema decisions.
- Assume tables / datasets will grow to be extremely large.

## Don't

- Replace transactional guarantees with best-effort sequencing or application cleanup.
- Assume development-sized data proves query performance.
- Introduce duplicate persisted fields, summary tables, materialized views, cached columns, or alternate representations without approval.
- Denormalise for local implementation convenience.

## Before approving denormalisation

The design must identify:

1. The canonical source and why normalisation is insufficient.
2. Measured scale, reporting, or latency needs and the relevant query/index paths.
3. How copies update, including partial failures and consistency guarantees.
4. How stale reads are tolerated or prevented.
5. Backfill, migration, repair, and invalidation procedures.

If these decisions are unclear, keep the model normalised and seek a database-design decision.
