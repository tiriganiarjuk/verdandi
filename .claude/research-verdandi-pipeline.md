# Research: Verdandi in the End-to-End Pipeline

**Agent:** Claude Code (Research Collector)
**Timestamp:** 2026-03-20T00:00:00Z
**Status:** Complete

---

## Executive Summary

Verdandi is a typed composition hierarchy expressed as 348 YAML files organized into 15 levels across 4 architectural tiers. It is the single source of truth defining what fields exist in agent definitions, what types those fields have, and how they compose into nested objects and arrays. Rather than serving as an executable system, Verdandi functions as a domain-specific data format that feeds downstream tools: Draupnir (schema compiler), Nornir (validation gates), Regin (resolution pipeline), and Galdr (template renderer).

The architecture decouples what an agent IS (captured in Verdandi/definitions) from how it is expressed (vendor-specific formats). When platforms change, only format transformers update — definitions and the universal format remain stable. This document synthesizes the complete pipeline from YAML composition through multi-stage schema generation to final rendered prompts, with particular focus on how hierarchical constraints enable compile-time correctness guarantees.

---

## Part 1: The Complete Pipeline Architecture

### Pipeline Flow: From YAML to Rendered Prompt

The end-to-end pipeline implements a two-stage architecture separating deterministic resolution from vendor-specific rendering:

```
Verdandi YAML (tools/verdandi/, 348 files, 15 levels)
    |
    v
Draupnir (Python/Pydantic compiler)
    |
    +---> JSON Schemas (13 pipeline stages + 7 include fragments)
    |
    v
scaffold_definition.py  -----> TOML scaffolds (authoring templates)
inspect_schema.py       -----> HTML inspection reports
Nornir (Rust compilation) ----> Validation gates embedded at compile time
    |
    v
Author fills in TOML definition --> definitions/agents/{name}.toml (raw)
    |
    v
Regin (13-step DAG pipeline, Python)
    |
    +---> Level 1: Paths Verified
    +---> Level 2: Paths Resolved (absolutization)
    +---> Level 3: Includes Reduced (external files inlined)
    +---> Level 4: Permissions Resolved (minimum privilege calculation)
    +---> Level 5: Sections Merged (execution, criteria, examples, guardrails)
    +---> Level 6: Universal Format (convergence of two branches)
    +---> Level 7: Universal Render (regrouped for template consumption)
    +---> Level 8: Anthropic Render (vendor-specific lookup tables)
    |
    v
Galdr (Jinja2 template renderer) -- SCAFFOLDED, NOT YET OPERATIONAL
    |
    v
.claude/agents/{name}.md (deployed prompt file)
```

### Two-Stage Architecture Rationale

**Stage 1 (Levels 1-7: Regin Resolver)** produces a deterministic, vendor-agnostic universal format. This stage:
- Expands `@include` references to external files
- Absolutizes all file paths (relative → absolute)
- Computes minimum privilege sets from capability declarations
- Merges execution, criteria, and guardrails branches
- Regroups sections for optimal template consumption

The resolver stays "dumb" — it does not make behavioral decisions, only structural transformations. Intelligence lives in two places: the definition itself (what the agent IS) and type constraints (what is valid).

**Stage 2 (Level 8: Format Transformer)** applies vendor-specific logic. A thin adapter that:
- Maps capability tiers to platform model names (via `tier_to_model` lookup table)
- Discovers custom output tools from the tool registry
- Resolves capability verbs to platform tool names and bash commands
- Builds platform-specific frontmatter
- Computes enforcement hook configurations

The design anticipates multiple format transformers consuming the same universal format: `anthropic_subagent`, `anthropic_skill`, inspection, and future platforms.

### Schema Generation Cascade (Critical Sequence)

When Verdandi YAML changes, a mandatory rebuild cascade ensures downstream consumers stay synchronized:

```bash
# 1. Regenerate JSON Schemas from Verdandi YAML
draupnir --all

# 2. Rebuild Nornir (schemas embedded at compile time via include_str!())
cd tools/nornir && cargo build --release

# 3. Deploy validation gates and CLI check tools
./tools/nornir/deploy_gates.py

# 4. Regenerate Pydantic models in Regin
/tools/regin/generate_structures.py

# 5. (Optional) Regenerate Pydantic models in Galdr
/tools/galdr/generate_structures.py
```

**This cascade is not optional.** Modifying Verdandi YAML without running the full cascade means:
- Nornir validation gates are stale (schemas embedded at compile time)
- Regin pipeline models don't match generated schemas
- Scaffold and inspection tools produce incorrect output

The Nornir rebuild is especially critical because schemas are embedded via `include_str!()` — direct file system edits to JSON Schema files are silently overwritten by the next rebuild.

