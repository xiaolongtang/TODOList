---

name: find-derived-lineage
description: Find a small set of derived lineage matches between source and target attributes by querying the database through the jdbc-explorer MCP tool. Use this skill when the user asks to find derived lineage, derived attribute matches, same-name source-target attribute lineage, or demo-ready lineage matches.
argument-hint: "[optional limit, default 10]"
---------------------------------------------

# Find Derived Lineage

## Purpose

Use this skill to find demo-ready derived lineage matches from the database.

The goal is to query existing lineage metadata and return a small number of clean, easy-to-explain matches where:

* The lineage is derived.
* The lineage record is active.
* Both source and target data models are physical data models.
* In business terminology, both source and target data models are attributes.
* Both source and target attributes are active.
* Both source and target attribute names are present.
* The source attribute name and target attribute name are the same.
* The final answer uses business terminology, not database table terminology.

Do not claim that the language model mathematically inferred or generated the match. It is acceptable to say: "I found the following derived lineage matches."

## Required Tool

Use the `jdbc-explorer` MCP tool to inspect and query the database.

If the exact tool name is available as a tool reference, use it directly. If not, choose the available JDBC/database exploration tool whose server name is `jdbc-explorer`.

## Business Terminology Mapping

Use this terminology in the final answer:

| Database / Technical Term                | Business Term to Use |
| ---------------------------------------- | -------------------- |
| physical data model                      | attribute            |
| physical model field name / `filed_name` | attribute name       |
| transformation                           | lineage              |
| `model_owner_name` / owner name          | system name          |
| model owner                              | system               |
| source data model                        | source attribute     |
| target data model                        | target attribute     |

Important: the database column is named `filed_name`, even though the business term is "attribute name".

## Relevant Tables

### `data_model`

Important columns:

* `id`: primary key.
* `model_id`: unique identifier for each data model.
* `current_indicator`: `1` means active; `0` means inactive or soft-deleted.
* `filed_name`: attribute name.
* `data_model_type`: either `LOGICAL` or `PHYSICAL`.
* `indexed_model_id`: for logical models, stores the model id of the automatically created index physical model.
* `owner_id`: joins to `model_owner.owner_id`.

Rules:

* A physical data model is called an attribute.
* Only use records where `data_model_type = 'PHYSICAL'`.
* Only use records where `current_indicator = 1`.
* Only use records where `filed_name` is not null and not blank.

### `transformation`

Important columns:

* `id`: primary key.
* `transformation_id`: unique lineage identifier.
* `current_indicator`: `1` means active; `0` means inactive.
* `derivation_type`: use only `DERIVED`.

Rules:

* A transformation is called lineage.
* Only use records where `current_indicator = 1`.
* Only use records where `derivation_type = 'DERIVED'`.

### `transformation_model_xref`

Important columns:

* `id`: primary key.
* `transformation_id`: references the transformation record.
* `transformation_type`: either `SOURCE` or `TARGET`.
* `data_model_id`: references `data_model.model_id`.

Rules:

* Source rows identify source attributes.
* Target rows identify target attributes.
* Join source and target rows through the same active derived lineage.

### `model_owner`

Important columns:

* `owner_id`: joins to `data_model.owner_id`.
* `model_owner_name` or `owner_name`: business term is system name.

If the schema uses `owner_name` instead of `model_owner_name`, use the actual available column and alias it as system name.

## Query Procedure

1. Use `jdbc-explorer` to confirm the available schema if needed.
2. Query active derived lineage records from `transformation`.
3. Join to `transformation_model_xref` twice:

   * one join for source rows where `transformation_type = 'SOURCE'`
   * one join for target rows where `transformation_type = 'TARGET'`
4. Join source and target xref rows to `data_model` by `data_model.model_id`.
5. Join source and target data models to `model_owner` by `owner_id`.
6. Filter both source and target data models:

   * `current_indicator = 1`
   * `data_model_type = 'PHYSICAL'`
   * `filed_name IS NOT NULL`
   * trimmed `filed_name` is not empty
