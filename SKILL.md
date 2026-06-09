---
name: sqlgraph
description: Maintains evidence-driven SQL semantic models and generates base SQL drafts. Use when initializing or updating SQL semantic models, remembering natural language or SQL as metrics/relationships/entities/business terms, auditing model conflicts, migrating large models into split YAML or SQL graph storage, or when generating SQL or SQL-related code logic.
---

# SQLGraph

## Purpose

Use this skill to maintain a lightweight, evidence-driven SQL semantic model and to generate base SQL drafts before project-specific code conversion.

The model starts simple and scales only when needed:

- Level 1: single-file YAML for small models.
- Level 2: split YAML plus a lightweight index for medium models.
- Level 3: optional SQLite semantic graph for relationship-heavy models.

The model is evidence-driven. It separates:

- `entities`: business objects, table mappings, primary keys, soft-delete rules, stable enums, and evidence-backed field-level foreign keys.
- `relationships`: named reusable entity associations or business paths only.
- `metrics`: business metric definitions, parameters, measures, filters, paths, examples.
- `business_terms`: scoped natural-language terms and aliases.
- `evidence`: where each entity or relationship fact came from.

## Always Follow

1. Keep `relationships` pure: only describe named entity associations or reusable business paths. Do not store metric filters, query parameters, or business measures in relationships.
2. Keep `entities` minimal: table mapping, primary key, soft delete, stable enum fields, evidence-backed field-level foreign keys, and confirmed stable semantics only. Ordinary query fields usually belong in `metrics`.
3. Treat `entities.*.foreign_keys` as atomic field facts only, such as `sales_order.customer_id -> customer.id`. Do not put filters, measures, query parameters, path names, or multi-hop chains in entity foreign keys.
4. Prefer confirmed `relationships` when generating SQL. Use entity foreign keys only as fallback path hints when no relationship path exists or when auditing missing relationships.
5. `business_terms` must be scoped by entity, metric, or domain context. Do not treat words like "已过账", "数量", or "金额" as global truths.
6. Initializing or updating `entities`, entity foreign keys, and `relationships` requires evidence. If there is no evidence, keep the item as draft and do not treat it as confirmed.
7. Conflicting updates must be confirmed by the user before changing existing model facts.
8. Generate base SQL only. Let the outer task convert it to `LambdaQueryWrapper`, XML, annotations, `MD.query*`, or other project-specific styles after checking local patterns.
9. When generating SQL for delete semantics, never generate physical `DELETE`. Generate a logical delete `UPDATE ... SET <soft_delete_field> = 1 ...` using the modeled soft-delete field, usually `deleted`.
10. Do not compile or run validation commands unless the user explicitly asks.

## Files

Default project files live under:

```text
.cursor/skills/sqlgraph/semantic_model/
```

Core files:

- `DATA_SOURCE.md`: user-maintained contract describing available data sources and how to interpret them.
- `evidence.yaml`: evidence registry and confidence policy.
- `semantic-model.yaml`: Level 1 single-file semantic model.
- `index.yaml`: Level 2+ lightweight routing index. Read this first when present.
- `entities/`: Level 2 entity files, usually one entity per file.
- `relationships.yaml` or `relationships/`: Level 2 relationship facts.
- `metrics/`: Level 2 metric files grouped by business domain.
- `terms/`: Level 2 business term files grouped by scope domain.
- `semantic_graph.sqlite`: optional Level 3 relationship graph index.

`DATA_SOURCE.md` is an interface contract, not a connector. Follow its instructions to read local docs, DDL, schema exports, common SQL, relation notes, or other user-provided sources. Do not assume a fixed external data format.

Before using the model, read `DATA_SOURCE.md` and `evidence.yaml` when present. Then detect the active model storage:

1. If `semantic_graph.sqlite` exists and the task needs path search or graph audit, treat Level 3 as available.
2. If `index.yaml` exists, treat split YAML as the canonical routing layer.
3. Otherwise use `semantic-model.yaml`.

## Storage Strategy

### Level 1: Single-File YAML

Use `semantic-model.yaml` when the model is small. This remains the default for new models because it is easiest to inspect and maintain.

Prefer Level 1 while the model is below most of these thresholds:

- `semantic-model.yaml` is under about 800-1200 lines.
- Metrics are under about 30.
- Relationships are under about 50.
- `generate` can match intent without repeatedly loading a large model.

### Level 2: Split YAML With Index

Use split YAML when the model becomes too large for comfortable full-file reading, but is still understandable as human-maintained YAML.

Target structure:

```text
semantic_model/
├── DATA_SOURCE.md
├── evidence.yaml
├── index.yaml
├── entities/
│   ├── <entity_name>.yaml
│   └── ...
├── relationships.yaml
│   # or relationships/<domain>_relationships.yaml when relationship count grows
├── metrics/
│   ├── <domain>_metrics.yaml
│   └── ...
└── terms/
    ├── <domain>_terms.yaml
    └── ...
```