---

## Part 2: The 15-Level Composition Hierarchy

Verdandi organizes 348 YAML files into 15 levels across 4 architectural tiers, each with a specific purpose and composition constraints.

### Tier 1: Type Tier (Structural Constraints)

These levels define what values are structurally valid, enforced by pattern matching and enumeration.

| Level | Files | Purpose | Example |
|-------|-------|---------|---------|
| **pattern** | 21 | Raw regex patterns (building blocks) | `regex: "^[a-z0-9]+(-[a-z0-9]+)*$"` |
| **atom** | 18 | Constrained base types (string/integer/boolean + pattern + exclusions) | `type: string, pattern: pattern/kebab, excludes: [pattern/traversal]` |
| **enum** | 19 | Enumerated allowed values | `values: [batch, full, streaming]` |
| **element** | 3 | Atom/enum + runtime format contract | `inherits: atom/path_absolute, format: exists` |
| **union** | 4 | OneOf composition of atoms/enums/elements | `union: [atom/path_relative, atom/path_absolute]` |
| **tuple** | 1 | Fixed-length typed sequence | `items: [atom/integer, atom/integer], constraint: ascending_strict` |

**Key design principle:** Each level adds exactly one constraint layer. An atom builds on pattern. An element adds runtime semantics to atom. A union combines type-tier models. Containers reference fields, never raw types — the field level is mandatory.

### Tier 2: Field Membrane (Semantic Attachment)

| Level | Files | Purpose |
|-------|-------|---------|
| **field** | 138 | Where a type reference merges with a per-field description, semantic intent, and x-extensions |

The field membrane is the architectural innovation that recovered semantic meaning lost when toolcompose collapsed 60 unique primitives into 18 shared atoms. Each field carries:

- **type**: Reference to type tier (atom/, enum/, element/, union/, tuple/)
- **description**: What this field means in context
- **semantic** (optional): Quality/intent checks, anti-patterns, severity level
- **constraint** (optional): Cross-field/domain rules enabling validation at authoring time
- **default/const**: Default or constant values (emitted as JSON Schema defaults)

**Type narrowing with double underscore convention:** Pipeline stages narrow field types via naming:
```
field/context_path__any       --> raw stage (relative or absolute paths)
field/context_path__abs       --> resolved stage (absolute only)
field/enforcement_output__x_any  --> raw, with exists check
field/enforcement_output__x_abs  --> resolved, with exists check
```

The `x` prefix indicates `format: exists` — a runtime filesystem existence contract on top of pattern validation.

### Tier 3: Container Tier (Structural Arrangement)

These levels compose fields and other containers into objects and arrays.

**Simple Containers (fields only, no nested containers):**

| Level | Files | Purpose | Example |
|-------|-------|---------|---------|
| **simplearray** | 27 | Variable-length array of ONE field type | `items: field/role_expertise_item` |
| **simplegroup** | 24 | Flat object with required/optional/conditional zones | `required: {name: field/param_name}, optional: {desc: field/param_description}` |

**Complex Containers (fields + nested containers below them):**

| Level | Files | Purpose | Example |
|-------|-------|---------|---------|
| **array** | 20 | Variable-length array of simplegroups (supports oneOf for polymorphism) | `items: {oneOf: [simplegroup/instruction_step, simplegroup/include_instruction]}` |
| **group** | 27 | Object with separate fields section and containers section | `fields: {required: {...}}, containers: {optional: {...}}` |
| **superarray** | 4 | Variable-length array of groups (or simplegroups) | `items: {oneOf: [group/example_group, simplegroup/include_example]}` |

**Load order constraint:** Arrays load before groups. This enforces that arrays only reference simplegroups, while groups reference both simple containers AND arrays. The strictly downward reference model prevents circular dependencies.

### Tier 4: Top Level (Schema Composition)

| Level | Files | Purpose |
|-------|-------|---------|
| **section** | 15 | Named complex containers (no direct fields, only container aggregation) |
| **shallowsection** | 17 | Fields + limited containers (maximum 2 containers, prevents abuse) |
| **schema** | 13 | Root files using exactly one of: `sections:` (deep, domain-organized) or `shallowsections:` (wide, consumption-organized) |

All three levels use the same zone pattern: `required:`, `optional:`, and `conditional:` to declare membership semantics.

---

## Part 3: Composition Rules and Hierarchical Locking

### The Fundamental Composition Rule

Each level references ONLY levels below it in the load order. No sideways references. No upward references. This strict directionality prevents circular dependencies and enables supply-driven, level-ordered loading.

