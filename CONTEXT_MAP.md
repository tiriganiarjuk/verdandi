# Verdandi -- Context Map

## 1. Purpose

Verdandi ("what is becoming") is a typed composition hierarchy of ~179 YAML files organized into 14 levels across 4 tiers. It defines the structural vocabulary for agent tool definitions -- what fields exist, what types they have, and how they compose into groups, sections, and complete schemas. Verdandi is a data directory, not an executable. Other tools (Draupnir, Nornir, transform_to_universal, scaffold_definition.py, inspect_schema.py) consume it to generate JSON Schemas, validate agent definitions, and produce platform-specific outputs. This CONTEXT_MAP helps new agents orient to where knowledge about Verdandi lives and helps working LLMs know where to refresh context mid-session.

---

## 2. Primary References

| Resource | Path | What It Contains | When to Read |
|----------|------|------------------|--------------|
| Context summary | `context/verdandi.md` | Complete system description: 14 levels, file inventory, composition rules, raw/resolved/reduced pipeline, Draupnir connection, gotchas, enrichments from 6 documentation files, reconciliation log | First read for any verdandi work. The single most comprehensive resource. |
| Quickstart | `tools/verdandi/QUICKSTART.md` | Concise operational guide: hierarchy overview, stage variants, conditional fields, polymorphic arrays, what NOT to do | When you need to work with verdandi YAML files and want the rules without full context |
| Composition Rules | `tools/verdandi/COMPOSITION_RULES.md` | Formal specification of all 14 levels: accepted fields, YAML structure, reference model, compile-time enforcement | When you need to create or modify a verdandi type and need the authoritative spec for that level |
| Migration Checklist | `tools/verdandi/MIGRATION_CHECKLIST.md` | Maps every old toolcompose file to its verdandi equivalent, documents what collapsed, what split, provenance of each type | When you need to understand where a type came from or why it exists |
| Tool Building (Section 5) | `definitions/TOOL_BUILDING.md` (lines 670-779) | Verdandi within the full pipeline context: design rationale, tier summary, type examples, raw/resolved/reduced, conditional system, composition discipline D10, hierarchical locking | When you need to understand how verdandi fits the end-to-end pipeline |

---

## 3. Documentation References

All `documentation/` files that cover verdandi topics, identified from the DOCUMENT_INDEX cross-references.

| Document | Path | Relevance to Verdandi | Freshness Note |
|----------|------|-----------------------|----------------|
| Hierarchy Rebuild Design | `documentation/toolcompose/HIERARCHY_REBUILD_DESIGN.md` | Design decisions for the verdandi hierarchy replacing the old toolcompose system. Problem that drove the rebuild (55 broken YAML files). The `composed/` split rationale. | STALE: Lists 12 levels; implementation has 14 (missing `pattern`, `enum`). Uses old terms (`primitive`/`supergroup` vs current `atom`+`enum`+`union`/`group`). Historical design record -- not updated to reflect final implementation. |
| Schema Composition Explained (by LLM, for LLM) | `documentation/toolcompose/SCHEMA_COMPOSITION_EXPLAINED_BY_LLM_FOR_LLM.md` | Operational guide to composition hierarchy written from experience of getting it wrong. Pragmatic exceptions to hierarchy rule, `requirement: inherit` mechanism, composition discipline rules. | STALE: References 8-level hierarchy with old directory names (`primitives/`, `composed/`, `groups/`). File counts (~249) do not match current (~179). References old loading pattern vs current supply-driven level-ordered loading. |
| Why Schema Composition Is the Key | `documentation/toolcompose/WHY_SCHEMA_COMPOSITION_IS_THE_KEY.md` | Core design philosophy: types represent lifecycle stages not just structural groupings. DAG schema validation, @include semantics, hierarchical locking, Phoenix project evidence. | Contains fundamental design insight that remains accurate. Structural references use old terminology but concepts are current. |
| Agent Decomposition | `documentation/toolcompose/AGENT_DECOMPOSITION.md` | 10-agent inventory that informed the schema design. Structural findings that shaped verdandi types (processing_capability_model, write tool categories, security read path scope, guardrails split). | Pre-rebuild analysis. Agent findings are accurate; type references use old naming. |
| Critical Decisions | `documentation/pipeline/CRITICAL_DECISIONS.md` | Design Decisions D1-D11 that shaped the verdandi type system. Nornir validation gates. Authoritative status document for the rebuild. | STALE SECTION: "What's Built" describes the old 8-level system (`standards/toolcompose/` directory, old file counts). Staleness note added to document. Design decisions D1-D11 remain accurate. |
| Definition Format | `documentation/pipeline/DEFINITION_FORMAT.md` | Complete authoritative tree specification for all 6 agent definition sections with every field, type, and composition relationship. x-constraint patterns, enforcement output tool invocation signature. | STALE: Uses old type annotations (`p` = primitive, `c` = composed, `G` = super-group). Tree legend does not include current level names. Staleness note added. Field-level content remains accurate. |
| Tool Compose Reasoning | `documentation/toolcompose/TOOL_COMPOSE_REASONING.md` | Design rationale behind the multilayered constraint architecture (three layers: prompt awareness, runtime recovery, structural limitation). Naive agent principle. | Background rationale. Not verdandi-specific but explains WHY the type system enforces constraints. |

