# Research: Verdandi Core System Architecture

**Researcher:** Claude Code Agent
**Timestamp:** 2026-03-20T00:00:00Z
**Status:** Complete
**Focus:** Complete structural specification of the Verdandi typed composition hierarchy

---

## Executive Summary

Verdandi is a 15-level typed composition hierarchy comprising ~348 YAML files across 4 tiers, organized into a membrane architecture. It defines the structural vocabulary for agent tool definitions — what fields exist, what types they have, and how they compose into containers, sections, and complete schemas. The system enforces strict downward-only reference rules, zone-based requirement semantics, and conditional field presence based on sibling and cross-section values. Draupnir compiles this YAML into 13 JSON Schema files that validate agent definitions throughout the pipeline. The architecture achieves type reuse through shared atoms while recovering per-field semantic meaning through a field-level membrane that sits between raw types and container structures.

---

## 1. The 15-Level Hierarchy: Load Order and Tier Assignment

Verdandi organizes types into a strict 15-level hierarchy with load-order enforcement. Each level can reference only levels below it in this sequence. Draupnir enforces these boundaries at compile time.

### Complete Load Order

```
1.  pattern           (Tier 1 - Type Tier)
2.  atom             (Tier 1)
3.  enum             (Tier 1)
4.  element          (Tier 1)
5.  union            (Tier 1)
6.  tuple            (Tier 1)
7.  field            (Field Membrane - architectural boundary)
8.  simplearray      (Tier 2 - Container Tier, simple)
9.  simplegroup      (Tier 2)
10. array            (Tier 3 - Container Tier, complex)
11. group            (Tier 3)
12. superarray       (Tier 3)
13. section          (Tier 4 - Top Level)
14. shallowsection   (Tier 4)
15. schema           (Tier 4, root)
```

### Tier Definitions

**Tier 1 — Type Tier (Levels 1-6): Structural Constraint**
- What values are valid
- No composition of types upward — types only reference lower types or stand alone
- 6 levels establishing type constraints from raw patterns to composed unions and tuples

**Field Membrane (Level 7): Architectural Boundary**
- Mandatory transition layer where type references merge with per-field descriptions
- Carries x-extension metadata (semantic, constraint, validation)
- Lowest import level for all containers
- Highest inclusion level for all types
- Enables type reuse at the type tier while preserving per-field semantic meaning

**Tier 2 — Container Tier (Levels 8-9): Simple Containers**
- Fields organized into requirement zones, no nesting of containers
- Can contain field references and tuples only
- 27 simplearray files + 24 simplegroup files = 51 total

**Tier 3 — Container Tier (Levels 10-12): Complex Containers**
- Can nest other containers below them
- Three distinct container shapes: array (variable simplegroups), group (fields + containers), superarray (variable groups)
- 20 array files + 27 group files + 4 superarray files = 51 total

**Tier 4 — Top Level (Levels 13-15): Schema Organization**
- Section (15 files): composes complex containers only, no direct fields
- Shallowsection (17 files): composes fields + limited containers (max 2)
- Schema (13 files): root — must use exactly one of `sections:` (deep) or `shallowsections:` (wide)

### Directory Inventory (348 YAML files total)

| Level | Directory | Files | Purpose |
|-------|-----------|-------|---------|
| 1 | `pattern/` | 21 | Raw regex patterns (absolute, camel, claude, env, filename_template, has_whitespace, home, kebab, markdown, no_whitespace, pascal, path_extension, path_multi_segment, prose, relative, root, semver, snake, timestamp_zulu, title_case, traversal) |
| 2 | `atom/` | 16 | Constrained base types (boolean, camel_string, filename_template, integer, kebab_string, pascal_string, path_absolute, path_relative, semver_string, snake_string, string_markdown, string_prose, string_text, string_token, timestamp_zulu, title_string) |
| 3 | `enum/` | 19 | Enumerated allowed values (anthropic_model, anthropic_tools, capabilities, dispatch_background_mode, dispatch_input_delivery, dispatch_mode, enforcement_tool_format, input_format, instruction_mode, invocation_variant, notify_delivery, output_format_kind, output_name_known, param_type, permission_mode, processing_capability, return_mode, unix_command, write_frequency) |
| 4 | `element/` | 3 | Atom/enum + runtime format contract (path_exists_absolute, path_exists_relative, path_exists_schema) |
| 5 | `union/` | 4 | OneOf composition of atoms/enums/elements (path_any, path_exists_any, string_identifier, tool_or_command) |
| 6 | `tuple/` | 1 | Fixed-length typed sequences (dispatch_batch_size: [integer, integer]) |
| 7 | `field/` | 136 | Type reference + per-field description + x-extensions (semantic, constraint, validation) |
| 8 | `simplearray/` | 27 | Variable-length arrays of one field type |
| 9 | `simplegroup/` | 24 | Flat objects with required/optional/conditional zones |
| 10 | `array/` | 20 | Variable-length arrays of simplegroups (with oneOf for polymorphism) |
| 11 | `group/` | 27 | Objects with fields section + containers section |
| 12 | `superarray/` | 4 | Variable-length arrays of groups (with oneOf for polymorphism) |
| 13 | `section/` | 15 | Complex container composition only (no direct fields) |
| 14 | `shallowsection/` | 17 | Fields + limited containers (max 2) |
| 15 | `schema/` | 13 | Root — `sections:` (deep) XOR `shallowsections:` (wide) |

**Total: 347 YAML files + 5 documentation files (CLAUDE.md, CONTEXT_MAP.md, COMPOSITION_RULES.md, QUICKSTART.md, PLAN_SCHEMA_ZONE_UNIFICATION.md)**

---

## 2. Type Tier Specification (Levels 1-6)

### Level 1: pattern/

**Purpose:** Raw regex patterns reusable across atoms

**YAML Structure:**
```yaml
# pattern/relative.yaml
title: Relative                                    # required: PascalCase name
regex: "^\."                                       # required: regex pattern
description: Starts with dot (relative path)      # required: explanation
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name for Python class generation |
| regex | yes | string | Regular expression pattern |
| description | yes | string | Human-readable explanation for scaffolding |

**References:**
- Referenced by: atoms (via `pattern:` and `excludes:` lists)

---

### Level 2: atom/

**Purpose:** Constrained base types — string/integer/boolean with patterns and exclusions

**YAML Structure:**
```yaml
# atom/path_relative.yaml
title: PathRelative                               # required
type: string                                      # required: string|integer|boolean
pattern: pattern/relative                         # optional: one pattern reference
excludes: [pattern/traversal, pattern/env]        # optional: list of patterns to reject
min_length: 2                                      # optional: minimum string length
description: Relative path starting with ./       # required
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| type | yes | enum | `string`, `integer`, or `boolean` |
| pattern | no | string | Single `pattern/` reference (strings only) |
| excludes | no | list | List of `pattern/` references (strings only) |
| min_length | no | integer | Minimum string length (JSON Schema `minLength`) |
| description | yes | string | Human-readable explanation |