```
simplearray    accepts: field/
simplegroup    accepts: field/, simplearray/
array          accepts: simplegroup/
group fields   accepts: field/
group containers accepts: simplearray/, simplegroup/, array/
superarray     accepts: group/, simplegroup/
section        accepts: group/, array/, superarray/
shallowsection accepts: field/, simplearray/, array/, superarray/, group/
schema         accepts: section/ or shallowsection/ (mutually exclusive)
```

**Draupnir enforces this at compile time.** Violations produce explicit errors during schema generation, not silent bugs at runtime.

### Zone Pattern (Requirement Semantics)

Every container uses three zones:

- **`required:`** — Must be present. Appears in generated Pydantic models as non-optional.
- **`optional:`** — May be absent. Generated models use `| None = None`.
- **`conditional:`** — Presence depends on sibling or cross-section field values. Expressed as `if/then/else` in JSON Schema.

This zone pattern appears consistently at every level from simplegroup through schema, creating a predictable authoring experience.

### Hierarchical Locking (Mandatory Propagation)

A critical design decision addresses the risk of stale or missed mandatory requirements: **MANDATORY fields propagate upward with compiler failure, not silent auto-promotion.**

When an element declares `required: mandatory` (a compile-time lock), Draupnir verifies that:
1. Every group containing that element's field is required
2. Every section containing that group is required
3. Every schema containing that section is required

If any level violates this, Draupnir FAILS with an explicit error rather than auto-promoting groups to required status. This design choice prevents the masking of YAML errors that would otherwise remain silent.

**Why failure beats auto-promotion:** Auto-promotion papers over missing context. An LLM that loses architectural understanding can still produce YAML that "looks right" (proper nesting, correct field references) but violates implicit constraints. Compiler failure acts as a guardrail, redirecting the author back to understanding design intent rather than guessing at structure.

### Conditional Fields System

Conditionals control field presence based on sibling or cross-section values using three directional keywords:

- **`only_when`**: Required when condition true. FORBIDDEN when false.
- **`always_when`**: Required when condition true. No constraint when false.
- **`not_when`**: Forbidden when condition true. No constraint when false.

**Three predicate types (mutually exclusive per conditional):**

1. **Value-based const:** `equals: value` — single value match
2. **Value-based enum:** `in: [val1, val2]` — any of several values
3. **Presence-based:** `exists: true` — field is present in the object

**Cross-section conditionals use leading-dot convention:**

```yaml
# Sibling in same object (no dot)
conditional:
  dispatch_batch_size:
    ref: field/dispatch_batch_size
    only_when:
      field: dispatch_mode
      equals: batch

# Root traversal into different section (leading dot)
conditional:
  io_input_file:
    ref: field/io_input_file__abs
    only_when:
      field: .indirect.dispatch.dispatch_input_delivery
      equals: file
```

The leading dot unambiguously signals root traversal vs sibling reference, enabling both same-object and cross-section conditionals without heuristic guessing. At the schema root level, where "siblings" are complex sections rather than scalar fields, conditionals always use leading-dot paths (cross-section only).

---

## Part 4: The Raw / Resolved / Reduced Pipeline

Many types exist in three variants tracking agent definition processing stages:

| Variant | When | What It Contains |
|---------|------|------------------|
| **`_raw`** | Raw definition (authored) | Relative paths, include references to external files, loose references |
| **Resolved (no suffix)** | After path absolutization | All paths converted to absolute, includes still present but with absolute paths |
| **`_reduced`** | After include inlining | External file contents merged in, no more include references |

Example chain:
```
array/execution_instructions_raw.yaml        --> accepts inline + include refs
array/execution_instructions.yaml            --> (implicit: resolved paths)
array/execution_instructions_reduced.yaml    --> (implicit: includes inlined)

section/execution_raw.yaml
section/execution.yaml                       --> resolved paths
(no execution_reduced because section doesn't carry content)

shallowsection/instructions_reduced.yaml     --> final render stage (includes inlined)
```

Schema files compose these variants for each pipeline stage:
```
agent-raw-definition.yaml            --> all _raw sections
agent-paths-resolved.yaml            --> all resolved sections
agent-includes-merged.yaml           --> all reduced sections
agent-universal-format.yaml          --> final domain-organized format
agent-universal-render.yaml          --> regrouped for template rendering
agent-anthropic-render.yaml          --> vendor-specific Anthropic format
```

This staging enables three critical capabilities:
1. **Include expansion with path absolutization** — includes reference external files with relative paths; after resolution, all paths are absolute
2. **Late binding of content** — definitions author with relative paths and file references; resolver makes them absolute and inline content
3. **Multi-stage validation** — each pipeline stage validates against a schema matching that stage's constraints (raw uses looser types, resolved uses tighter types)

