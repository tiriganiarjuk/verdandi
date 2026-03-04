# Verdandi -- Context Map

## 1. Purpose

Verdandi ("what is becoming") is a typed composition hierarchy of 351 YAML files organized into 15 levels across 4 tiers. It defines the structural vocabulary for agent tool definitions -- what fields exist, what types they have, and how they compose into groups, sections, and complete schemas. Verdandi is a data directory, not an executable. Other tools (Draupnir, Regin, Galdr, Nornir, scaffold_definition.py, inspect_schema.py) consume it to generate JSON Schemas, validate agent definitions, and produce platform-specific outputs. This CONTEXT_MAP helps new agents orient to where knowledge about Verdandi lives and helps working LLMs know where to refresh context mid-session.

---

## 2. Primary References

| Resource | Path | What It Contains | When to Read |
|----------|------|------------------|--------------|
| Context summary | `context/verdandi.md` | Complete system description: 15 levels, file inventory, composition rules, raw/resolved/reduced pipeline, Draupnir connection, gotchas, enrichments from 6 documentation files, reconciliation log | First read for any verdandi work. The single most comprehensive resource. |
| Quickstart | `tools/verdandi/QUICKSTART.md` | Concise operational guide: hierarchy overview, stage variants, conditional fields, polymorphic arrays, what NOT to do | When you need to work with verdandi YAML files and want the rules without full context |
| Composition Rules | `tools/verdandi/COMPOSITION_RULES.md` | Formal specification of all 15 levels: accepted fields, YAML structure, reference model, compile-time enforcement | When you need to create or modify a verdandi type and need the authoritative spec for that level |
| Migration Checklist | `tools/verdandi/MIGRATION_CHECKLIST.md` | Maps every old toolcompose file to its verdandi equivalent, documents what collapsed, what split, provenance of each type | When you need to understand where a type came from or why it exists |
| Tool Building (Section 5) | `definitions/TOOL_BUILDING.md` (lines 670-779) | Verdandi within the full pipeline context: design rationale, tier summary, type examples, raw/resolved/reduced, conditional system, composition discipline D10, hierarchical locking | When you need to understand how verdandi fits the end-to-end pipeline |

---

## 3. Documentation References

All `documentation/` files that cover verdandi topics, identified from the DOCUMENT_INDEX cross-references.

| Document | Path | Relevance to Verdandi | Freshness Note |
|----------|------|-----------------------|----------------|
| Hierarchy Rebuild Design | `documentation/toolcompose/HIERARCHY_REBUILD_DESIGN.md` | Design decisions for the verdandi hierarchy replacing the old toolcompose system. Problem that drove the rebuild (55 broken YAML files). The `composed/` split rationale. | STALE: Lists 12 levels; implementation has 15 (missing `pattern`, `enum`, `field`). Uses old terms (`primitive`/`supergroup` vs current `atom`+`enum`+`union`/`group`). Historical design record -- not updated to reflect final implementation. |
| Schema Composition Explained (by LLM, for LLM) | `documentation/toolcompose/SCHEMA_COMPOSITION_EXPLAINED_BY_LLM_FOR_LLM.md` | Operational guide to composition hierarchy written from experience of getting it wrong. Pragmatic exceptions to hierarchy rule, `requirement: inherit` mechanism, composition discipline rules. | STALE: References 8-level hierarchy with old directory names (`primitives/`, `composed/`, `groups/`). File counts (~249) do not match current (~348). References old loading pattern vs current supply-driven level-ordered loading. |
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
| agent-paths-resolved | `schemas/agent-paths-resolved.schema.json` | `schema/agent-paths-resolved.yaml` | After path absolutization |
| agent-permissions-resolved | `schemas/agent-permissions-resolved.schema.json` | `schema/agent-permissions-resolved.yaml` | After security permission calculation |
| agent-execution-merged | `schemas/agent-execution-merged.schema.json` | `schema/agent-execution-merged.yaml` | After execution section merge |
| agent-criteria-merged | `schemas/agent-criteria-merged.schema.json` | `schema/agent-criteria-merged.yaml` | After criteria section merge |
| agent-instructions-reduced | `schemas/agent-instructions-reduced.schema.json` | `schema/agent-instructions-reduced.yaml` | After instruction @include inlining |
| agent-examples-reduced | `schemas/agent-examples-reduced.schema.json` | `schema/agent-examples-reduced.yaml` | After example @include inlining |
| agent-guardrails-reduced | `schemas/agent-guardrails-reduced.schema.json` | `schema/agent-guardrails-reduced.yaml` | After guardrails @include inlining |
| agent-success-reduced | `schemas/agent-success-reduced.schema.json` | `schema/agent-success-reduced.yaml` | After success/failure criteria inlining |
| agent-includes-merged | `schemas/agent-includes-merged.schema.json` | `schema/agent-includes-merged.yaml` | After all includes merged (content branch convergence) |
| agent-universal-format | `schemas/agent-universal-format.schema.json` | `schema/agent-universal-format.yaml` | Final platform-agnostic output |
| agent-universal-render | `schemas/agent-universal-render.schema.json` | `schema/agent-universal-render.yaml` | Consumption-oriented regrouping for rendering |
| agent-anthropic-render | `schemas/agent-anthropic-render.schema.json` | `schema/agent-anthropic-render.yaml` | Vendor-specific Anthropic platform format |