**Constraint Logic:**
- `pattern:` is a POSITIVE constraint (must match)
- `excludes:` is a NEGATIVE constraint (must NOT match any)
- They work together: atom must match `pattern` AND must not match any `excludes` patterns

**References:**
- Referenced by: elements, unions, fields (via `type:`)

---

### Level 3: enum/

**Purpose:** Enumerated allowed values

**YAML Structure:**
```yaml
# enum/dispatch_mode.yaml
title: DispatchMode                               # required
values: [batch, full]                             # required: list of allowed values
description: How dispatcher feeds input           # required
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| values | yes | list | List of allowed values (strings, integers, or booleans) |
| description | yes | string | Human-readable explanation |

**References:**
- Referenced by: elements, unions, fields (via `type:`)

---

### Level 4: element/

**Purpose:** Atom or enum PLUS runtime format contract

**YAML Structure:**
```yaml
# element/path_exists_relative.yaml
title: PathExistsRelative                         # required
inherits: atom/path_relative                      # required: atom/ or enum/
format: exists                                    # optional: runtime validation keyword
required: mandatory                               # optional: compile-time lock
description: Relative path that must exist       # required
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| inherits | yes | string | Reference to `atom/` or `enum/` |
| format | no | string | Runtime validation keyword (e.g., `exists`, `exists_schema`) |
| required | no | enum | Only value: `mandatory`. Draupnir verifies this element appears in every schema using it |
| description | yes | string | Human-readable explanation |

**Format Contract:**
- `format:` bridges schema-time and runtime validation
- Schema enforces pattern + exclusions from inherited atom
- Nornir enforces the format contract (e.g., filesystem existence check)

**References:**
- Referenced by: unions, fields (via `type:`)

---

### Level 5: union/

**Purpose:** OneOf composition — combines multiple type-tier models

**YAML Structure:**
```yaml
# union/path_any.yaml
title: PathAny                                    # required
union: [atom/path_relative, atom/path_absolute]  # required: list of type refs
description: Filesystem path — relative or       # required
  absolute
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| union | yes | list | List of `atom/`, `enum/`, or `element/` references |
| description | yes | string | Human-readable explanation |

**Constraint:**
- Pure composition — no additional constraints
- Members must be type-tier models only (atom, enum, or element)
- No sideways references (cannot reference other unions)

**References:**
- Referenced by: field/ (via `type:`), tuple/ (via `items:`)

---

### Level 6: tuple/

**Purpose:** Fixed-length typed sequence (type composition, not a container)

**YAML Structure:**
```yaml
# tuple/dispatch_batch_size.yaml
title: DispatchBatchSize                          # required
items: [atom/integer, atom/integer]               # required: ordered list of type refs
constraint: ascending_strict                      # optional: validation rule
description: >                                    # required
  [min, max] records per batch. Min < max.
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| items | yes | list | Ordered list of type-tier references (atom/enum/element/union) |
| constraint | no | string | Validation rule (e.g., `ascending_strict`) |
| description | yes | string | Human-readable explanation |

**Properties:**
- Length is implicit from `items` array length
- Each position can reference a different type-tier model
- Represents a specific ordered structure (not a variable-length array)

**References:**
- Referenced by: field/ (via `type:`)

---

## 3. Field Membrane Architecture (Level 7)

The field level is the **mandatory architectural boundary** between type constraint (Tier 1) and container arrangement (Tiers 2-4). It is where:
- Type references merge with per-field semantic descriptions
- X-extension metadata (semantic, constraint, validation) attach
- Type narrowing via double-underscore suffixes enables pipeline-stage-specific type constraints

### Field Membrane Property

```
TYPE TIER (Levels 1-6): structural constraint
  pattern → atom → enum → element → union, tuple

                    ┃ field ┃  ← THE MEMBRANE

CONTAINER TIER (Levels 8-15): structural arrangement
  simplearray → simplegroup → array → group →
  superarray → section → shallowsection → schema
```

**Critical Rule:** Containers reference `field/` files, NEVER raw type-tier files directly. This is enforced by Draupnir at compile time.

### YAML Structure

**Minimal field:**
```yaml
# field/agent_description.yaml
title: AgentDescription                           # required: PascalCase
type: atom/string_prose                           # required: type-tier reference
description: >                                    # required: per-field meaning
  One-sentence purpose statement. Appears in
  scaffolding, inspection reports, identity.
```

**Field with x-extensions:**
```yaml
# field/agent_description.yaml
title: AgentDescription
type: atom/string_prose
min_length: 40                                    # optional: field-level length override
description: >
  One-sentence purpose statement. Describes WHAT
  the agent does, not HOW.
semantic:                                         # optional x-extension
  intent: >
    Must describe WHAT the agent does, not HOW.
    Focus on user-facing capability.
  checks:
    - Describes a concrete end state
  anti_patterns:
    - Implementation details instead of capability
  severity: error
constraint:                                       # optional x-extension
  - rule: required_when
    field: .task.processing.output_file_format
    in: [jsonl, json]
    severity: error
    reason: >
      Structured output (JSON/JSONL) requires
      a schema for validation.
```

### Field-Level YAML Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name for Python class generation |
| type | yes | string | Reference to type tier: `atom/`, `enum/`, `element/`, `union/`, or `tuple/` |
| description | yes | string | Per-field semantic meaning — what this field represents in context |
| default | no | any | Default value (string, integer, or boolean). Emitted as JSON Schema `default` |
| const | no | string | Constant value. Emitted as JSON Schema `const` |
| min_length | no | integer | Minimum string length override (applies to this field only) |
| semantic | no | object | Quality/intent checks — emitted as `x-semantic` in JSON Schema |
| constraint | no | list | Cross-field/domain rules — emitted as `x-constraint` in JSON Schema |

### Semantic Extension Structure

```yaml
semantic:
  intent: Primary purpose / semantic intent
  checks:
    - Validation criteria 1
    - Validation criteria 2
  anti_patterns:
    - Known anti-pattern 1
    - Known anti-pattern 2
  severity: error|warning
```

| Subfield | Required | Description |
|----------|----------|-------------|
| intent | no | Primary purpose and semantic intent for the field |
| checks | no | List of validation criteria (strings) |
| anti_patterns | no | List of known anti-patterns to avoid (strings) |
| severity | no | Severity level (`error` or `warning`) |

### Constraint Extension Structure

```yaml
constraint:
  - rule: required_when
    field: .task.processing.output_file_format
    in: [jsonl, json]
    severity: error
    reason: Why this constraint exists
```

| Subfield | Required | Description |
|----------|----------|-------------|
| rule | yes | Constraint type (e.g., `required_when`) |
| severity | yes | Severity level |
| reason | yes | Explanation of why this constraint exists |
| field | no | Target field path (supports leading-dot cross-section paths) |
| order | no | Ordering metadata |
| in | no | List of values for enum-style checks |

### Double-Underscore Type Narrowing

**Pattern:** When the same logical field needs different type constraints at different pipeline stages, the field name carries a double-underscore suffix indicating the type variant.