---

## 4. Schema References

All schemas in `schemas/` are generated by Draupnir from verdandi YAML compositions. They are derived artifacts -- editing them directly is forbidden. The chain is: Verdandi YAML -> Draupnir -> `schemas/*.schema.json`.

| Schema | Path | Verdandi Source | Pipeline Role |
|--------|------|-----------------|---------------|
| agent-raw-definition | `schemas/agent-raw-definition.schema.json` | `schema/agent-raw-definition.yaml` | Entry point: validates authored TOML definitions |
| agent-includes-resolved | `schemas/agent-includes-resolved.schema.json` | `schema/agent-includes-resolved.yaml` | After @include path resolution |
| agent-paths-resolved | `schemas/agent-paths-resolved.schema.json` | `schema/agent-paths-resolved.yaml` | After path absolutization |
| agent-permissions-resolved | `schemas/agent-permissions-resolved.schema.json` | `schema/agent-permissions-resolved.yaml` | After security permission calculation |
| agent-execution-merged | `schemas/agent-execution-merged.schema.json` | `schema/agent-execution-merged.yaml` | After execution section merge |
| agent-criteria-merged | `schemas/agent-criteria-merged.schema.json` | `schema/agent-criteria-merged.yaml` | After criteria section merge |
| agent-instructions-reduced | `schemas/agent-instructions-reduced.schema.json` | `schema/agent-instructions-reduced.yaml` | After instruction @include inlining |
| agent-examples-reduced | `schemas/agent-examples-reduced.schema.json` | `schema/agent-examples-reduced.yaml` | After example @include inlining |
| agent-guardrails-reduced | `schemas/agent-guardrails-reduced.schema.json` | `schema/agent-guardrails-reduced.yaml` | After guardrails @include inlining |
| agent-sf-reduced | `schemas/agent-sf-reduced.schema.json` | `schema/agent-sf-reduced.yaml` | After success/failure criteria inlining |
| agent-universal-format | `schemas/agent-universal-format.schema.json` | `schema/agent-universal-format.yaml` | Final platform-agnostic output |
| agent-universal-render | `schemas/agent-universal-render.schema.json` | `schema/agent-universal-render.yaml` | Consumption-oriented regrouping for rendering |

---

## 5. Consuming Tools

Tools that read verdandi data or its derived artifacts, with paths and how they use it.

