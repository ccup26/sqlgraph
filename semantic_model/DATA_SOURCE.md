# Data Source Contract

This file tells the SQL semantic modeling skill how to use project-specific data sources.

The skill does not require a fixed source format. Add any local docs, DDL files, schema exports, common SQL files, relation notes, or other sources below. For each source, describe what it contains, how to parse it, and how trustworthy it is.

## Global Rules

- Do not infer relationships from field names alone unless this file explicitly allows it.
- Source-declared `table.field -> table.id` mappings can become entity foreign key candidates, but still need evidence entries.
- Source-declared reusable associations or multi-hop paths can become relationship candidates, but still need evidence entries.
- SQL `join` / `on` clauses are relationship evidence.
- SQL `where` clauses are usually metric filters, not relationship facts.
- Field comments can suggest business terms, but ambiguous terms must stay scoped.
- Conflicts with existing model facts must be confirmed by the user before changes are applied.

## Sources

### example_schema_doc

- path: `docs/db/schema.md`
- type: markdown
- purpose: Table names, fields, comments, enum hints.
- usage:
  - Use table names to propose `entities`.
  - Use primary key and soft-delete descriptions when explicitly stated.
  - Use enum descriptions to update `entities.*.enums`.
  - Do not infer `relationships` from similar field names only.
- confidence:
  - entity_table: high
  - enum_value: medium
  - relationship: none

### example_relation_doc

- path: `docs/db/relations.md`
- type: markdown
- purpose: Human-maintained table relationship notes.
- usage:
  - Explicit `table.field -> table.field` mappings can propose `relationships`.
  - If a mapping only proves one field references another entity key, record it first under `entities.*.foreign_keys`.
  - Keep each mapping as evidence of type `source_declared_relationship`.
  - If relation notes conflict with SQL joins or existing confirmed relationships, ask the user.
- confidence:
  - relationship: high

### example_common_sql

- path: `docs/sql/common-sql.md`
- type: markdown
- purpose: Common business SQL and query examples.
- usage:
  - Extract `metrics`, `business_terms`, and relationship evidence from SQL.
  - Treat repeated joins as stronger relationship evidence.
  - Treat filters and aggregations as metric facts.
  - Do not turn every SQL example into a new metric if it only changes parameter entry.
- confidence:
  - sql_join: medium_high
  - metric: medium
  - business_term: medium

## Add Project Sources Here

### source_models

- path: `.cursor/skills/sqlgraph/semantic_model/source_models/`
- type: exported model json directory
- purpose: Local table model metadata, field comments, enum dictionaries, main fields, system fields, and declared model relations.
- indexes:
  - `_index.jsonl`: one model per line, including filename, key, tableName, modelType, propsType, and parentKey.
  - `_key_to_file.json`: maps model key to the json filename.
  - `_summary.json`: extraction summary and coverage counts.
- usage:
  - Use `_index.jsonl` and `_key_to_file.json` to locate model files by model key or table name.
  - Use `props.props.tableName` and model `name` to propose or update `entities`.
  - Use `props.props.mainField` only as stable business-key metadata; do not expand all fields into `entities`.
  - Use system fields such as `id` and `deleted` to update primary key and soft-delete metadata.
  - Use `dictPros.dictValues` to update stable enum definitions under `entities`.
  - Use `relationMeta` as source-declared relationship evidence. Record field-level references under `entities.*.foreign_keys`; promote them to `relationships` only when they are reusable associations or business paths.
  - Keep relationship facts as candidates unless they match existing code/SQL evidence or are later confirmed by the user.
  - If multiple model files share the same table name under different domains, prefer the model whose key/domain matches the current code module and record the ambiguity when needed.
- confidence:
  - entity_table: high
  - primary_key: high when system field `id` is present and unique
  - soft_delete: high when system field `deleted` is present and `physicalDelete` is false
  - enum_value: high
  - source_declared_relationship: high
  - metric: none