**Example:**
```
field/context_path__any.yaml     type: union/path_any          (raw stage: relative or absolute)
field/context_path__abs.yaml     type: atom/path_absolute      (resolved stage: absolute only)
field/context_path__x_any.yaml   type: union/path_exists_any   (raw stage: must exist, any format)
field/context_path__x_abs.yaml   type: element/path_exists_abs (resolved stage: must exist, absolute)
```

| Suffix | Meaning | Type Reference | Pipeline Stage |
|--------|---------|----------------|---|
| `__any` | Accepts relative or absolute paths | `union/path_any` | Raw (pre-resolution) |
| `__abs` | Absolute paths only | `atom/path_absolute` | Resolved (post-resolution) |
| `__x_any` | Path must exist, relative or absolute | `union/path_exists_any` | Raw (pre-resolution) |
| `__x_abs` | Path must exist, absolute only | `element/path_exists_absolute` | Resolved (post-resolution) |

The `x` prefix means "exists" — the field references an `element/` type with `format: exists`, adding a runtime filesystem existence contract.

**Application:**
- The base name (`context_path`, `include_instruction`) identifies the logical field
- The suffix identifies the type narrowing for that pipeline stage
- Different schemas reference the same logical field with stage-appropriate type constraints
- This allows loose types (relative/absolute) to drop away after path resolution

---

## 4. Container Tier Specification (Levels 8-12)

### Level 8: simplearray/

**Purpose:** Variable-length array of **one field type**

**YAML Structure:**
```yaml
# simplearray/role_expertise.yaml
title: RoleExpertise                              # required
items: field/role_expertise_item                  # required: single field/ reference
description: Agent areas of expertise             # required
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| items | yes | string | Single `field/` reference — no oneOf |
| description | yes | string | Human-readable explanation |

**Constraint:**
- Simplearrays do NOT support `oneOf` — items must be a single `field/` reference
- For polymorphic arrays, use `array/` or `superarray/`

**Downward References:**
- Accepts: `field/` only

---

### Level 9: simplegroup/

**Purpose:** Flat object with fields organized into requirement zones

**YAML Structure:**
```yaml
# simplegroup/parameter_item.yaml
title: ParameterItem                              # required
description: Named parameter at dispatch time    # required
required:                                         # optional: required zone
  param_name: field/param_name
  param_type: field/param_type
optional:                                         # optional: optional zone
  param_description: field/param_description
conditional:                                      # optional: conditional zone
  param_sensitive:
    ref: field/param_sensitive
    only_when:
      field: param_type
      in: [password, api_key]
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| description | yes | string | Human-readable explanation |
| required | no | map | Field names → `field/` or `simplearray/` references |
| optional | no | map | Field names → `field/` or `simplearray/` references |
| conditional | no | map | Conditional field entries (see Conditionals section) |

**Zone Logic:**
- At least one of `required`, `optional`, or `conditional` must exist
- `required` zone: fields must be present
- `optional` zone: fields may be omitted
- `conditional` zone: presence depends on sibling field values or presence

**Downward References:**
- Accepts: `field/`, `simplearray/` only
- Tuples accessed through `field/` refs (a field file with `type: tuple/...`)

---

### Level 10: array/

**Purpose:** Variable-length array of simplegroups (with oneOf support for polymorphism)

**YAML Structure:**
```yaml
# array/dispatch_parameters.yaml
title: DispatchParameters
items: simplegroup/parameter_item
description: Named parameters agent accepts

# array/execution_instructions.yaml (polymorphic)
title: ExecutionInstructions
items:
  oneOf: [simplegroup/instruction_step, simplegroup/include_instruction]
description: Ordered processing steps
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| items | yes | string or object | `simplegroup/` reference OR `oneOf: [simplegroup/...]` |
| description | yes | string | Human-readable explanation |

**Polymorphic Arrays:**
- `items: simplegroup/single_type` — homogeneous array
- `items: { oneOf: [simplegroup/type1, simplegroup/type2] }` — polymorphic array with known types

**Downward References:**
- Accepts: `simplegroup/` only (for items)

---

### Level 11: group/

**Purpose:** Object containing fields PLUS containers. Has two separate sections: `fields:` and `containers:`

**YAML Structure - Fields Only:**
```yaml
# group/metadata.yaml
title: Metadata
description: Agent identity and version
fields:
  required:
    agent_name: field/agent_name
    agent_description: field/agent_description
    semantic_version: field/semantic_version
```

**YAML Structure - Containers Only:**
```yaml
# group/enforcement_explicit_grants.yaml
title: EnforcementExplicitGrants
description: Author-defined capability grants
containers:
  required:
    read: simplearray/enforcement_explicit_read
    search: simplearray/enforcement_explicit_search
    write: simplearray/enforcement_explicit_write
```

**YAML Structure - Fields + Containers + Conditionals:**
```yaml
# group/dispatch.yaml
title: Dispatch
description: Dispatcher configuration
fields:
  required:
    dispatch_mode: field/dispatch_mode
    dispatch_background_mode: field/dispatch_background_mode
    dispatch_input_format: field/dispatch_input_format
    dispatch_input_delivery: field/dispatch_input_delivery
    dispatch_input_description: field/dispatch_input_description
  optional:
    dispatch_max_agents: field/dispatch_max_agents
  conditional:
    dispatch_batch_size:
      ref: field/dispatch_batch_size
      only_when:
        field: dispatch_mode
        equals: batch
containers:
  optional:
    dispatch_parameters: array/dispatch_parameters
```

**Fields:**
| Section | Required | Type | Description |
|---------|----------|------|-------------|
| title | yes | string | PascalCase name |
| description | yes | string | Human-readable explanation |
| fields | no | object | `required`/`optional`/`conditional` zones with `field/` references |
| containers | no | object | `required`/`optional`/`conditional` zones with container references |

**Zone Logic:**
- At least one of `fields` or `containers` must exist
- Both may be present
- Field zones reference: `field/`
- Container zones reference: `simplearray/`, `simplegroup/`, `array/`

**Downward References:**
- `fields:` section accepts: `field/`
- `containers:` section accepts: `simplearray/`, `simplegroup/`, `array/`

---

### Level 12: superarray/

**Purpose:** Variable-length array of groups (or simplegroups), with oneOf support

**YAML Structure:**
```yaml
# superarray/execution_examples.yaml
title: ExecutionExamples
items:
  oneOf: [group/example_group, simplegroup/include_example_group]
description: Example groups — inline or included

# superarray/display_entries.yaml
title: DisplayEntries
items: group/display_entry
description: Security boundary display entries
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| items | yes | string or object | `group/` ref OR `oneOf: [group/...]` |
| description | yes | string | Human-readable explanation |

**Polymorphic Superarrays:**
- `items: group/single_type` — homogeneous array
- `items: { oneOf: [group/type1, simplegroup/type2] }` — mixed group and simplegroup types

**Downward References:**
- Accepts: `group/`, `simplegroup/` (for items)

---

## 5. Top-Level Specification (Levels 13-15)

### Level 13: section/

**Purpose:** Composes complex containers only. No direct fields.