---

## 5. Consuming Tools

Tools that read verdandi data or its derived artifacts, with paths and how they use it.

| Tool | Path | How It Uses Verdandi |
|------|------|----------------------|
| **Draupnir** | `tools/draupnir/` | The compiler. Reads all verdandi YAML in dependency order (bottom-up through the Level enum), validates into frozen Pydantic models via `BeforeValidator` ref resolution, and generates JSON Schema files to `schemas/`. Supply-driven, level-ordered loading. IO in exactly two places: `loader.py` (reads) and `writer.py` (writes). |
| **Regin** | `tools/regin/` | The pipeline orchestrator. Consumes JSON Schemas generated by Draupnir to auto-generate Pydantic models (`generate_structures.py`). Runs the multi-step DAG pipeline transforming raw TOML definitions through resolution stages to universal format (levels 1-7) and vendor-specific render (level 8). Replaces the former `transform_to_universal` and `universal_to_canonical` tools. Does not read verdandi YAML directly -- consumes Draupnir's JSON Schema output. |
| **Galdr** | `tools/galdr/` | The render engine. Transforms universal-render format into final agent prompt files using Jinja2 templates. Reshapes sections, builds hook enforcement CLI commands from hooks_config data, composes frontmatter. Consumes Regin's universal-render output. |
| **Nornir** | `tools/nornir/` | Rust/PyO3 workspace with validation gates. Embeds generated JSON Schemas at compile time. Provides runtime validation of agent definition TOML files at every pipeline boundary. CLI check tools, writers, and dispatcher. |
| **inspect_schema.py** | `tools/inspect_schema.py` | Reads any generated JSON Schema and produces an interactive HTML inspection report showing field hierarchy, requirement status, extension metadata, polymorphic alternatives, and cross-group conditionals. Design decision D5: reads ONLY generated JSON Schema (not verdandi YAML), verifying what was actually produced. |
| **scaffold_definition.py** | `tools/scaffold_definition.py` | Reads `schemas/agent-raw-definition.schema.json` to produce TOML scaffold files with every field annotated by type, requirement status, and description. Placeholder values for authors to fill in. Consumes Draupnir output, not verdandi YAML directly. |

---

## 6. Directory Inventory

Complete listing of verdandi's subdirectories with current file counts and contents.

**Location:** `/Users/johnny/.ai/spaces/bragi/tools/verdandi/`

### Type Tier (structural constraint)

| Directory | Files | Description |
|-----------|-------|-------------|
| `pattern/` | 21 | Raw regex patterns. absolute, camel, claude, env, filename_template, has_whitespace, home, kebab, markdown, no_whitespace, pascal, path_extension, path_multi_segment, prose, relative, root, semver, snake, timestamp_zulu, title_case, traversal |
| `atom/` | 16 | Constrained base types (string/integer/boolean + pattern + exclusions). boolean, camel_string, filename_template, integer, kebab_string, pascal_string, path_absolute, path_relative, semver_string, snake_string, string_markdown, string_prose, string_text, string_token, timestamp_zulu, title_string |
| `enum/` | 19 | Enumerated allowed values. anthropic_model, anthropic_tools, capabilities, dispatch_background_mode, dispatch_input_delivery, dispatch_mode, enforcement_tool_format, input_format, instruction_mode, invocation_variant, notify_delivery, output_format_kind, output_name_known, param_type, permission_mode, processing_capability, return_mode, unix_command, write_frequency |
| `element/` | 3 | Atom/enum + runtime format contract. path_exists_absolute, path_exists_relative, path_exists_schema |
| `union/` | 4 | OneOf composition of atoms/enums/elements. path_any, path_exists_any, string_identifier, tool_or_command |
| `tuple/` | 1 | Fixed-length typed sequences (type composition, not a container). dispatch_batch_size |

### Field Membrane

| Directory | Files | Description |
|-----------|-------|-------------|
| `field/` | 136 | Where a type reference and a per-field description merge. Carries semantic meaning, x-extensions (semantic, constraint, validation), and type narrowing variants via double-underscore convention. |

### Container Tier -- Simple Containers (fields only, no nesting)

| Directory | Files | Description |
|-----------|-------|-------------|
| `simplearray/` | 27 | Variable-length arrays of one field type. Enforcement grants (explicit/implicit x read/write/edit), capabilities (requested/prohibited), security paths (raw/resolved x read/write), guardrails, display, frontmatter, success/failure evidence, role expertise. |
| `simplegroup/` | 24 | Flat objects with required/optional/conditional zones. context_item (raw/resolved), example_entry, failure_item, instruction_step, notify_item, parameter_item, success_item, tool_path_entry, command_path_entry, and various include_* types for @include references. |