`index.yaml` is a derived routing index, not the canonical fact source. Canonical facts live in the entity, relationship, metric, and term files. The index may duplicate only minimal lookup fields:

- Entity: `name`, `file`, `table`, `description`, optional `domain`.
- Relationship: `file` pointers, or domain-to-file pointers when relationships are split.
- Metric: `name`, `file`, `display`, `grain`, `keywords`, optional `domain`.
- Terms: `scope`, `file`, optional `domain`.

When adding or updating facts in split YAML, update the affected fact file and then update `index.yaml` if routing or keyword data changed. Audit must check index coverage and stale index entries.

Metric `keywords` should be generated from the metric display name, aliases, measure aliases, parameter aliases, and important filter meanings. Keep keywords short and intent-oriented; do not copy full filters, paths, SQL fragments, or evidence into the index.

### Level 3: SQLite Semantic Graph

Use `semantic_graph.sqlite` only when relationships become graph-like enough that YAML routing is no longer sufficient.

Suggest Level 3 when any of these are true:

- Relationships exceed about 150.
- Reusable paths frequently require 3 or more hops.
- Multiple domains share dense cross-domain relationships.
- `generate` often needs path search rather than direct metric lookup.
- Audits spend most of their effort finding duplicate, conflicting, or missing relationship paths.

The SQLite graph is an index over semantic facts, not a replacement for evidence discipline. It may store tables such as:

- `entities`
- `entity_fields`
- `foreign_keys`
- `relationships`
- `relationship_edges`
- `metrics`
- `business_terms`
- `evidence_links`

Keep the graph read-first for generation and auditing. If YAML files still exist as canonical facts, synchronize the graph after writes and report any mismatch before trusting generated SQL.

The graph is a local semantic-model artifact. Do not use it as permission to query or mutate external databases. External database access remains read-only unless the user explicitly grants a different permission for a specific task.

### Domain Rules

Prefer explicit `domain` fields when they exist. Otherwise infer domain in this order:

1. Entity domain from `grain`, `from`, `to`, or measure source.
2. Metric name prefix that matches a known entity or domain.
3. Term scope that matches an entity or metric.
4. `common`.

Do not move facts between domains silently. If regrouping would change established file ownership, present the proposed move and ask the user to confirm.

## Commands

### `/sqlgraph init`

Build an initial draft model from the codebase and any sources declared in `DATA_SOURCE.md`.

Look for evidence from:

- Entity-table mapping: `@TableName`, `@Table`, XML `resultMap`, Mapper, Repository.
- Entity foreign key evidence: database physical foreign keys, DDL constraints, explicit schema relation notes, exported model relation metadata, repeated joins, Service batch `IN` lookups.
- Relationship evidence: XML `join`/`on`, `association`, `collection`, Service assembly, batch `IN` queries, source-declared relation notes.
- Query logic: `LambdaQueryWrapper*`, `QueryWrapper`, `MD.query*`, `@Select`, XML SQL.
- Existing SQL examples that reveal logical chains.

Output or write only evidence-backed candidates. Mark uncertain inference as `draft` with `confidence: low`. If evidence only proves `table.field -> other_table.id`, record it as an entity foreign key first; promote it to a `relationship` only when the association is reusable, named, or business-meaningful.

Default to Level 1 single-file YAML. If an `index.yaml` already exists, treat initialization as an update to the existing split model instead of creating a parallel model.

### `/sqlgraph update`

Refresh the model from sources declared in `DATA_SOURCE.md`.

Use this for schema docs, field comments, dictionary values, source-declared relationships, DDL, SQL examples, or other external data. Produce a diff first. Do not silently overwrite existing facts.

Read according to the active storage level:

- Level 1: read `semantic-model.yaml`.
- Level 2: read `index.yaml`, then only the entity, relationship, metric, or term files relevant to the refreshed source.
- Level 3: query `semantic_graph.sqlite` for candidate affected facts, then read canonical files if YAML remains the source of truth.

When adding new facts in Level 2, update the affected file and `index.yaml`. When adding relationship facts in Level 3, update or rebuild the graph index after the canonical write.

### `/sqlgraph remember`

Use user-provided natural language or SQL to add or update semantic facts.

Typical additions:

- New or updated `metrics`.
- Scoped `business_terms`.
- Metric parameters, measures, filters, examples.
- Evidence-backed `entities`, entity foreign keys, or `relationships`.

If the new input conflicts with existing facts, stop and ask the user how to resolve it.

For split YAML, use `index.yaml` to locate the canonical file before writing. For new metrics or terms, determine the domain first, write to the domain file, then update index keywords or scope pointers.

### `/sqlgraph generate`

Generate a base SQL draft from the current semantic model.

Process:

1. Detect the active storage level.
2. Level 1: read `semantic-model.yaml`.
3. Level 2: read `index.yaml`, match user intent against metric keywords and scoped terms, then read only the matched metric file, relevant term file, relationship file or domain relationship file, and needed entity files.
4. Level 3: query the graph for candidate metrics, terms, entities, and paths, then read canonical details for the selected candidates.
5. Match user intent to scoped `business_terms`.
6. Select the best `metric`.
7. Resolve parameters, measure, filters, and explicit relationship paths.
8. If no relationship path is available, try to connect entities through `entities.*.foreign_keys` or the graph path index. Treat inferred paths as lower confidence and cite each foreign key or relationship edge used.
9. Generate SQL with default soft-delete filters where modeled.
10. If the intent is deletion, generate logical delete SQL only: `UPDATE <table> SET <soft_delete_field> = 1 WHERE ...`; do not generate `DELETE FROM`.
11. Report uncertainty and cite the model facts used.

### `/sqlgraph audit`

Review the semantic model for:

- Entities, entity foreign keys, or relationships without evidence.
- Conflicting joins for the same entity pair.
- Relationships that duplicate or contradict entity foreign keys.
- Missing relationships that should be promoted from repeatedly used foreign-key paths.
- Global terms that should be scoped.
- Metrics with duplicated business meaning.
- Metrics whose examples disagree with parameters, filters, or measures.
- Relationships carrying filters or measures.
- Split YAML indexes that miss facts or point to stale files.
- SQLite graph rows that disagree with canonical YAML facts.

For Level 2 audits, read `index.yaml` and all split files. For Level 3 audits, compare the graph index with canonical facts and report whether the graph should be rebuilt.

### `/sqlgraph migrate-split`

Migrate Level 1 `semantic-model.yaml` into Level 2 split YAML after user approval.

Use this only when the model is large enough to benefit from selective reading. Do not migrate small models just because split YAML is available.

Migration rules:

1. Back up `semantic-model.yaml` to `semantic-model.yaml.bak` before writing split files.
2. Preserve every canonical fact exactly once in the split files.
3. Treat `index.yaml` as a derived lookup index, so duplicated summaries in the index are allowed but not canonical.
4. Keep `DATA_SOURCE.md` and `evidence.yaml` unchanged.
5. Group entities one per file.
6. Group metrics and terms by explicit domain first, then by the domain inference rules.
7. Keep relationships in `relationships.yaml` while relationship count is moderate; use `relationships/<domain>_relationships.yaml` when the relationship file would become the next large bottleneck.
8. Read back the new files and compare counts for entities, relationships, metrics, and term scopes.
9. Verify every entity, metric, and term scope is represented in `index.yaml`.
10. Keep the backup and report that the user may remove it after confirming the migration.

Stop and report discrepancies instead of deleting or rewriting the original model.

### `/sqlgraph migrate-graph`

Prepare or refresh Level 3 `semantic_graph.sqlite` for relationship-heavy models after user approval.

Use this when relationship count, multi-hop path search, or cross-domain relationship density makes YAML lookup inefficient.

Process:

1. Identify the canonical source of facts: split YAML when present, otherwise `semantic-model.yaml`.
2. Design or reuse graph tables for entities, fields, foreign keys, relationships, relationship edges, metrics, business terms, and evidence links.
3. Load only evidence-backed or explicitly draft facts with their `status`, `confidence`, and evidence references.
4. Preserve logical-delete, scoped-term, and metric-boundary semantics exactly as modeled.
5. Verify row counts against canonical fact counts.
6. During generation, use the graph for candidate lookup and path discovery, but cite canonical facts and evidence.

Do not replace YAML canonical facts with the SQLite graph unless the user explicitly asks for a database-first semantic model.

## Conflict Handling

When a new fact conflicts with an existing fact, present the conflict clearly and ask the user to choose. Do not auto-fix.

Conflict examples:

- Existing relationship: `delivery_note.rel_sls_order_id = sales_order.id`; new source says `delivery_note.so_code = sales_order.so_code`.
- Existing entity foreign key: `delivery_note.rel_sls_order_id -> sales_order.id`; new source says it references `sales_order.so_code`.
- Existing term: delivery note "已过账" means `biz_status = POSTED`; new SQL uses `status = AUDITED`.
- Existing metric measure is `settle_qty`; new SQL says the same metric should sum `actual_qty`.

Resolution options should usually include:

- Keep existing fact and record the new input as an exception or low-confidence candidate.
- Update the existing fact.
- Add a scoped new fact.
- Ask for more evidence.

## Model Boundary

This skill does not own project implementation style. For SQL-related coding:

1. Use this skill to produce the base source SQL and explain semantic assumptions.
2. Then inspect the project module's existing style.
3. Convert the SQL into the local pattern only after the base SQL is clear.

Do not put ordinary fields into `entities` just because they appear in one query. Prefer:

- `entities.*.foreign_keys` only for evidence-backed fields that reference another entity key.
- `metrics.parameters` for query inputs such as contract code or id.
- `metrics.measure` for aggregation fields.
- `metrics.filters` for metric-specific state or time constraints.
- `business_terms` for scoped natural-language aliases.