**YAML Structure:**
```yaml
# section/execution.yaml
title: Execution
description: >
  How the agent processes work — includes paths
  absolutized
containers:
  required:
    execution_instructions: array/execution_instructions
    execution_examples: superarray/execution_examples

# section/security.yaml
title: Security
description: Access control after path resolution
containers:
  required:
    security_io: group/security_io
    security_schema: group/security_schema
    security_paths: group/security_paths
    context: group/context
  optional:
    security_capabilities: group/security_capabilities
```

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| description | yes | string | Human-readable explanation |
| containers | yes | object | `required`/`optional`/`conditional` zones with complex container refs |

**Containers Zones:**
- `required:` zone — must be present
- `optional:` zone — may be omitted
- `conditional:` zone — presence conditional on sibling or cross-section values

**Downward References:**
- Accepts: `group/`, `array/`, `superarray/` (via containers zones)

---

### Level 14: shallowsection/

**Purpose:** Composes fields alongside containers. Field refs live at the top level (not nested under `fields:` key). Maximum 2 containers.

**YAML Structure:**
```yaml
# shallowsection/identity.yaml
title: Identity
description: Agent identity — who it is, tier
required:
  title: field/agent_title
  description: field/agent_description
  role_identity: field/role_identity
optional:
  role_description: field/role_description
containers:
  optional:
    role_expertise: simplearray/role_expertise

# shallowsection/dispatcher.yaml (with conditional field)
title: Dispatcher
description: Everything dispatcher needs
required:
  agent_name: field/agent_name
  dispatch_mode: field/dispatch_mode
optional:
  max_agents: field/dispatch_max_agents
conditional:
  batch_size:
    ref: field/dispatch_batch_size
    only_when:
      field: dispatch_mode
      equals: batch
containers:
  optional:
    parameters: array/dispatch_parameters
```

**Fields:**
| Section | Required | Type | Description |
|---------|----------|------|-------------|
| title | yes | string | PascalCase name |
| description | yes | string | Human-readable explanation |
| required | no | map | Field names → `field/` references |
| optional | no | map | Field names → `field/` references |
| conditional | no | map | Conditional field entries (see Conditionals) |
| containers | no | object | `required`/`optional`/`conditional` zones with container refs |

**Container Limit:**
- Maximum 2 containers total per shallowsection
- This prevents shallowsections from becoming improper deep-composition vessels
- Use section/ + group/ for deeper nesting

**Downward References:**
- Top-level zones (required/optional/conditional) accept: `field/`
- `containers:` zones accept: `simplearray/`, `array/`, `superarray/`, `group/`

---

### Level 15: schema/

**Purpose:** Root file. Entry point to agent definitions. Uses exactly one of two composition keys.

**Mutual Exclusion Rule:** A schema must have exactly one of `sections:` or `shallowsections:`. Never both. Never neither. Draupnir enforces this at compile time.

**Both keys use the same zone pattern:** `required:`, `optional:`, `conditional:` sub-maps determine requirement semantics. There are no separate top-level `required:` or `nullable:` lists.

#### Deep Schema (sections:)

For domain-organized pipeline stages (raw → resolved → reduced → universal):

```yaml
# schema/agent-raw-definition.yaml
title: Agent Raw Definition
version: 3
description: Authored agent definition as written by humans
sections:
  required:
    indirect: section/indirect
    task: section/task
    security: section/security_raw
    execution: section/execution_raw
    criteria: section/criteria_raw
  optional:
    enforcement: section/enforcement
```

#### Wide Schema (shallowsections:)

For consumption-organized render stages:

```yaml
# schema/agent-universal-render.yaml
title: Agent Universal Render
version: 1
description: >
  Consumption-oriented regrouping of
  universal-format for rendering
shallowsections:
  required:
    dispatcher: shallowsection/dispatcher
    identity: shallowsection/identity
    input: shallowsection/input
    instructions: shallowsection/instructions_reduced
    output: shallowsection/output
    return_format: shallowsection/return_format
    security_boundary: shallowsection/security_boundary
  optional:
    implicit_grants: shallowsection/implicit_grants
    examples: shallowsection/examples_reduced
    writing_output: shallowsection/writing_output
    constraints: shallowsection/constraints
    anti_patterns: shallowsection/anti_patterns
```

#### Schema with Conditional Sections

```yaml
shallowsections:
  required:
    identity: shallowsection/identity
    input: shallowsection/input
  optional:
    examples: shallowsection/examples_reduced
  conditional:
    writing_output:
      ref: shallowsection/writing_output
      only_when:
        field: .enforcement.enforcement_output_tool.tool_format
        exists: true
```

**Schema-Level Conditionals Are Always Cross-Section:**
- At the schema root level, "siblings" are sections (complex objects), not leaf fields
- Every schema-level conditional MUST use a dotted path with leading dot to reach into a section to find a field
- This is not a limitation — the cross-section builder handles these naturally

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| title | yes | string | PascalCase name |
| version | yes | integer | Schema version number |
| description | yes | string | Human-readable explanation |
| sections | mutual | object | `required`/`optional`/`conditional` zones with `section/` refs (deep composition) |
| shallowsections | mutual | object | `required`/`optional`/`conditional` zones with `shallowsection/` refs (wide composition) |

**Downward References:**
- `sections:` zones accept: `section/`
- `shallowsections:` zones accept: `shallowsection/`

---

## 6. Conditional Field Presence System

Conditionals control field, container, or section presence based on sibling or cross-section values and presence checks. They appear in `conditional:` zones at every level that supports zones (simplegroup, group, shallowsection, section, schema).

### Conditional Structure

Every conditional entry has:
1. A `ref:` pointing to the type being conditionally required
2. Exactly one directional keyword: `only_when`, `always_when`, or `not_when`
3. Exactly one predicate inside the directional keyword: `equals`, `in`, or `exists`

```yaml
conditional:
  field_name:
    ref: field/some_field
    only_when:
      field: trigger_field
      equals: some_value
```

### Directional Keywords

| Keyword | Meaning | Effect |
|---------|---------|--------|
| `only_when` | if and only if | Required when condition TRUE, FORBIDDEN when FALSE |
| `always_when` | conditional required | Required when condition TRUE, no constraint when FALSE |
| `not_when` | inverse condition | Forbidden when condition TRUE, no constraint when FALSE |

### Predicates

Each condition uses **exactly one predicate** (mutually exclusive):

```yaml
equals: value           # const check — matches a single value
in: [value1, value2]    # enum check — matches any of several values
exists: true            # presence check — field is present in object
```

**Important:** `exists: false` is NOT supported. Use `not_when` with `exists: true` for the inverse.

### Value-Based Conditionals (Const Check)

```yaml
conditional:
  dispatch_batch_size:
    ref: field/dispatch_batch_size
    only_when:
      field: dispatch_mode
      equals: batch
```

Semantics:
- `only_when` + `equals: batch` = required when dispatch_mode == 'batch', forbidden otherwise
- `always_when` + `equals: batch` = required when dispatch_mode == 'batch', no constraint otherwise
- `not_when` + `equals: batch` = forbidden when dispatch_mode == 'batch', no constraint otherwise

### Value-Based Conditionals (Enum Check)

```yaml
conditional:
  return_status:
    ref: field/return_status
    only_when:
      field: return_mode
      in: [status, status-metrics]
```