---

## Part 5: Schema Zone Unification and Conditional Improvements

### Three Recent Architectural Improvements (Completed)

A coordinated update unified schema-level structure with the rest of the hierarchy and extended the conditional system:

#### 1. Schema Zone Unification

**Before:** Schema level used a different structural pattern than every other level:
```yaml
required:      # separate list
  - dispatcher
  - identity
nullable:      # another separate list
  - implicit_grants
shallowsections:  # flat map
  dispatcher: shallowsection/dispatcher
```

**After:** Schema uses the same zone pattern as other levels:
```yaml
shallowsections:
  required:
    dispatcher: shallowsection/dispatcher
    identity: shallowsection/identity
  optional:
    implicit_grants: shallowsection/implicit_grants
    examples: shallowsection/examples_reduced
```

**Justification:** `nullable` was functionally dead weight. Generated Pydantic models collapse nullable and optional into `SomeSection | None = None`. TOML (the authoring format) cannot express null anyway. Unifying the pattern makes schema authoring predictable.

#### 2. Presence-Based Conditionals

**New predicate:** `exists: true` enables conditions based on field presence, not just value matching:

```yaml
conditional:
  resource_timeout:
    ref: field/resource_timeout
    only_when:
      field: resource_url
      exists: true         # NEW: resource_timeout required when resource_url exists
```

Directional keywords combine naturally:
- `only_when` + `exists: true` = required when present, forbidden when absent
- `always_when` + `exists: true` = required when present, no constraint when absent
- `not_when` + `exists: true` = forbidden when present, no constraint when absent

This works at every level: simplegroup, group, shallowsection, section, and schema.

#### 3. Leading-Dot Cross-Section Paths

**New convention:** Leading dot means "start from root." No dot means "sibling in same object."

```yaml
# Sibling (no dot)
conditional:
  dispatch_batch_size:
    ref: field/dispatch_batch_size
    only_when:
      field: dispatch_mode            # sibling in same group
      equals: batch

# Root traversal (leading dot)
conditional:
  some_field:
    ref: field/some_field
    only_when:
      field: .indirect.dispatch.dispatch_input_delivery   # traversal from root
      equals: file
```

At schema root level, where "siblings" are sections (complex objects, not scalar fields), the convention disambiguates:
```yaml
# Sibling section (presence check at root)
conditional:
  security_details:
    ref: shallowsection/security_details
    only_when:
      field: enforcement         # sibling: check if enforcement section exists
      exists: true

# Traversal into a section
conditional:
  something:
    ref: shallowsection/something
    only_when:
      field: .enforcement.enforcement_output_tool.tool_format  # traversal: deep path
      exists: true
```

This unambiguously routes conditionals to the correct builder (same-section vs cross-section), enabling schema-level conditional sections.

---

## Part 6: Design Discipline (D10 and the Composition Rules)

### Composition Discipline: Seven Rules Against Decomposition-for-Decomposition

Design Decision D10 codifies seven rules preventing unnecessary fragmentation:

1. **Single-consumer files are fragmentation.** A type consumed by exactly one parent is not composition — it's an unexplained intermediate. Merge it into the parent unless you can articulate why it's separate.

2. **Same-folder imports are a red flag.** If a file in `group/` references another `group/` type, you have a peer dependency. Rethink the structure.

3. **Do not redefine shared types with bespoke descriptions.** If `atom/string_text` exists and meets your need, use `field/my_field` referencing that atom with a field-level description. Creating a new atom with a similar-but-different description is siloed reinvention.

4. **The reuse test:** Explain why this file is separate without saying "because composition." If you can't articulate a reason (e.g., "shared by 4 different groups," "variant for a specific pipeline stage"), it's not a reuse boundary.

5. **Shared primitives use `requirement: inherit`.** When a group contains an element marked `required: mandatory`, that group becomes required in any section containing it. This propagates upward; don't fight the hierarchy.

6. **Import from below, never sideways or up.** Directional references only. Enforce tier boundaries.

7. **Pragmatic exceptions are acceptable when documented.** Not every edge case fits the rules. When you break a rule, document why in a comment. Future LLMs reading your YAML should understand your reasoning.

### Verdandi File Counts and Maturity

