# Database Correctness and Scale

Data-layer conventions must cover concurrency, scale, and integrity. Claude otherwise omits transactions, under-specifies referential rules, denormalizes casually, ignores indexing, and writes unscalable queries.

Database mistakes affect correctness, operations, and performance.

Treat denormalization as a database design decision, not a local implementation shortcut. Duplicating fields into a convenient table, document, or cache creates synchronization, staleness, invalidation, migration, and source-of-truth problems.

Keep derived logic in the API or application layer by default. Code is much easier to refactor than a schema or persisted data model. Moving a calculation, projection, or response-shaping rule is cheap compared with changing stored data, backfilling rows, maintaining dual writes, or repairing corrupted state. Local implementation convenience does not justify denormalization.

The convention should be explicit: Claude must not introduce denormalized storage, duplicate persisted fields, summary tables, materialized views, cached database columns, or alternate persisted representations of the same fact without asking. If denormalization is genuinely needed for scale, reporting, or query latency, the design should name:

- the canonical source of truth
- the reason normalization is insufficient
- how duplicated data is updated
- what happens when updates fail halfway through
- how stale reads are tolerated or prevented
- how backfills, migrations, and repair jobs work
- which indexes and query paths justify the extra complexity

If those answers are not clear, Claude should keep the model normalized and ask for a database-design decision before proceeding.

The convention should push Claude to assume the database matters:

- use transactions whenever a unit of work spans multiple writes, read-modify-write steps, or any sequence that must preserve an invariant
- treat transactions as the default integrity mechanism for multi-step database changes, not as an advanced optional feature
- preserve referential soundness with appropriate constraints, foreign keys, cascades, and deletion/update rules according to the project's data model
- avoid application-level cleanups or "best effort" sequencing when a transaction or constraint can make the invariant durable
- think about query shape, cardinality, and access patterns before writing database code that will run in loops, on hot paths, or behind common product flows
- design queries to use existing indexes where possible instead of assuming the database will sort it out
- add obviously required indexes as part of the schema change when introducing new lookup, join, ordering, uniqueness, or filtering patterns
- consider query plans and index selectivity when the data volume could plausibly matter
- ask about consistency, isolation, migration, scale expectations, and fundamental schema choices when they are not clear from the existing system

Exact choices depend on the stack and workload. Unless told otherwise, optimize for transactional correctness, referential integrity, and reasonable performance rather than throwaway implementation.

Returning the right rows in development is insufficient if a change lacks transactional soundness, leaves orphaned data, creates conflicting facts, misses indexes, or degrades under load.