Semantics:
- Required when return_mode is ANY of the listed values
- Forbidden/unconstrained based on directional keyword

### Presence-Based Conditionals (Existence Check)

```yaml
conditional:
  resource_timeout:
    ref: field/resource_timeout
    only_when:
      field: resource_url
      exists: true
```

Semantics:
- `only_when` + `exists: true` = required when resource_url is present, forbidden when absent
- `always_when` + `exists: true` = required when resource_url is present, no constraint when absent
- `not_when` + `exists: true` = forbidden when resource_url is present, no constraint when absent

### Cross-Section Paths with Leading Dot

The `field:` value in a conditional uses two conventions:

**No dot** — same-object sibling reference:
```yaml
# Both dispatch_mode and dispatch_batch_size are in the same group
conditional:
  dispatch_batch_size:
    ref: field/dispatch_batch_size
    only_when:
      field: dispatch_mode         # sibling in same group
      equals: batch
```

**Leading dot** — root traversal (cross-section):
```yaml
# io_input_file is in one group, indirect.dispatch.dispatch_input_delivery
# is in a different section entirely
conditional:
  io_input_file:
    ref: field/io_input_file__abs
    only_when:
      field: .indirect.dispatch.dispatch_input_delivery  # root traversal
      equals: file
```

**Semantics:**
- No dot: "Look in this object's siblings"
- Leading dot: "Start from the schema root and traverse down"

The leading dot disambiguates same-object references from root traversal, which is critical at the schema root level where siblings are sections (complex objects).

### Where Conditionals Appear

| Level | Conditional Host | Ref Type |
|-------|------------------|----------|
| simplegroup | top-level zones | `field/`, `simplearray/` |
| group fields | `fields:` section zones | `field/` |
| group containers | `containers:` section zones | `simplearray/`, `simplegroup/`, `array/` |
| shallowsection | top-level zones | `field/` |
| section containers | `containers:` section zones | `group/`, `array/`, `superarray/` |
| schema sections | `sections:` zones | `section/` |
| schema shallowsections | `shallowsections:` zones | `shallowsection/` |

---

## 7. Downward-Only Reference Rule

**Fundamental Law:** Each level references only levels below it in load order. Sideways and upward references are compile-time errors.

### Reference Matrix

| Level | Can Reference |
|-------|---------------|
| `pattern/` | (none) |
| `atom/` | `pattern/` |
| `enum/` | (none) |
| `element/` | `atom/`, `enum/` |
| `union/` | `atom/`, `enum/`, `element/` |
| `tuple/` | `atom/`, `enum/`, `element/`, `union/` |
| `field/` | `atom/`, `enum/`, `element/`, `union/`, `tuple/` |
| `simplearray/` | `field/` |
| `simplegroup/` | `field/`, `simplearray/` |
| `array/` | `simplegroup/` |
| `group fields` | `field/` |
| `group containers` | `simplearray/`, `simplegroup/`, `array/` |
| `superarray/` | `group/`, `simplegroup/` |
| `section` | `group/`, `array/`, `superarray/` |
| `shallowsection` | `field/`, `simplearray/`, `array/`, `superarray/`, `group/` |
| `schema sections:` | `section/` |
| `schema shallowsections:` | `shallowsection/` |

### Enforcement

Draupnir's supply-driven level-ordered loading validates at compile time:
1. Attempts to reference sideways → compilation error
2. Attempts to reference upward → compilation error
3. Referenced type doesn't exist → compilation error
4. Circular dependencies → impossible due to downward-only rule

---

## 8. The Raw / Resolved / Reduced Pipeline

Many container types exist in two or three variants tracking stages of **agent definition processing** (not Verdandi processing — Verdandi is static data). These variants enable looser constraints early and stricter constraints later.

### Three Stages

**Raw (`_raw` suffix)**
- What the author writes in TOML definitions
- Accepts relative paths and include references to external files
- Example: `array/execution_instructions_raw.yaml`

**Resolved (no suffix)**
- After path resolution
- All paths absolutized
- Includes still present but with absolute paths
- Example: `array/execution_instructions.yaml`

**Reduced (`_reduced` suffix)**
- After include inlining
- External file contents merged inline
- No more include references
- Example: `array/execution_instructions_reduced.yaml`

### Common Types with Variants

| Logical Type | Raw | Resolved | Reduced |
|---|---|---|---|
| context_item | `simplegroup/context_item_raw` | `simplegroup/context_item` | — |
| execution_instructions | `array/execution_instructions_raw` | `array/execution_instructions` | `array/execution_instructions_reduced` |
| success_criteria | `array/success_criteria_raw` | `array/success_criteria` | `array/success_criteria_reduced` |
| failure_criteria | `array/failure_criteria_raw` | `array/failure_criteria` | `array/failure_criteria_reduced` |
| security (section) | `section/security_raw` | `section/security` | — |
| execution (section) | `section/execution_raw` | `section/execution` | — |
| criteria (section) | `section/criteria_raw` | `section/criteria` | — |

### Schema Composition by Pipeline Stage

```yaml
# Raw stage — accepts relative paths, include refs
schema/agent-raw-definition.yaml
  sections:
    required:
      security: section/security_raw
      execution: section/execution_raw

# Resolved stage — paths absolutized
schema/agent-paths-resolved.yaml
  sections:
    required:
      security: section/security
      execution: section/execution

# Reduced stage — includes inlined
schema/agent-universal-format.yaml
  sections:
    required:
      execution: section/execution  (with instructions_reduced, examples_reduced)
      criteria: section/criteria    (with success/failure_criteria_reduced)

# Render stage — regrouped for presentation
schema/agent-universal-render.yaml
  shallowsections:
    required:
      instructions: shallowsection/instructions_reduced
      examples: shallowsection/examples_reduced
```

### Field Type Narrowing by Stage

```yaml
# Raw — accept relative or absolute paths
field/context_path__any.yaml
  type: union/path_any

# Resolved — absolute paths only
field/context_path__abs.yaml
  type: atom/path_absolute

# Raw with existence check — must exist, relative or absolute
field/include_instruction__x_any.yaml
  type: union/path_exists_any

# Resolved with existence check — must exist, absolute
field/include_instruction__x_abs.yaml
  type: element/path_exists_absolute
```

---

## 9. Zone System (required / optional / conditional)

**Zone Pattern:** Every container level uses the same three-zone structure to declare requirement semantics. Zone membership determines whether a field/container is required, optional, or conditional.

### Zone Semantics

| Zone | Requirement | TOML Behavior |
|------|-------------|---|
| `required:` | Must be present | Author must provide a value |
| `optional:` | May be omitted | Author may leave out; generated as `None`/default |
| `conditional:` | Presence conditional | Required/forbidden/unconstrained based on trigger field |

### Zone Structure

```yaml
# Generic zone structure used at every level
required:
  field_or_container_name: field/reference
  another_field: field/another_reference

optional:
  optional_field: field/optional_reference

conditional:
  conditional_field:
    ref: field/conditional_reference
    only_when:
      field: trigger_field
      equals: trigger_value
```