| Level | Files | Notes |
|-------|-------|-------|
| pattern | 21 | Stable — regex patterns rarely change |
| atom | 18 | Stable — core type definitions |
| enum | 19 | Stable — enumerated values for dispatch, permissions, modes |
| element | 3 | Stable — path existence contract mostly |
| union | 4 | Stable — path combinations |
| tuple | 1 | Minimal — only batch size pair |
| field | 138 | Active — most frequent edits here (semantic intent, constraints) |
| simplearray | 27 | Active — arrays for enforcement grants, capabilities, guardrails |
| simplegroup | 24 | Active — inline vs include variants for every content type |
| array | 20 | Active — dispatch parameters, instructions, examples, criteria |
| group | 27 | Stable core, active metadata — most domain knowledge lives here |
| superarray | 4 | Minimal — example groups and display entries |
| section | 15 | Stable — structural organization (indirect, task, security, execution, criteria, enforcement) |
| shallowsection | 17 | Active — render-stage regrouping for template consumption |
| schema | 13 | Active — pipeline stages and render stages |

**Total: 348 files**

---

## Part 7: Draupnir Schema Compiler

### What Draupnir Does

Draupnir is a Python/Pydantic compiler that:

1. **Reads all 348 Verdandi YAML files** in dependency order (bottom-up through the Level enum)
2. **Validates each file** against its level's specification using Pydantic `BeforeValidator` for ref resolution
3. **Freezes models** as immutable (ConfigDict frozen=True) to prevent accidental mutation
4. **Generates JSON Schema output** — 13 pipeline stage schemas + 7 include fragment schemas

### Key Design Points

**Supply-driven loading:** Draupnir processes levels in order: patterns first, then atoms, then enums, etc. When a file is loaded, all its dependencies (types it references) are already loaded and frozen.

**Ref resolution via BeforeValidator:** References like `atom/path_absolute` are not strings — they are replaced with the actual frozen Pydantic model during `model_validate()`. This happens in a `BeforeValidator` hook, not in a separate resolution step. Loading IS the pipeline.

**Compile-time enforcement:** Tier boundaries, mandatory element propagation, conditional validity, type consistency, and schema mutual exclusion are all enforced at compile time. Bad YAML fails immediately with explicit error messages.

**Two export paths:**
- **Shared types:** Collected in `$defs` (JSON Schema definitions)
- **Schema top-level:** Properties built from zones, required fields collected, conditionals built as `if/then/else`

---

## Part 8: Integration with Downstream Tools

### Nornir (Validation Gates)

**What it does:** Runtime validation of TOML definitions against generated schemas using compiled Rust validators.

**How it connects to Verdandi:** Draupnir generates JSON Schema files. Nornir embeds these schemas at compile time via `include_str!()`. When a Rust gate is built, it includes the JSON Schema directly. If Verdandi YAML changes, Nornir must rebuild (`cargo build --release`) to pick up new schemas.

**Critical dependency:** Nornir gates validate at every Regin pipeline boundary:
- Raw definition validation
- Paths verified
- Universal format
- Anthropic render

Stale Nornir builds accept invalid definitions.

### Regin (13-Step Resolution Pipeline)

**What it does:** Transforms raw TOML definitions through 13 stages, applying deterministic transformations and calling Nornir gates at boundaries.

**How it connects to Verdandi:**
- Regin's `generate_structures.py` reads JSON Schemas generated by Draupnir
- Auto-generates Pydantic models matching each pipeline stage's schema
- Consumes definitions conforming to these models
- Applies transformations (path absolutization, include inlining, permissions calculation)
- Validates intermediate outputs against stage-specific schemas

**If Verdandi YAML changes:** Draupnir generates new schemas, Regin regenerates models, Nornir rebuilds gates.

### Scaffold and Inspection Tools

**scaffold_definition.py:** Reads `agent-raw-definition.schema.json` and generates TOML authoring templates with every field annotated by type, requirement status, and description.

**inspect_schema.py:** Reads any pipeline stage schema and generates HTML inspection reports showing field hierarchy, requirement status, x-extensions, polymorphic alternatives, and cross-group conditionals. Updated to handle presence-based conditionals (`exists` predicates).

Both tools prove that Draupnir output is correct — if scaffolds or inspection reports are wrong, Draupnir failed to represent the intended schema.

### Galdr (Template Renderer)

**Current status:** Scaffolded, not yet operational.

**Design:** Consumes Regin's `universal_render.toml` output (level 7) and applies Jinja2 templates to produce final `.md` prompt files.

**Template variants:** Three `writing_output` template variants selected by variant field:
- **standard:** Single-record write tool with hardcoded heredoc
- **batch:** Pipe-to-stdin with batch_size configuration
- **wrapper:** Tool name only, no paths (simpler invocation)

Templates do not contain conditionals/loops — format transformers (Regin level 8) decide structure. Templates are pure presentation: headings, formatting, ordering, placeholder substitution.

---

## Part 9: Old Toolcompose to Verdandi Migration

### Migration Strategy