| Tool | Path | How It Uses Verdandi |
|------|------|----------------------|
| **Draupnir** | `tools/draupnir/` | The compiler. Reads all verdandi YAML in dependency order (bottom-up through the Level enum), validates into frozen Pydantic models via `BeforeValidator` ref resolution, and generates JSON Schema files to `schemas/`. Supply-driven, level-ordered loading. IO in exactly two places: `loader.py` (reads) and `writer.py` (writes). |
| **transform_to_universal** | `tools/transform_to_universal/` | Consumes the JSON Schema files generated by Draupnir. Uses `datamodel-code-generator` to auto-generate Pydantic models from each schema (`generate_structures.py`). Implements an 11-step DAG pipeline transforming raw TOML definitions through resolution stages to universal format. Does not read verdandi YAML directly -- consumes Draupnir's JSON Schema output. |
| **inspect_schema.py** | `tools/inspect_schema.py` | Reads any generated JSON Schema and produces an interactive HTML inspection report showing field hierarchy, requirement status, extension metadata, polymorphic alternatives, and cross-group conditionals. Design decision D5: reads ONLY generated JSON Schema (not verdandi YAML), verifying what was actually produced. |
| **scaffold_definition.py** | `tools/scaffold_definition.py` | Reads `schemas/agent-raw-definition.schema.json` to produce TOML scaffold files with every field annotated by type, requirement status, and description. Placeholder values for authors to fill in. Consumes Draupnir output, not verdandi YAML directly. |
| **Nornir** | `tools/nornir/` | Rust/PyO3 workspace with 19 validation gates. Compiles schemas into Rust validation modules. Uses the generated JSON Schemas for runtime validation of agent definition TOML files at every pipeline boundary. 8 CLI check tools, 2 writers, 1 dispatcher. |

---

## 6. Directory Inventory

Complete listing of verdandi's subdirectories with current file counts and contents.

**Location:** `/Users/johnny/.ai/spaces/bragi/tools/verdandi/`

### Tier 1: Fields (leaf nodes)

| Directory | Files | Description |
|-----------|-------|-------------|
| `pattern/` | 11 | Raw regex patterns. absolute, claude, env, home, kebab, relative, root, semver, snake, timestamp_zulu, traversal |
| `atom/` | 10 | Constrained base types (string/integer/boolean + pattern + exclusions). agent_title, boolean, integer, kebab_string, path_absolute, path_relative, semver_string, snake_string, string_plain, timestamp_zulu |
| `enum/` | 14 | Enumerated allowed values. capabilities, dispatch_background_mode, dispatch_input_delivery, dispatch_mode, enforcement_tool_format, input_format, instruction_mode, notify_delivery, output_format_kind, output_name_known, param_type, processing_capability, return_mode, write_frequency |
| `element/` | 5 | Atom/enum + runtime format contract. agent_description, agent_name, path_exists_absolute, path_exists_relative, path_exists_schema |
| `union/` | 2 | OneOf composition of fields. path_any, path_exists_any |

### Tier 2: Simple Containers (fields only, no nesting)

| Directory | Files | Description |
|-----------|-------|-------------|
| `tuple/` | 1 | Fixed-length typed sequences. dispatch_batch_size |
| `simplearray/` | 33 | Variable-length arrays of one field type. Enforcement grants (explicit/implicit x read/write/edit/append/search/inspect), guardrails, security paths/capabilities/IO, context paths, success/failure evidence, role expertise. Many in raw/resolved pairs. |
| `simplegroup/` | 18 | Flat objects with required/optional/conditional zones. context_item (raw/resolved), example_entry, failure_item, instruction_step, notify_item, parameter_item, success_item, and various include_* types for @include references. |

### Tier 3: Complex Containers (can nest other containers)

| Directory | Files | Description |
|-----------|-------|-------------|
| `array/` | 18 | Variable-length arrays of simplegroups. dispatch_parameters, execution_instructions (raw/reduced/resolved), success/failure_criteria (raw/reduced/resolved), context_required/available (raw/resolved), example_entries (raw/reduced/resolved), notify. |
| `group/` | 25 | Objects containing fields + containers. metadata, role, enforcement grants (explicit/implicit), enforcement_output_tool, guardrails (raw/reduced/resolved), security IO/paths/schema/capabilities, processing, return, example_group variants, dispatch, communication, context. |
| `superarray/` | 3 | Variable-length arrays of groups. execution_examples_raw, execution_examples_reduced, execution_examples_resolved |

### Tier 4: Top Level

| Directory | Files | Description |
|-----------|-------|-------------|
| `section/` | 17 | Complex container composition. criteria (raw/reduced/resolved), enforcement, execution (raw/reduced/resolved), indirect, security (raw/resolved), task, cross-stage combinatorial variants (examples_resolved_instructions_reduced, etc.) |
| `shallowsection/` | 10 | Fields + limited containers (max 2). anti_patterns, constraints, dispatcher, identity, implicit_grants, input, output, return_format, security_boundary, writing_output. Built for the universal-render schema. |
| `schema/` | 12 | Root files referencing sections/shallowsections. agent-raw-definition, agent-includes-resolved, agent-paths-resolved, agent-permissions-resolved, agent-execution-merged, agent-instructions-reduced, agent-examples-reduced, agent-guardrails-reduced, agent-criteria-merged, agent-sf-reduced, agent-universal-format, agent-universal-render |