### Generated Pydantic Behavior

**Required field:**
```python
class MyGroup(BaseModel):
    field_name: str  # no default, must be provided
```

**Optional field:**
```python
class MyGroup(BaseModel):
    field_name: str | None = None  # default None
```

**Conditional field:**
```python
class MyGroup(BaseModel):
    conditional_field: str | None = None  # always optional in model
    # Constraint enforced at validation time, not in type signature
```

### Zone Constraints

**Simplegroup/Group/Shallowsection:**
- At least one of `required`, `optional`, or `conditional` must exist

**Group with Both Sections:**
- `fields:` section and `containers:` section both optional
- But at least one must exist
- Each section can independently have `required`, `optional`, `conditional` zones

**Schema:**
- Both `sections:` and `shallowsections:` follow the zone pattern
- But exactly one key must be present (mutual exclusion)
- Each has `required`, `optional`, `conditional` zones

---

## 10. Compile-Time Enforcement Rules

Draupnir validates these constraints at compile time:

### 1. Tier Boundary Enforcement
- Every reference points to a level at or below the source level in load order
- Sideways references rejected
- Upward references rejected

### 2. No Same-Level References
- A simplegroup cannot reference another simplegroup
- A group cannot reference a section
- Draupnir rejects these during supply-driven loading

### 3. Mandatory Elements
- Elements with `required: mandatory` must appear in every schema
- Draupnir verifies this at compile time

### 4. Conditional Validity
- `only_when`/`always_when`/`not_when` reference fields that:
  - Exist in the same container (no dot), OR
  - Use leading-dot paths for cross-section root traversal
- Invalid references rejected during compilation

### 5. Type Consistency
- Reference targets exist and are the correct level for their position
- Type mismatches rejected

### 6. Schema Mutual Exclusion
- Exactly one of `sections:` or `shallowsections:` must be present
- Zero or two causes compilation error
- Enforced by `exactly_one_section_type` validator

### 7. Shallowsection Container Limit
- Maximum 2 containers total per shallowsection
- Limit violation rejected

### 8. Zone Invariants
- At least one zone must be populated where required
- Empty zones rejected

### 9. Conditional Predicate Exclusivity
- Exactly one of `equals`, `in`, or `exists` per conditional
- Multiple or zero predicates cause validation error

### 10. Exists Predicate Constraint
- `exists: true` is valid
- `exists: false` is forbidden (use `not_when` with `exists: true` instead)

---

## 11. X-Extension Metadata Patterns

X-extensions are per-field metadata that travel with the field through the container hierarchy. They enable LLM-driven quality guidance without breaking JSON Schema validation.

### Three Extension Types

All emitted to JSON Schema with `x-` prefix:

| Extension | JSON Key | Purpose | Location |
|-----------|----------|---------|----------|
| `semantic:` | `x-semantic` | Quality/intent checks (non-schema-validatable) | `field/` level |
| `constraint:` | `x-constraint` | Cross-field/domain rules | `field/` level |
| `validation:` | `x-validation` | Runtime validation hints | `field/` level |

### Semantic Extension

```yaml
# field/agent_description.yaml
semantic:
  intent: Must describe WHAT the agent does, not HOW
  checks:
    - Describes a concrete end state, not ongoing activity
    - Refers to process succeeding, not data quality
    - Observable from outputs
  anti_patterns:
    - Confusing process success with data quality
    - Implementation details instead of capability
  severity: error
```

**JSON Schema Emission:**
```json
{
  "$ref": "#/$defs/atom__string_prose",
  "description": "One-sentence purpose statement...",
  "x-semantic": {
    "intent": "Must describe WHAT the agent does, not HOW",
    "checks": [...],
    "anti_patterns": [...],
    "severity": "error"
  }
}
```

### Constraint Extension

```yaml
# field/output_schema__x_abs.yaml
constraint:
  - rule: required_when
    field: .task.processing.output_file_format
    in: [jsonl, json]
    severity: error
    reason: >
      Structured output (JSON or JSONL) requires
      a schema for validation.
```

**JSON Schema Emission:**
```json
{
  "x-constraint": [
    {
      "rule": "required_when",
      "field": ".task.processing.output_file_format",
      "in": ["jsonl", "json"],
      "severity": "error",
      "reason": "Structured output..."
    }
  ]
}
```

### Validation Extension

```yaml
# field/some_field.yaml
validation:
  instruction: Should not simply restate the file format
  severity: warning
```

---

## 12. Schema-Specific Rules (sections vs shallowsections)

### Mutual Exclusion Enforcement

A schema file contains **exactly one** of:
- `sections:` — for deep, domain-organized composition
- `shallowsections:` — for wide, consumption-organized composition

Draupnir's `exactly_one_section_type` validator enforces this at compile time.

### Deep Schema (sections:)

**Purpose:** Domain-organized pipeline stages (logical grouping by processing phase)

**Structure:**
```yaml
sections:
  required:
    section_name: section/reference
  optional:
    optional_section: section/reference
  conditional:
    conditional_section:
      ref: section/reference
      only_when: ...
```

**References:** `section/` types only

**Example:** agent-raw-definition schema groups security, execution, criteria, task, indirect by domain

### Wide Schema (shallowsections:)

**Purpose:** Consumption-oriented render stages (logical grouping by who reads it)

**Structure:**
```yaml
shallowsections:
  required:
    shallowsection_name: shallowsection/reference
  optional:
    optional_shallowsection: shallowsection/reference
  conditional:
    conditional_shallowsection:
      ref: shallowsection/reference
      only_when: ...
```

**References:** `shallowsection/` types only

**Example:** agent-universal-render schema groups by rendering audience (identity, dispatcher, input, instructions, output)

### Zone Pattern at Schema Level

Both `sections:` and `shallowsections:` use identical zone pattern to every other level:

```yaml
sections:
  required: {...}      # must be present
  optional: {...}      # may be omitted
  conditional: {...}   # conditional on cross-section values
```

### Schema-Level Conditionals Are Always Cross-Section

At schema root level, "siblings" are sections — complex objects, not leaf fields. Every schema-level conditional MUST use a leading-dot path to reach into a section:

```yaml
shallowsections:
  required:
    identity: shallowsection/identity
  optional:
    writing_output: shallowsection/writing_output
  conditional:
    writing_output:
      ref: shallowsection/writing_output
      only_when:
        field: .enforcement.enforcement_output_tool.tool_format
        exists: true
```

The path `.enforcement.enforcement_output_tool.tool_format` starts from root, traverses into enforcement section, then enforcement_output_tool group, then reaches the tool_format field.

---

## 13. Draupnir Compilation Pipeline

Verdandi YAML → Draupnir compiler → JSON Schema files (13 outputs)

### Compilation Stages

1. **Supply-Driven Level-Ordered Loading**
   - Loads all ~348 files in exact load order (pattern → schema)
   - Each level's files freeze into Pydantic models
   - Later levels can reference lower levels in BeforeValidator

2. **Validation Into Frozen Pydantic Models**
   - Each level has a Pydantic model class
   - BeforeValidator resolves file path references to actual objects
   - Frozen=True prevents mutations