The pre-rebuild `toolcompose` system had ~55 broken YAML files and a flat type structure. The Verdandi rebuild addressed:

1. **Type proliferation:** ~60 unique primitive types collapsed into 18 atoms, recovered semantic meaning through the field membrane
2. **Composition fragmentation:** Replaced 8-level flat hierarchy with 15-level membrane architecture
3. **Path handling:** Raw/resolved/reduced staging with late binding of content
4. **Conditional system:** Unified syntax across all levels, added presence-based conditions, leading-dot cross-section paths

### Key Mappings

| Old Concept | New Verdandi | Notes |
|-------------|-------------|-------|
| Primitive types (~60 unique) | Atoms (18) + field descriptions | Reusable base types, semantic meaning on field |
| Composed types | Unions (4) | Explicit OneOf composition |
| Type groups | Simplegroups, groups, sections | Explicit container types |
| Raw/resolved | field__any vs field__abs | Double-underscore type narrowing |
| Includes | Array/group variants with `include_*` simplegroups | Late binding, include expansion in pipeline |

The migration preserved domain structure (indirect, task, security, execution, criteria) while reorganizing internal composition to prevent circular dependencies and enable compile-time validation.

---

## Part 10: Template and Rendering Architecture

### Template Design Philosophy

**Templates are pure presentation.** They do NOT contain:
- Conditionals or loops
- Field extraction logic
- Structural decisions

**Format transformers (Regin level 8) decide:**
- Which fields to include and in what order
- Whether sections are present
- Hook configurations
- Capability-to-tool mappings

**Templates receive:** Pre-structured, vendor-ready data with all decisions made. They inject data into fixed placeholders.

### Current Template Variants

Located in `definitions/templates/`:

| Template | Purpose |
|----------|---------|
| `agent.md` | Standard agent with per-record write tool |
| `agent_batch.md` | Agent with batch write tool (pipe-to-stdin) |
| `agent_wrapper_write.md` | Agent with simplified wrapper write tool |
| `dispatch.md` | Dispatcher/skill with argument handling and state assessment |

### Tool Registry

`tools/regin/tool_registry.toml` lists deployed Nornir writer tools. During Regin level 8 (anthropic resolve), the `tool_discovery` module matches:
- `enforcement_output_tool.output_format` (json vs jsonl)
- `enforcement_output_tool.write_frequency` (per-record, batched, etc.)
- `enforcement_output_tool.schema_path` (output validation schema)
- File/directory paths

On mismatch, `CustomToolNotFound` is raised with a `ToolBuildRequest` containing everything Nornir needs to build the missing tool.

---

## Part 11: Critical Design Decisions Summary

### D1: Three-Tier Field Requirement Model

Replaces binary REQUIRED/OPTIONAL with: MANDATORY (global), REQUIRED (local group consistency), OPTIONAL, plus conditional variants (ONLY_WHEN, ALWAYS_WHEN, NOT_WHEN).

### D2: Field-Level Intelligence

Fields carry semantic intent, constraint rules, and validation hints. Groups are pure clustering. The `field/` membrane recovers semantic meaning lost in type consolidation.

### D3: Mandatory Propagation with Fail-and-Report

MANDATORY fields force containing groups and sections to be required. Draupnir fails with explicit error rather than auto-promoting. Auto-promotion masks YAML errors.

### D4: Platform-Agnostic Capability Model

Vendor-specific model names replaced with descriptive tiers: `fast-cheap`, `standard`, `analytical`, `complex-reasoning`, `max-capability`, `inherit`. Mapping lives in format transformers (vendor lookup tables).

### D5: Schema Inspector (Independent Verification)

`inspect_schema.py` reads ONLY generated JSON Schema — independent verification of what the compiler produced, not what YAML intended. Disagreement reveals generator bugs.

### D6: Two-Level Path Permissions

Replace 6 per-capability path arrays with 2 levels: read-level (read, search, inspect) and write-level (all capabilities). Intersection with requested capabilities determines actual grants.

### D7: Platform-Agnostic Output Tool Model

`security_tools` section eliminated. Processing concerns live in `task.processing`. Enforcement contains only `enforcement_output_tool` (computed, only for json/jsonl) and `enforcement_grants`.

### D8: Return Mode Spectrum

`return_mode` enum: `[status, status-metrics, metrics-output, output]`. Three const instruction strings carry behavioral instructions; format transformers compose appropriate combinations.

### D9: Dispatch Consolidation

`io_contract` eliminated. Dispatcher-consumed operational fields in dispatch section; security-relevant declarations in security section.

### D10: Composition Discipline

Seven rules preventing decomposition-for-decomposition's-sake. See earlier section for details.

### D11: Security Schema Group

