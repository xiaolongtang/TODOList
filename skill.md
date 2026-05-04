---
name: db-migration-governance
description: Use this skill when designing, implementing, or reviewing database migrations involving schema changes, new tables, table splits, table merges, data backfills, constraints, indexes, or destructive database changes.
---

# Database Migration Governance Skill

Use this skill for any task involving database schema changes, table creation, table splitting, table merging, data transformation, backfills, constraints, indexes, or migration SQL.

## Primary objective

Produce safe, auditable, evidence-based database migrations. The live database schema inspected through MCP is the source of truth.

## Non-negotiable rules

1. Do not generate migration SQL until actual schema discovery is complete.
2. Do not infer the schema only from application code.
3. Use database MCP tools before proposing SQL.
4. Record what was discovered from MCP.
5. Separate DDL, data migration, validation, application cutover, and cleanup.
6. Avoid destructive changes in the same migration as additive changes.
7. Prefer expand-and-contract migrations.
8. Include validation queries for every migration.
9. Include rollback, remediation, or forward-fix strategy.
10. Call out uncertainty instead of guessing.

## Phase 0: Clarify migration intent

Identify:

- Business goal
- Target entities
- Existing tables involved
- New tables or columns needed
- Whether data will be split, merged, deduplicated, copied, moved, or transformed
- Expected data volume
- Whether zero-downtime or backward compatibility is required
- Migration framework used by the repo
- Target database engine and version

If any item is missing but not blocking, make an explicit assumption.

## Phase 1: Discover actual database schema through MCP

Before SQL generation, use MCP tools to inspect:

- tables
- columns and data types
- nullable flags
- default values
- primary keys
- foreign keys
- unique constraints
- check constraints
- indexes
- triggers
- views
- sequences / identity columns
- enum types or domain types if relevant
- migration history table
- approximate row counts for affected tables
- existing data quality problems relevant to the change

Also inspect relevant application code only after MCP schema discovery.

Output a section called `MCP schema evidence` containing the discovered facts.

## Phase 2: Compare DB schema with code assumptions

Compare:

- actual DB schema
- ORM/entity/model definitions
- existing migration files
- repository conventions
- query code and repository/service layer usage

If there is a mismatch, explicitly mark it as:

- `DB/code mismatch`
- `migration history mismatch`
- `unknown`

Do not silently resolve conflicts.

## Phase 3: Design the migration

Prefer this order:

1. Add new tables/columns/indexes in a backward-compatible way.
2. Backfill data using deterministic mapping.
3. Add constraints only after data is valid.
4. Update application code to read/write the new structure.
5. Run validation.
6. Remove old columns/tables in a later migration.

For table split/merge operations, provide a mapping table:

| Source table | Source column | Target table | Target column | Transformation | Null/default handling | Conflict handling |
|---|---|---|---|---|---|---|

## Phase 4: SQL generation rules

When asked to generate SQL:

- Generate SQL for the target database engine only.
- Use the repository's migration framework conventions.
- Keep schema migration and data migration separate unless the project convention requires otherwise.
- Include comments explaining non-obvious operations.
- Include validation queries.
- Include rollback SQL when safe.
- If rollback is unsafe, provide a forward-fix/remediation plan.
- Avoid `SELECT *`.
- Avoid unbounded updates for large tables; propose batch strategy.
- Mention lock risks for `ALTER TABLE`, index creation, constraint validation, and large updates.
- Do not drop old data in the first migration unless explicitly approved.

## Phase 5: Review checklist

Before finalizing, verify:

- Actual schema was inspected through MCP.
- All affected tables and columns are listed.
- Data mapping is explicit.
- Nullable/default behavior is defined.
- Foreign keys and indexes are accounted for.
- Existing duplicate or invalid data is handled.
- Migration order is backward-compatible.
- Validation queries are included.
- Rollback/remediation is included.
- SQL matches the repo's migration framework.
- Application code changes are identified but not mixed blindly with SQL.

## Output format for planning

Use this structure:

1. Summary
2. MCP schema evidence
3. Code evidence
4. Current schema
5. Target schema
6. Data mapping
7. Migration sequence
8. Validation queries
9. Rollback/remediation
10. Risks
11. Open questions
12. Do not proceed to final SQL until approval