3. **JSON Schema Generation**
   - Processes through exporter to JSON Schema format
   - Emits `$defs` deduplication for type tier
   - Combines type constraints with field descriptions
   - Adds x-extension metadata

4. **13 Generated Schema Files**
   - agent-raw-definition — authored TOML entry point
   - agent-paths-resolved — after path absolutization
   - agent-permissions-resolved — after permission calculation
   - agent-execution-merged — after execution section merge
   - agent-criteria-merged — after criteria section merge
   - agent-instructions-reduced — after instruction inlining
   - agent-examples-reduced — after example inlining
   - agent-guardrails-reduced — after guardrails inlining
   - agent-success-reduced — after success/failure criteria inlining
   - agent-includes-merged — after all includes merged
   - agent-universal-format — final pipeline-agnostic format
   - agent-universal-render — consumption-oriented regrouping
   - agent-anthropic-render — vendor-specific Anthropic format

### Error Handling

Any validation failure at any level causes the entire compile to fail. Errors specify:
- File path with issue
- Level name
- Specific constraint violated
- Suggested fix where applicable

---

## 14. File Path Reference Semantics

**Fundamental:** Every reference is a file path. The file path IS the identity.

### Reference Format

```
level/filename_without_extension
```

Examples:
```
field/agent_description         references  field/agent_description.yaml
group/dispatch                  references  group/dispatch.yaml
section/security_raw            references  section/security_raw.yaml
shallowsection/identity         references  shallowsection/identity.yaml
```

### Naming Conventions

**Type-Tier Files (pattern through tuple):**
- PascalCase in YAML (`title: PathAbsolute`)
- snake_case filename (`path_absolute.yaml`)
- Derived name matches filename (no separate `name:` field needed)

**Field Files:**
- PascalCase in YAML (`title: AgentDescription`)
- snake_case filename (`agent_description.yaml`)
- Type narrowing: `field_name__variant.yaml`

**Container Files:**
- PascalCase in YAML (`title: ExecutionInstructions`)
- snake_case filename (`execution_instructions.yaml`)
- Variants for pipeline stages: `type_raw.yaml`, `type.yaml`, `type_reduced.yaml`

**Section/Shallowsection/Schema Files:**
- PascalCase in YAML (`title: Security`)
- snake_case filename (`security.yaml`)
- Variants: `security_raw.yaml`, `security.yaml`

### No Inline Type Declarations

Everything above the pattern level must be a file reference. No inline objects, no anonymous types.

**Forbidden:**
```yaml
# DO NOT DO THIS
fields:
  required:
    some_field:
      type: string
      description: Inline type
```

**Correct:**
```yaml
# CREATE field/some_field.yaml, then reference it
fields:
  required:
    some_field: field/some_field
```

---

## 15. Maturity Assessment

### What's Implemented and Proven

- **15-level hierarchy** — fully defined, deployed, load order enforced
- **Downward-only reference rule** — compile-time enforcement working
- **Zone system** (required/optional/conditional) — used across all container levels
- **Field membrane architecture** — recovering per-field semantics, x-extensions working
- **Double-underscore type narrowing** — deployed for path field variants (__any, __abs)
- **Raw/Resolved/Reduced variants** — tracking agent definition processing stages
- **Conditional field presence** — value-based (equals, in) predicates working
- **Cross-section conditionals** — working with dotted paths
- **Simplearray/Simplegroup/Array** — proven in deployments
- **Group dual-section** (fields + containers) — stable
- **Section composition** — stable
- **Shallowsection composition** — stable, container limit enforced
- **Schema mutual exclusion** — sections XOR shallowsections enforced
- **Draupnir compiler** — supply-driven loading, Pydantic models, JSON Schema export working
- **13 generated schemas** — covering all pipeline stages
- **Type reuse via $defs** — working, atoms/enums deduplicated

### Recent Enhancements (Schema Zone Unification Plan)

- **Presence-based conditionals** — `exists: true` predicate added (Phase 2 complete)
- **Leading-dot convention** — disambiguates same-object from cross-section paths (Phase 1 complete)
- **Schema zone unification** — schema level now uses required/optional/conditional zones like every other level (Phase 3 complete)
- **Nullable removal** — `nullable:` lists deleted, optional fields use standard zone pattern
- **Schema-level conditionals** — can now condition section presence (Phase 3 complete)
- **Section conditional containers** — sections can have conditional container zones (Phase 3 enhancement)

### Design Debt Addressed

**Before Redesign:**
- Per-field descriptions lost when types collapsed into shared atoms
- X-extensions had no home (too specific for shared types, too invariant for containers)
- Type narrowing implicit and undocumented

**After Field Membrane Redesign:**
- Field membrane recovers per-field descriptions
- X-extensions (semantic, constraint, validation) attach at field level
- Type narrowing explicit with double-underscore convention
- Draupnir disassembly separates `$ref` (type) from `description` (field meaning) in output

### What's Ready for Use

- All 348 YAML files
- All 13 schema files covering agent definition pipeline
- Field membrane with x-extensions
- Conditional presence system
- Type narrowing by pipeline stage

### What's Aspirational

- Dead types under review: `atom/timestamp_zulu`, `union/string_identifier` (zero consumers)
- Pattern portability: `pattern/home` hardcoded to `/Users/johnny/` (known limitation)

---

## 16. Critical Insights for Working with Verdandi

### 1. The Membrane Is Mandatory
Containers never reference raw types. They reference `field/` files. This is the architectural foundation that enables per-field semantic meaning while maintaining type reuse. Attempting to reference atoms/enums directly from containers will fail Draupnir compilation.

**Evidence:** COMPOSITION_RULES.md Level 9-15 all specify `field/` as the lowest import level.

### 2. Load Order Is Enforcement
The 15-level ordering is not documentation — it's enforced at compile time. Attempting to reference sideways or upward causes immediate failure. The downward-only rule is absolute.

**Evidence:** Draupnir's supply-driven level-ordered loading in the compiler validates references as each level loads.

### 3. Zones Determine Requirement Semantics
Zone membership (`required`, `optional`, `conditional`) is the single source of truth for whether a field/container must be present. There are no separate requirement lists, no `required: mandatory` flags at the container level (only at element level for compile-time verification).

**Evidence:** Schema zone unification plan documents that `nullable` was functionally identical to `optional` — both produce `Type | None = None` in Pydantic models.

### 4. Conditional Predicates Are Mutually Exclusive
Every conditional uses exactly one of `equals`, `in`, or `exists`. The validator rejects multiple predicates or zero predicates. This forces explicit, unambiguous condition semantics.

**Evidence:** ConditionalWhen model validator in PLAN_SCHEMA_ZONE_UNIFICATION.md specifies exactly_one_predicate validation.

### 5. Cross-Section Conditionals Require Leading Dot
Without a leading dot, a conditional references a sibling in the same object. With a leading dot, it starts from the schema root and traverses down. At schema level, every conditional must use a leading dot because siblings are sections (complex objects), not leaf fields.

**Evidence:** COMPOSITION_RULES.md section "Cross-Section Paths (Leading Dot)" and PLAN_SCHEMA_ZONE_UNIFICATION.md Phase 1.