Output schema, input schema, and embed flag in security section. Resolver needs this for engineering control derivation.

---

## Part 12: Maturity Assessment

### What's Proven (Production-Ready)

- **Type tier (pattern/atom/enum):** Stable, comprehensive coverage of agent definition needs
- **Field level:** Semantic descriptions and constraint annotations working correctly
- **Container tier (simplegroup/group/array):** Hierarchical nesting, conditional fields, oneOf polymorphism all functional
- **Section and shallowsection:** Top-level composition working, zone pattern established
- **Draupnir compiler:** Generates correct JSON Schemas, enforces tier boundaries, handles includes
- **Raw/resolved/reduced staging:** Include expansion, path absolutization, late binding working
- **Regin levels 1-7 (resolver):** Path verification, permissions calculation, universal format production proven
- **Nornir gates:** Runtime validation at pipeline boundaries
- **Scaffold generation:** TOML templates from schemas working
- **Schema inspection:** HTML reports with requirement analysis and conditional detection

### What's Designed But Incomplete

- **Regin level 8 (anthropic resolve):** Core logic implemented (model resolution, tool discovery, grant explosion, hook composition), but not yet systematically tested
- **Galdr (template renderer):** Architecture designed (Jinja2 templates, placeholder substitution), scaffolded but not operational
- **Conditional sections:** Schema-level section conditionals designed, implemented, but not yet used (no current schema needs conditional sections)
- **Presence-based conditionals:** Extended conditional system (exists predicate, leading-dot paths) implemented, tested at group level, broader adoption pending use cases

### What's Aspirational (Design Phase)

- **Multiple format transformers:** Current implementation supports Anthropic only. Design anticipates `opencode_subagent`, `anthropic_skill`, `inspection` as future format transformers
- **Vendor-specific gates:** Future enhancement to embed vendor-specific validation rules (e.g., Anthropic model availability checks)
- **Skill (dispatcher) definitions:** YAML structure designed, not yet integrated into Regin pipeline

---

## Part 13: Critical Insights and Implications

### Insight 1: Composition Hierarchy Prevents Circular Dependencies

The strictly-downward reference model (each level references only lower levels) prevents cycles. Supply-driven loading (load types before use them) enables single-pass compilation. This is why Draupnir can validate all 348 files in one pass without iterative resolution. Contrast with previous toolcompose where circular references caused loading failures — Verdandi's hierarchy enforcement prevented that class of error entirely.

**Implication:** Any proposed new level must fit into the ordering. "I want to add a level between group and section" requires proving it doesn't create cycles when groups reference it and sections reference groups. Without this proof, the proposal is architecturally invalid.

### Insight 2: The Field Membrane Recovers Local Semantics Lost in Type Consolidation

When toolcompose's 60 primitives collapsed into 18 atoms, per-field descriptions were discarded. The field level solved this by making each field carry its own semantic intent, constraint rules, and anti-patterns. This is why `field/agent_description` can differ from `field/role_description` despite both referencing `atom/string_prose` — the semantic meaning is at field level, not type level.

**Implication:** Adding a new field always requires creating a `field/` file. Never reference raw type-tier files from containers. This is a non-negotiable rule because it's where semantic meaning lives.

### Insight 3: Hierarchical Locking Via Mandatory Propagation Prevents Stale Constraints

When an element declares `required: mandatory`, Draupnir verifies propagation upward. If a group containing that element is NOT required, compilation fails. This prevents the silent degradation of constraints — a definition author who loses context cannot accidentally weaken a mandatory requirement.

**Implication:** Mandatory is a contract, not a suggestion. Once set, it cannot be demoted without Draupnir failure. This is intentional guardrail design. Auto-promotion would hide mistakes; failure redirects the author back to understanding.

### Insight 4: Zone Pattern Produces Consistent Authoring Experience

Every container from simplegroup through schema uses the same `required:`, `optional:`, `conditional:` pattern. Conditional fields appear at the same level as required/optional fields, not in separate sections. This consistency reduces cognitive load and prevents nesting mistakes.

**Implication:** New container types must use the zone pattern. If you're adding a new level, implement `required/optional/conditional` zones if it contains any references.

### Insight 5: Presence-Based Conditionals Enable Natural Composition

Value-based conditionals (equals/in) work for modes and enums, but many composition decisions are naturally about presence: "if this object exists, then that field is required." The `exists: true` predicate extends the conditional system to match natural composition patterns.

**Implication:** Don't force value-based conditions where presence is the natural constraint. Use `exists: true` and let the schema builder emit `if: { required: [...] }`.

### Insight 6: Leading-Dot Convention Resolves Schema-Root Ambiguity