7. Keep only rows where the source and target attribute names are the same.
8. Prefer source and target attributes from different systems when enough rows exist.
9. Return at most 10 rows unless the user asks for a different number.
10. Do not expose internal primary keys unless the user asks.

## SQL Template

Use the actual SQL dialect supported by the database. The following template assumes a SQL dialect that supports `LIMIT`.

If `transformation_model_xref.transformation_id` references `transformation.id`, use this version:

```sql
SELECT DISTINCT
    source_owner.model_owner_name AS source_system_name,
    source_dm.filed_name AS source_attribute_name,
    target_owner.model_owner_name AS target_system_name,
    target_dm.filed_name AS target_attribute_name
FROM transformation t
JOIN transformation_model_xref source_xref
    ON source_xref.transformation_id = t.id
   AND UPPER(source_xref.transformation_type) = 'SOURCE'
JOIN transformation_model_xref target_xref
    ON target_xref.transformation_id = t.id
   AND UPPER(target_xref.transformation_type) = 'TARGET'
JOIN data_model source_dm
    ON source_dm.model_id = source_xref.data_model_id
JOIN data_model target_dm
    ON target_dm.model_id = target_xref.data_model_id
LEFT JOIN model_owner source_owner
    ON source_owner.owner_id = source_dm.owner_id
LEFT JOIN model_owner target_owner
    ON target_owner.owner_id = target_dm.owner_id
WHERE t.current_indicator = 1
  AND UPPER(t.derivation_type) = 'DERIVED'
  AND source_dm.current_indicator = 1
  AND target_dm.current_indicator = 1
  AND UPPER(source_dm.data_model_type) = 'PHYSICAL'
  AND UPPER(target_dm.data_model_type) = 'PHYSICAL'
  AND source_dm.filed_name IS NOT NULL
  AND target_dm.filed_name IS NOT NULL
  AND TRIM(source_dm.filed_name) <> ''
  AND TRIM(target_dm.filed_name) <> ''
  AND UPPER(TRIM(source_dm.filed_name)) = UPPER(TRIM(target_dm.filed_name))
  AND source_dm.model_id <> target_dm.model_id
ORDER BY
    source_system_name,
    source_attribute_name,
    target_system_name
LIMIT 10;
```

If `transformation_model_xref.transformation_id` references `transformation.transformation_id`, use this join instead:

```sql
ON source_xref.transformation_id = t.transformation_id
ON target_xref.transformation_id = t.transformation_id
```

If the database does not support `LIMIT`, use the equivalent syntax:

* SQL Server: `SELECT DISTINCT TOP 10 ...`
* Oracle / DB2: `FETCH FIRST 10 ROWS ONLY`
* Older Oracle: wrap the query and use `WHERE ROWNUM <= 10`

If the owner name column is `owner_name` instead of `model_owner_name`, replace:

```sql
source_owner.model_owner_name
target_owner.model_owner_name
```

with:

```sql
source_owner.owner_name
target_owner.owner_name
```

## Final Answer Format

Return only a concise explanation and one Markdown table.

Use these exact column headers:

| Source System Name | Source Attribute Name | Target System Name | Target Attribute Name |
| ------------------ | --------------------- | ------------------ | --------------------- |

Do not include SQL in the final answer unless the user asks for it.

Do not use database table names in the final answer unless the user asks for technical details.

Do not use the phrase "physical data model" in the final answer. Use "attribute".

Do not use the phrase "field name" or "filed_name" in the final answer. Use "attribute name".

Do not use the phrase "model owner". Use "system name" or "system".

## Final Response Template

Use this style:

```markdown
I found the following active derived lineage matches where the source and target attributes have the same attribute name:

| Source System Name | Source Attribute Name | Target System Name | Target Attribute Name |
|---|---|---|---|
| System A | CUSTOMER_ID | System B | CUSTOMER_ID |
| System C | ACCOUNT_NUMBER | System D | ACCOUNT_NUMBER |
```

If no rows are found, say:

```markdown
I could not find active derived lineage matches where both source and target attributes are physical attributes with the same non-empty attribute name.
```

Then briefly mention that the user may relax one of these filters:

* allow different attribute names
* include inactive lineage
* include non-derived lineage
* include logical models