### 6. Pipeline Narrowing Is Explicit
Type narrowing uses double-underscore naming: `field/context_path__any` (raw) vs `field/context_path__abs` (resolved). The suffix is part of the file identity. This allows loose types early (relative/absolute) and strict types later (absolute only).

**Evidence:** COMPOSITION_RULES.md "Double-Underscore Type Narrowing" section; field/ directory contains separate files for each variant.

### 7. The Schema Level Is Special
Schema is the only level that uses `sections:` OR `shallowsections:` (not both). This choice determines composition style:
- `sections:` — domain-organized, deep (references section/)
- `shallowsections:` — consumption-organized, wide (references shallowsection/)

**Evidence:** Draupnir's `exactly_one_section_type` validator enforces mutual exclusion.

### 8. Shallowsection Is Not a Lightweight Schema
Shallowsection is a proper container level with a 2-container limit, not a schema shortcut. The limit prevents degradation into improper deep composition. Use section/ + group/ for deeper structures.

**Evidence:** COMPOSITION_RULES.md specifies "Maximum 2 containers total per shallowsection."

### 9. Raw/Resolved/Reduced Describe Definition Processing, Not Verdandi Processing
These variants track how agent definitions flow through the resolver pipeline (raw → absolutized paths → inlined includes), NOT how Verdandi itself processes. Verdandi is static data; the variants describe constraint tightening at each definition stage.

**Evidence:** QUICKSTART.md section "Raw / Resolved / Reduced Stages" and PLAN_SCHEMA_ZONE_UNIFICATION.md discusses these as agent definition stages.

### 10. Documentation Drift Causes LLM Reversion
Stale documentation actively causes LLMs to revert features to match old docs. When guides describe the pre-redesign system, LLMs reading those guides will "fix" new code to match. Documentation updates are feature landing, not cleanup.

**Evidence:** PLAN_SCHEMA_ZONE_UNIFICATION.md "HIGH risk — documentation drift" section explicitly warns of this.

---

## Connections and Dependencies

### Draupnir (Consumer)
- Reads all 348 Verdandi YAML files in load order
- Validates with Pydantic models
- Generates 13 JSON Schema files
- Location: `/Users/johnny/ai/spaces/bragi/tools/draupnir/`

### Regin (Transform Engine)
- Consumes Draupnir's JSON Schema output
- Does NOT read Verdandi YAML directly
- Runs multi-step DAG pipeline: raw → paths-resolved → permissions-resolved → execution-merged → criteria-merged → includes-merged → universal-format → vendor-specific renders
- Generates Pydantic models from schemas

### Nornir (Validation Runtime)
- Embeds generated JSON Schemas at Rust compile time
- Validates TOML definitions at every pipeline boundary
- Checks enforcement, paths, permissions at runtime

### scaffold_definition.py
- Reads `schemas/agent-raw-definition.schema.json`
- Produces TOML authoring scaffolds
- Annotates every field with type, requirement, description

### inspect_schema.py
- Reads generated JSON Schema files
- Produces HTML inspection reports
- Shows field hierarchy, conditionals, polymorphic alternatives
- Must recognize presence-based conditionals (`if: { required: [...] }`)

### Agent Definitions (Consumer)
- TOML files in `definitions/agents/`
- Conform to agent-raw-definition schema
- Flow through pipeline to vendor-specific renders
- Evidence of system in use

---

## Recommendations

### For New Work on Verdandi

1. **Before modifying YAML**, read CLAUDE.md context refresh protocol
2. **Verify load order understanding** — name all 15 levels and tier assignments
3. **Use COMPOSITION_RULES.md as authoritative reference** for level-specific fields
4. **Always use Read tool first** before editing files to understand full context
5. **Run Draupnir after YAML changes** — schemas are derived artifacts
6. **Update documentation WITH code changes** — stale docs cause LLM reversion
7. **Test Draupnir compilation** — early feedback on constraint violations

### For Consuming Verdandi Output

1. **Generated schemas are read-only** — edit the YAML source, not the JSON output
2. **Verify schema generation** — compare old vs new output when making changes
3. **Check reference validity** — downward-only rule applies strictly
4. **Understand zone semantics** — requirement logic flows from zone membership

### For Future Enhancements

1. **Resolve dead types** — `atom/timestamp_zulu` and `union/string_identifier` have zero consumers
2. **Fix pattern portability** — `pattern/home` hardcoded to `/Users/johnny/`
3. **Extend conditional builders** — consider presence-based conditionals at container/section level
4. **Document schema variants** — comprehensive guide to which schema covers which pipeline stage

### For Context Preservation

1. **CLAUDE.md is the refresh protocol** — read before each work session
2. **COMPOSITION_RULES.md is the reference manual** — authoritative specs
3. **This research document is the comprehensive overview** — full mental model
4. **PLAN_SCHEMA_ZONE_UNIFICATION.md documents recent work** — context for active features

---

## Appendix: Complete Reference

### Key Files

| File | Purpose | Path |
|------|---------|------|
| CLAUDE.md | Context refresh protocol, common mistakes, recovery | `/Users/johnny/ai/smidja/verdandi/` |
| CONTEXT_MAP.md | Resource inventory, documentation references | `/Users/johnny/ai/smidja/verdandi/` |
| COMPOSITION_RULES.md | Formal specification, level-by-level reference | `/Users/johnny/ai/smidja/verdandi/` |
| QUICKSTART.md | Operational guide, examples, what not to do | `/Users/johnny/ai/smidja/verdandi/` |
| PLAN_SCHEMA_ZONE_UNIFICATION.md | Recent enhancements: zones, exists predicate, leading-dot | `/Users/johnny/ai/smidja/verdandi/` |
| VERDANDI_REDESIGN.md | Field membrane design rationale | `/Users/johnny/ai/smidja/verdandi/` |

### Generated Schemas

| Schema | Pipeline Role | Validates |
|--------|---------------|-----------|
| agent-raw-definition | Entry point | Authored TOML definitions |
| agent-paths-resolved | After path absolutization | All paths absolute |
| agent-permissions-resolved | After permission calculation | Security state resolved |
| agent-execution-merged | After execution merge | Execution section merged |
| agent-criteria-merged | After criteria merge | Criteria section merged |
| agent-instructions-reduced | After instruction inlining | Instructions inline |
| agent-examples-reduced | After example inlining | Examples inline |
| agent-guardrails-reduced | After guardrails inlining | Guardrails inline |
| agent-success-reduced | After success/failure inlining | Success/failure inline |
| agent-includes-merged | All includes merged | Convergence point |
| agent-universal-format | Final pipeline format | Platform-agnostic |
| agent-universal-render | Consumption-oriented | Rendering-focused |
| agent-anthropic-render | Vendor-specific | Anthropic format |

---

**End of Research Document**

This research captures the complete structural vocabulary of Verdandi at the time of analysis. The system is proven, enforced, and deployed. Future work should reference this document for the architectural foundations, COMPOSITION_RULES.md for specifics, and CLAUDE.md for context refresh protocols.