At schema root level, "siblings" are sections (complex objects). Without a convention, `field: security` is ambiguous — is it the sibling section (presence check) or a traversal into that section? Leading dot (`.security.io.output_path`) unambiguously signals root traversal.

**Implication:** Schema-level conditionals always use leading dots. Group-level conditionals use no dot for siblings. This is not style — it's a semantic signal that routes to the correct builder.

---

## Part 14: Connections and Dependencies

### Verdandi → Draupnir → Schemas (Critical Path)

Verdandi YAML is the source of truth. Draupnir generates JSON Schemas from it. All downstream consumers depend on these schemas:

- **Nornir:** Embeds schemas at compile time
- **Regin:** Auto-generates Pydantic models from schemas
- **Scaffold:** Reads schemas to generate authoring templates
- **Inspect:** Reads schemas to generate HTML reports

Changing Verdandi requires: `draupnir --all`, then rebuild all downstream tools.

### Regin → Galdr (Render Pipeline)

Regin produces `universal_render.toml` (level 7), which Galdr consumes. Galdr is scaffolded but non-operational. Once operational:
- Regin outputs structured data (sections, fields, lists)
- Galdr applies templates to produce final prompt
- Templates remain simple (substitution, not logic)

### Nornir Gates at Boundaries

Regin calls Nornir gates at 4 pipeline boundaries:
1. Raw definition validation (catches schema violations early)
2. Universal format validation (catches transformation errors)
3. Anthropic render validation (catches vendor-specific issues)

### Tool Registry Loop

Regin level 8 discovers tools from `tool_registry.toml`. If a tool is missing:
- Raises `CustomToolNotFound` with `ToolBuildRequest`
- Operator runs `generate_writer.py` to build missing tool
- Nornir compiles new writer crate
- Returns to Regin with tool available

---

## Part 15: Recommendations

### For Active Development

1. **Extend semantic annotations:** The `semantic` field carries intent checks and anti-patterns. Expand coverage to every field — this is where type safety moves from structural (patterns match) to semantic (field satisfies intent).

2. **Operationalize Galdr:** Template rendering is blocked. Unblock by:
   - Building missing Jinja2 templates
   - Testing template rendering against sample `universal_render.toml`
   - Deploying as `.claude/agents/` final artifact

3. **Document Regin level 8:** Anthropic resolve logic (model resolution, tool discovery, grant explosion, hook composition) is implemented but under-documented. Comprehensive documentation would help future maintainers.

4. **Add schema-level conditional section tests:** Conditional sections are designed and implemented but untested in practice. Create test schemas using conditional sections to verify the routing to cross-section builders.

### For Future Enhancement

1. **Multiple format transformers:** Design and implement `anthropic_skill`, `opencode_subagent`, and `inspection` format transformers consuming the same universal format.

2. **Vendor-specific gates:** Extend Nornir with vendor-specific validation. Example: Anthropic model names must be currently supported.

3. **Integrate skill definitions:** Design complete Regin pipeline for dispatch definitions (currently in `definitions/skills/`, not pipeline-integrated).

4. **Expand conditional predicates:** Current system supports value and presence. Future predicates could enable: string length, array size, pattern matching within conditions.

### For Maintenance

1. **Keep COMPOSITION_RULES.md current:** This is the authoritative spec. Any Verdandi change must update it.

2. **Update context documents after schema changes:** Stale documentation actively causes LLMs to revert new features to match old docs. Update concurrently with code.

3. **Run full cascade after Verdandi edits:** The schema generation cascade is mandatory. Partial updates create sync issues. Document in QUICKSTART.md.

4. **Monitor hierarchical locking effectiveness:** Mandatory propagation failure is a guardrail. When it triggers, investigate whether the failure is correct (design intent) or false positive (hierarchy misunderstanding).

---

## Conclusion

Verdandi is a mature, compile-time-validated type system that decouples what an agent IS (captured in definitions and types) from how it is expressed (vendor-specific formats). Its 15-level hierarchy, enforced through downward references and supply-driven loading, enables single-pass compilation with no circular dependencies. The field membrane recovers semantic meaning lost in type consolidation, while hierarchical locking prevents silent constraint degradation.

The pipeline from Verdandi YAML through Draupnir to Regin to Galdr implements a clean separation between deterministic resolution (what is true regardless of platform) and vendor-specific rendering (how platforms express it). When vendors change formats or add features, only format transformers update — definitions and universal format remain stable.

The system is production-ready for raw-to-universal-format pipeline stages and vendor rendering at Anthropic. Future work should focus on operationalizing Galdr, extending format transformers to support multiple platforms, and enriching semantic annotations to push type safety into intent checking.