### Container Tier -- Complex Containers (can nest other containers)

| Directory | Files | Description |
|-----------|-------|-------------|
| `array/` | 20 | Variable-length arrays of simplegroups. dispatch_parameters, execution_instructions (raw/resolved/reduced), success/failure_criteria (raw/resolved/reduced), context_required/available (raw/resolved), example_entries (raw/resolved/reduced), notify, tool_path_entries, command_path_entries. |
| `group/` | 27 | Objects containing fields + containers. metadata, role, enforcement grants (explicit/implicit), enforcement_output_tool, guardrails (raw/resolved/reduced), security IO/paths/schema/capabilities (raw/resolved), processing, return, example_group variants, dispatch, communication, context, context_resources, display_entry, hooks_config. |
| `superarray/` | 4 | Variable-length arrays of groups. execution_examples (raw/resolved/reduced), display_entries. |

### Top Level

| Directory | Files | Description |
|-----------|-------|-------------|
| `section/` | 15 | Complex container composition. criteria (raw/reduced/resolved + guardrails_reduced + success_reduced), enforcement, execution (raw/reduced/resolved + examples_reduced + instructions_reduced), indirect, security (raw/resolved), task. |
| `shallowsection/` | 17 | Fields + limited containers (max 2). anti_patterns, constraints, critical_rules, dispatcher, examples_reduced, frontmatter, identity, identity_anthropic, implicit_grants, input, instructions_reduced, output, return_format, security_boundary, security_boundary_anthropic, writing_output, writing_output_anthropic. |
| `schema/` | 13 | Root files. Deep schemas use `sections:` key, wide schemas use `shallowsections:` key (mutually exclusive). agent-raw-definition, agent-paths-resolved, agent-permissions-resolved, agent-execution-merged, agent-criteria-merged, agent-instructions-reduced, agent-examples-reduced, agent-guardrails-reduced, agent-success-reduced, agent-includes-merged, agent-universal-format, agent-universal-render, agent-anthropic-render. |

### Documentation files in this directory

| File | Purpose |
|------|---------|
| `QUICKSTART.md` | Operational quickstart guide |
| `COMPOSITION_RULES.md` | Formal specification of all levels |
| `MIGRATION_CHECKLIST.md` | Old toolcompose to verdandi file mapping |
| `CONTEXT_MAP.md` | This file |

**Total: 347 YAML files + 5 documentation files** (including PLAN_SCHEMA_ZONE_UNIFICATION.md)

---

## 7. Context Refresh Guide

For an LLM mid-session that needs to refresh understanding:

| If you need to... | Read this |
|--------------------|-----------|
| Understand what composition types exist and how they relate | `context/verdandi.md` -- the full context summary with 15-level hierarchy, file inventory, and composition rules |
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

**Dead types under review:**
- `atom/timestamp_zulu` -- zero consumers anywhere. No field, union, or element references it. Flagged for removal or documented future use.
- `union/string_identifier` -- zero consumers. Provisioned for future use or orphaned after refactor.

**Documentation staleness:** Four documentation files describe the pre-rebuild 8-level system and have not been updated to reflect the current 15-level verdandi hierarchy:
- `HIERARCHY_REBUILD_DESIGN.md` -- historical design record (12 levels, old names)
- `SCHEMA_COMPOSITION_EXPLAINED_BY_LLM_FOR_LLM.md` -- 8-level operational guide (old directory names, wrong file counts)
- `CRITICAL_DECISIONS.md` -- "What's Built" section describes old system (staleness note added)
- `DEFINITION_FORMAT.md` -- old type annotations (staleness note added)

These are pre-rebuild artifacts. Rather than rewriting historical documents (which would destroy their value as design records), the reconciliation preserved them as-is with staleness notes where appropriate.

**Draupnir enforces sections:/shallowsections: mutual exclusion** at compile time via the `exactly_one_section_type` model validator on the Schema class. A schema file must have exactly one of these keys — neither or both is a validation error.

**pattern/home portability:** The `pattern/home` regex is hardcoded to `/Users/johnny/`. This is a known limitation if definitions are ever used on another machine.

**ACTIVE REFACTOR: Schema Zone Unification** (branch: `refactor/schema-zone-unification`)

Three changes in progress — see `PLAN_SCHEMA_ZONE_UNIFICATION.md` for full details:
1. **Schema level gets required/optional/conditional zones** — replacing flat map + separate required/nullable lists. Nullable removed (functionally identical to optional).
2. **Presence-based conditionals** — `exists: true` predicate added to ConditionalWhen, enabling conditions based on property presence not just value matching.
3. **Leading-dot cross-section paths** — `field: .indirect.dispatch.dispatch_mode` (leading dot) disambiguates root traversal from same-object sibling references. Fixes ambiguity at schema root level.

Key files that change in Verdandi:
- All 13 `schema/*.yaml` files (zone restructure)
- 8 YAML files with cross-section conditionals (leading dot migration)
- `COMPOSITION_RULES.md` (schema level spec update)