### Documentation files in this directory

| File | Purpose |
|------|---------|
| `QUICKSTART.md` | Operational quickstart guide |
| `COMPOSITION_RULES.md` | Formal specification of all levels |
| `MIGRATION_CHECKLIST.md` | Old toolcompose to verdandi file mapping |
| `CONTEXT_MAP.md` | This file |

**Total: 179 YAML files + 4 documentation files**

---

## 7. Context Refresh Guide

For an LLM mid-session that needs to refresh understanding:

| If you need to... | Read this |
|--------------------|-----------|
| Understand what composition types exist and how they relate | `context/verdandi.md` -- the full context summary with 14-level hierarchy, file inventory, and composition rules |
| Understand how verdandi fits the end-to-end pipeline | `definitions/TOOL_BUILDING.md` Section 5 (lines 670-779) for verdandi in pipeline context, Section 2 (lines 82-160) for the full pipeline flow |
| Add or modify a type | `tools/verdandi/COMPOSITION_RULES.md` -- formal spec for each level's accepted fields and YAML structure. Then `tools/verdandi/QUICKSTART.md` for the "What NOT to Do" list |
| Understand design rationale | `documentation/toolcompose/WHY_SCHEMA_COMPOSITION_IS_THE_KEY.md` for core philosophy (types = lifecycle stages). `documentation/toolcompose/HIERARCHY_REBUILD_DESIGN.md` for the rebuild decisions (note: uses old terminology). `definitions/TOOL_BUILDING.md` Section 5 for design rationale summary |
| Understand the raw/resolved/reduced pipeline | `context/verdandi.md` section "The Raw / Resolved / Reduced Pipeline" or `tools/verdandi/QUICKSTART.md` section "Raw / Resolved / Reduced Stages" |
| Understand conditional fields | `tools/verdandi/COMPOSITION_RULES.md` section on simplegroup/ (conditional keywords: only_when, always_when, not_when) |
| Map old toolcompose files to verdandi equivalents | `tools/verdandi/MIGRATION_CHECKLIST.md` |
| Understand which schemas are generated and what they validate | Section 4 of this file (Schema References) or `context/verdandi.md` section on schema/ level |
| Understand what consumes verdandi output | Section 5 of this file (Consuming Tools) or `context/verdandi.md` opening section on consumers |
| Check the authoritative field-level definition spec | `documentation/pipeline/DEFINITION_FORMAT.md` (note: uses old type annotations but field content is accurate) |

---

## 8. Known Issues / Active Work

**agent-anthropic-render.schema.json:** This schema is being built in verdandi as part of the universal_to_canonical platform adapter work. The `shallowsection/` level (10 files) and `schema/agent-universal-render.yaml` were created specifically for this render format. This is known active work, not a gap.

**Documentation staleness:** Four documentation files describe the pre-rebuild 8-level system and have not been updated to reflect the current 14-level verdandi hierarchy:
- `HIERARCHY_REBUILD_DESIGN.md` -- historical design record (12 levels, old names)
- `SCHEMA_COMPOSITION_EXPLAINED_BY_LLM_FOR_LLM.md` -- 8-level operational guide (old directory names, wrong file counts)
- `CRITICAL_DECISIONS.md` -- "What's Built" section describes old system (staleness note added)
- `DEFINITION_FORMAT.md` -- old type annotations (staleness note added)

These are pre-rebuild artifacts. Rather than rewriting historical documents (which would destroy their value as design records), the reconciliation preserved them as-is with staleness notes where appropriate. The `context/verdandi.md` file reflects the current 14-level system and should be treated as the authoritative source.

**No unresolved discrepancies:** All differences between documentation and implementation are clearly attributable to the documentation predating the 14-level rebuild. The reconciliation log in `context/verdandi.md` documents this with 4 STALE_DOC items and 0 ALERT items.
