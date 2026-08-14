# Database Correctness and Scale

Database work deserves explicit protection too. Claude often writes data-layer code as if it were manipulating a toy dataset with no concurrency, no scale, and no integrity requirements. It will forget transactions, under-specify referential rules, casually denormalize data, ignore indexing, and write query shapes that are only acceptable if the system never grows.

That default is dangerous because database mistakes are not just style mistakes. They become correctness problems, operational problems, and performance problems.

Denormalization needs special caution. Claude has a strong bias toward duplicating fields into the locally convenient table, document, or cached shape because it makes the current query or screen easier to implement. That is usually the wrong default. Denormalization is a fundamental database design decision, not a local implementation trick. It creates synchronization problems, stale data risks, invalidation complexity, migration complexity, and ambiguity about which copy is authoritative.

Claude should usually be pushed toward keeping derived logic in the API or application-code layer first. Code is several orders of magnitude easier to refactor than a database schema or persisted data model. Moving a calculation, projection, or response-shaping rule between functions is cheap compared with changing stored data, backfilling rows, maintaining dual-write paths, or repairing corrupted historical state. If the only reason to denormalize is that it makes the current implementation look simpler, the design is probably wrong.

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

The exact answers depend on the stack and workload, so the guide should stay generic. The point is that Claude should not assume it is building a throwaway internal toy unless the project clearly says so. It should be pushed toward transactional correctness, referential integrity, and reasonable performance by default.

This is another area where "make it work" is not enough. A change that returns the right rows in development can still be wrong if it is not transactionally sound, leaves orphaned data behind, creates conflicting copies of the same fact, misses necessary indexes, or degrades badly under load.

