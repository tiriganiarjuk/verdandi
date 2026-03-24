# Verdandi — Composition Rules

Formal specification for the Verdandi type composition system.
Replaces toolcompose with compile-time enforced hierarchy.

---

## Naming

**Verdandi** — "what is becoming." The types define what data
will become as it flows through the hierarchy.

**System name:** Verdandi
**Location:** `verdandi/{project}/source/`
**Consumer:** Draupnir (schema generator)
**Enforcer:** Draupnir (compile-time validation)

---

## Reference Model

Every reference is a file path: `level/filename` (without `.yaml`).
No name-based lookups. No inline type declarations above pattern level.

The file path IS the identity. At type levels (pattern through union),
the name is derived from the filename. No `name:` field needed.

**The field membrane is mandatory.** Containers (simplegroup, group,
shallowsection) reference `field/` files, never raw type-tier files
(atom/, enum/, etc.) directly. The field level is where per-field
descriptions, defaults, and x-extensions live.

---

## Composition Categories

### Types (structural constraint — what values are valid)

```
pattern          raw regex patterns — the lowest building block
atom             constrained base types (string/integer/boolean + pattern + exclusions)
enum             enumerated allowed values
element          atom + runtime format contract
union            oneOf composition of atoms, enums, or elements
tuple            fixed-length typed sequence (type composition, not a container)
```

### Simple Containers (fields only — no containers inside)

```
simplearray      variable-length array of one field type
simplegroup      flat object — fields organized into required/optional/conditional
```

### Complex Containers (fields + containers below them)

```
array            variable-length array — items are simplegroups
group            object containing fields + simple containers + arrays
superarray       variable-length array — items are groups (or simplegroups)
```

**Load order note:** arrays before groups. Arrays only reference
simplegroups. Groups reference simple containers AND arrays.

### Top Level

```
section          composes complex containers only (no direct fields)
shallowsection   composes fields + containers (simplearray/array/superarray/group)
schema           root — `sections:` key (deep) or `shallowsections:` key (wide)
```

---

## Composition Rule

Each level references only levels below it in the load order:

```
simplearray        accepts: field/
simplegroup        accepts: field/, simplearray/
array              accepts: simplegroup/
group fields       accepts: field/
group containers   accepts: simplearray/, simplegroup/, array/
superarray         accepts: group/, simplegroup/
section            accepts: group/, array/, superarray/
shallowsection     accepts: field/, simplearray/, array/, superarray/, group/
```

Draupnir enforces tier boundaries at compile time. Sideways and
upward references are compilation errors.

---

## Level Specifications

### pattern/

The lowest building block. Raw regex patterns reusable by atoms.

```yaml
# pattern/relative.yaml
title: Relative
regex: "^\."
description: Starts with dot (relative path prefix)
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| regex | yes | Regular expression pattern |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Referenced by: atoms (via `pattern:` and `excludes:`)

---

### atom/

Smallest useful constrained type. Name derived from filename.

```yaml
# atom/path_relative.yaml
title: PathRelative
type: string
pattern: pattern/relative
excludes: [pattern/traversal, pattern/env, pattern/home, pattern/root]
description: Relative path starting with ./ (excludes traversal and sensitive paths)
```

```yaml
# atom/integer.yaml
title: Integer
type: integer
description: Whole number value
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| type | yes | `string`, `integer`, or `boolean` |
| pattern | no | Reference to `pattern/` file (strings only) |
| excludes | no | List of `pattern/` references to reject (strings only) |
| min_length | no | Minimum string length (emitted as JSON Schema `minLength`) |
| description | yes | Human-readable explanation (used in scaffolding templates) |

`pattern:` is a positive constraint (must match). `excludes:` is a
negative constraint (must NOT match). They work together — the atom
must match the pattern AND must not match any excluded patterns.

Referenced by: elements, unions, fields (via type tier)

---

### enum/

Enumerated allowed values. Name derived from filename.

```yaml
# enum/dispatch_mode.yaml
title: DispatchMode
values: [batch, full]
description: How the dispatcher feeds input — all at once or in chunks
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| values | yes | List of allowed values |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Referenced by: elements, unions, fields (via type tier)

---

### element/

Adds runtime format contract to an atom or enum. Name derived from filename.

```yaml
# element/path_exists_relative.yaml
title: PathExistsRelative
inherits: atom/path_relative
format: exists
description: Relative path that must exist on the filesystem
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| inherits | yes | Reference to `atom/` or `enum/` |
| format | no | Runtime validation keyword (e.g., `exists`, `exists_schema`) |
| required | no | Only value: `mandatory`. Compile-time lock — Draupnir verifies this element appears in every schema |
| description | yes | Human-readable explanation (used in scaffolding templates) |

`format` bridges schema-time and runtime validation. Schema enforces
pattern + exclusions (from inherited atom). Nornir enforces the
format contract (e.g., filesystem existence).

Referenced by: unions, fields (via type tier)

---

### union/

Combines type-tier models via oneOf. Name derived from filename.

```yaml
# union/path_any.yaml
title: PathAny
union: [atom/path_relative, atom/path_absolute]
description: Filesystem path — either relative (./) or absolute (/)
```

```yaml
# union/path_exists_any.yaml
title: PathExistsAny
union: [element/path_exists_relative, element/path_exists_absolute]
description: Path that must exist — either relative or absolute
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| union | yes | List of `atom/`, `enum/`, or `element/` references |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Pure composition — no constraints added. Members must all be
type-tier models (atom, enum, or element).

Referenced by: field/ (via `type:`), tuple/ (via `items:`)

---

### tuple/

Fixed-length typed sequence. Name derived from filename.

```yaml
# tuple/dispatch_batch_size.yaml
title: DispatchBatchSize
items: [atom/integer, atom/integer]
constraint: ascending_strict
description: >-
  [min, max] records per batch chunk. Min must be strictly less than max.
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| items | yes | Ordered list of type-tier references (atom/enum/element/union) |
| constraint | no | Validation rule (e.g., `ascending_strict`) |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Length is implicit from `items` count. Positions can reference
different type-tier models.

Referenced by: field/ (via `type:`)

---

### field/

The membrane between types and containers. Each field file binds a
type reference to a per-field description and optional x-extensions.
This is where semantic meaning attaches to structural types.

**Every container references `field/` files, never raw type-tier files
directly.** This is the fundamental architectural rule. If you need a
string field in a group, create a `field/` file that references the
atom, then reference that field file from the group.

```yaml
# field/agent_description.yaml
title: AgentDescription
type: atom/string_prose
min_length: 40
description: >-
  One-sentence purpose statement. Appears in scaffolding, inspection
  reports, and agent identity sections.
semantic:
  intent: >-
    Must describe WHAT the agent does, not HOW. Avoid implementation
    details. Focus on the user-facing capability.
  checks:
    - Describes a concrete end state, not an ongoing activity
  anti_patterns:
    - Implementation details instead of user-facing capability
  severity: error
```

```yaml
# field/dispatch_max_agents.yaml
title: DispatchMaxAgents
type: atom/integer
default: 6
description: >-
  Maximum number of concurrent agent instances the dispatcher may spawn.
```

```yaml
# field/scratch_needed.yaml
title: ScratchNeeded
type: atom/boolean
default: false
description: >-
  Whether this agent needs a scratch directory for intermediate files.
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| type | yes | Reference to type tier: `atom/`, `enum/`, `element/`, `union/`, or `tuple/` |
| description | yes | Per-field semantic meaning (what this field represents in context) |
| default | no | Default value (`string`, `integer`, or `boolean`). Emitted as JSON Schema `default` |
| const | no | Constant value (`string`). Emitted as JSON Schema `const` |
| min_length | no | Minimum string length (emitted as JSON Schema `minLength`). Overrides atom-level min_length when higher |
| semantic | no | Quality/intent checks (emitted as `x-semantic` in JSON Schema). See structure below |
| constraint | no | Cross-field/domain rules (emitted as `x-constraint` in JSON Schema). See structure below |

Referenced by: simplearray (via `items:`), simplegroup/group/shallowsection
(via required/optional/conditional zones)

#### Semantic Extension Structure

The `semantic:` key is an object with structured fields, not a plain string:

```yaml
semantic:
  intent: >-
    The agent's goal state. LLMs are goal-driven — explicit success
    definitions pull behavior more effectively than constraints push it.
  checks:
    - Describes a concrete end state, not an ongoing activity
    - Refers to the PROCESS succeeding, not data quality outcomes
    - Observable — you can tell from outputs whether this was achieved
  anti_patterns:
    - Confusing process success with data quality
  severity: error
```

| Subfield | Required | Description |
|----------|----------|-------------|
| intent | no | Primary purpose / semantic intent |
| checks | no | List of validation criteria (strings) |
| anti_patterns | no | List of known anti-patterns (strings) |
| severity | no | Severity level (e.g., `error`, `warning`) |

#### Constraint Extension Structure

The `constraint:` key is a list of rule objects:

```yaml
constraint:
  - rule: required_when
    field: .task.processing.output_file_format
    in: [jsonl, json]
    severity: error
    reason: >-
      Structured output (JSON or JSONL) requires a schema for validation.
```

| Subfield | Required | Description |
|----------|----------|-------------|
| rule | yes | Constraint type (e.g., `required_when`) |
| severity | yes | Severity level |
| reason | yes | Explanation of why this constraint exists |
| field | no | Target field path (supports leading-dot cross-section paths) |
| order | no | Ordering metadata |
| in | no | List of values for enum-style checks |

#### Double-Underscore Type Narrowing

When the same logical field needs different type constraints at
different pipeline stages, the field name carries a double-underscore
suffix indicating the type variant:

| Suffix | Meaning | Type Reference | Stage |
|--------|---------|----------------|-------|
| `__any` | Accepts relative or absolute paths | `union/path_any` | Raw (pre-resolution) |
| `__abs` | Absolute paths only | `atom/path_absolute` | Resolved (post-resolution) |
| `__x_any` | Path must exist, relative or absolute | `union/path_exists_any` | Raw (pre-resolution) |
| `__x_abs` | Path must exist, absolute only | `element/path_exists_absolute` | Resolved (post-resolution) |

The `x` prefix means "exists" — the field references an `element/`
type with `format: exists`, adding a runtime filesystem existence
contract on top of the path pattern.

Examples:
```
field/context_path__any.yaml     type: union/path_any          (raw stage)
field/context_path__abs.yaml     type: atom/path_absolute      (resolved stage)
field/include_instruction__x_any type: union/path_exists_any   (raw, must exist)
field/include_instruction__x_abs type: element/path_exists_absolute (resolved, must exist)
```

The base name (`context_path`, `include_instruction`) identifies the
logical field. The suffix identifies the type narrowing applied at
that pipeline stage. This lets different schemas reference the same
logical field with stage-appropriate type constraints.

---

### simplearray/

Variable-length array of one field type.

```yaml
# simplearray/role_expertise.yaml
title: RoleExpertise
items: field/role_expertise_item
description: Agent areas of expertise
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| items | yes | Single `field/` reference (the field membrane) |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Simplearrays do NOT support `oneOf`. For polymorphic arrays, use
`array/` or `superarray/`.

---

### simplegroup/

Flat object with fields organized by requirement.

```yaml
# simplegroup/parameter_item.yaml
title: ParameterItem
description: Named parameter the agent accepts at dispatch time
required:
  param_name: field/param_name
  param_type: field/param_type
  param_required: field/param_required
optional:
  param_description: field/param_description
```

```yaml
# simplegroup/success_item.yaml
title: SuccessItem
description: A single success criterion with definition and observable evidence
required:
  success_definition: field/success_definition
  success_evidence: simplearray/success_evidence
```

| Section | Required | Description |
|---------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| description | yes | Human-readable explanation (used in scaffolding templates) |
| required | no | Named refs to `field/` or `simplearray/` |
| optional | no | Named refs to `field/` or `simplearray/` |
| conditional | no | Named conditional entries (see Conditionals below) |

At least one of `required`, `optional`, or `conditional` must exist.

Zone values reference: `field/`, `simplearray/`. Tuples are accessed
through `field/` refs (a field file with `type: tuple/...`), never
referenced directly in zones.

---

## Conditionals

Conditionals control field presence based on sibling field values or
cross-section field values. They appear in `conditional:` zones at
the simplegroup, group, shallowsection, and schema levels.

### Structure

Every conditional entry has one `ref:` and exactly one directional
keyword (`only_when`, `always_when`, or `not_when`):

```yaml
conditional:
  field_name:
    ref: field/some_field
    only_when:
      field: sibling_field
      equals: some_value
```

### Directional Keywords

```
only_when      if and only if — required when true, FORBIDDEN when false
always_when    required when condition is true (no effect when false)
not_when       forbidden when condition is true (no effect when false)
```

### Predicates

Each condition uses exactly one predicate — they are mutually exclusive:

```
equals: value           const check — matches a single value
in: [value1, value2]    enum check — matches any of several values
exists: true            presence check — matches when the field is present
```

`exists: false` is **not supported**. Use `not_when` with `exists: true`
for the inverse (forbidden when present).

### Value-Based Examples

```yaml
# const check — dispatch_batch_size required only when mode is batch
conditional:
  dispatch_batch_size:
    ref: field/dispatch_batch_size
    only_when:
      field: dispatch_mode
      equals: batch
```

```yaml
# enum check — return_status required for either status mode
conditional:
  return_status:
    ref: field/return_status
    only_when:
      field: return_mode
      in: [status, status-metrics]
```

### Presence-Based Example

```yaml
# presence check — resource_timeout required when resource_url exists
conditional:
  resource_timeout:
    ref: field/resource_timeout
    only_when:
      field: resource_url
      exists: true
```

The directional keywords combine with `exists: true` the same way they
combine with value predicates:

```
only_when   + exists: true   required when present, forbidden when absent
always_when + exists: true   required when present, no constraint when absent
not_when    + exists: true   forbidden when present, no constraint when absent
```

### Cross-Section Paths (Leading Dot)

The `field:` value in a conditional trigger uses two conventions:

- **No dot** — same-object sibling reference. `field: dispatch_mode` means
  "the `dispatch_mode` field in this same group/simplegroup."
- **Leading dot** — root traversal. `field: .indirect.dispatch.dispatch_mode`
  means "start from the schema root, traverse into the `indirect` section,
  then `dispatch`, then read `dispatch_mode`."

Same-object (sibling):
```yaml
# group/dispatch.yaml — dispatch_mode is a sibling in this group
conditional:
  dispatch_batch_size:
    ref: field/dispatch_batch_size
    only_when:
      field: dispatch_mode
      equals: batch
```

Cross-section (root traversal):
```yaml
# group/security_io.yaml — references a field in a different section
conditional:
  io_input_file:
    ref: field/io_input_file__abs
    only_when:
      field: .indirect.dispatch.dispatch_input_delivery
      equals: file
```

### Where Conditionals Appear

| Level | Conditional zones on | Ref type in conditional |
|-------|---------------------|------------------------|
| simplegroup | top-level zones | `field/`, `simplearray/` |
| group | `fields:` section | `field/` |
| group | `containers:` section | `simplearray/`, `simplegroup/`, `array/` |
| shallowsection | top-level zones | `field/` |
| schema | `sections:` / `shallowsections:` zones | `section/` or `shallowsection/` |

---

### group/

Object containing fields and containers. Two separate sections
keep fields and containers distinct.

```yaml
# group/metadata.yaml — fields only, no containers
title: Metadata
description: Agent identity and version
fields:
  required:
    agent_name: field/agent_name
    agent_description: field/agent_description
    semantic_version: field/semantic_version
```

```yaml
# group/enforcement_explicit_grants.yaml — containers only, no fields
title: EnforcementExplicitGrants
description: Author-defined capability grants — paths per capability bucket
containers:
  required:
    read: simplearray/enforcement_explicit_read
    search: simplearray/enforcement_explicit_search
    write: simplearray/enforcement_explicit_write
    edit: simplearray/enforcement_explicit_edit
    append: simplearray/enforcement_explicit_append
    inspect: simplearray/enforcement_explicit_inspect
```

```yaml
# group/dispatch.yaml — fields + containers + conditional
title: Dispatch
description: Dispatcher configuration — batching, concurrency, execution mode
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

| Section | Required | Description |
|---------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| description | yes | Human-readable explanation (used in scaffolding templates) |
| fields | no | `required`/`optional`/`conditional` zones — values reference `field/` |
| containers | no | `required`/`optional`/`conditional` zones — values reference `simplearray/`, `simplegroup/`, `array/` |

At least one of `fields` or `containers` must exist. Both may be present.

---

### array/

Variable-length array of simplegroups.

```yaml
# array/dispatch_parameters.yaml
title: DispatchParameters
items: simplegroup/parameter_item
description: Named parameters the agent accepts at dispatch time
```

```yaml
# array/execution_instructions.yaml
title: ExecutionInstructions
items:
  oneOf: [simplegroup/instruction_step, simplegroup/include_instruction]
description: Ordered processing steps — inline or included from file
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| items | yes | Reference to `simplegroup/` — or `oneOf` list for polymorphic arrays |
| description | yes | Human-readable explanation (used in scaffolding templates) |

---

### superarray/

Variable-length array of groups (or simplegroups).

```yaml
# superarray/execution_examples.yaml
title: ExecutionExamples
items:
  oneOf: [group/example_group, simplegroup/include_example_group]
description: Example groups — inline or included from file
```

```yaml
# superarray/display_entries.yaml
title: DisplayEntries
items: group/display_entry
description: Security boundary display entries
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| items | yes | Reference to `group/` or `simplegroup/` — or `oneOf` list for polymorphic arrays |
| description | yes | Human-readable explanation (used in scaffolding templates) |

---

### section/

Composes complex containers only. No direct fields.

```yaml
# section/execution.yaml
title: Execution
description: How the agent processes work — include paths absolutized
containers:
  required:
    execution_instructions: array/execution_instructions
    execution_examples: superarray/execution_examples
```

```yaml
# section/security.yaml
title: Security
description: Access control after path resolution — all paths absolutized
containers:
  required:
    security_io: group/security_io
    security_schema: group/security_schema
    security_paths: group/security_paths
    context: group/context
  optional:
    security_capabilities: group/security_capabilities
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| description | yes | Human-readable explanation (used in scaffolding templates) |
| containers | yes | `required`/`optional`/`conditional` zones — values reference `group/`, `array/`, `superarray/` |

---

### shallowsection/

Composes fields alongside containers. Field refs live at the top
level (not nested under a `fields:` key like group/).

```yaml
# shallowsection/identity.yaml
title: Identity
description: Agent identity — who it is, what tier it runs at
required:
  title: field/agent_title
  description: field/agent_description
  role_identity: field/role_identity
  role_responsibility: field/role_responsibility
  capability_tier: field/agent_capability_tier
optional:
  role_description: field/role_description
containers:
  optional:
    role_expertise: simplearray/role_expertise
```

```yaml
# shallowsection/dispatcher.yaml — with conditional field
title: Dispatcher
description: Everything the dispatcher skill needs to launch and feed the agent
required:
  agent_name: field/agent_name
  dispatch_mode: field/dispatch_mode
  # ... more required fields
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

| Section | Required | Description |
|---------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| description | yes | Human-readable explanation (used in scaffolding templates) |
| required | no | Named `field/` references |
| optional | no | Named `field/` references |
| conditional | no | Named conditional entries referencing `field/` |
| containers | no | `required`/`optional` zones — values reference `simplegroup/`, `simplearray/`, `array/`, `superarray/`, `group/` |

At least one of `required`, `optional`, `conditional`, or `containers`
must exist.

**Container limit:** Maximum 3 containers per shallowsection. This
prevents abuse — shallowsections are for mostly-flat structures with
occasional nesting, not an excuse to avoid proper deep composition.
If you need more than 3 containers, use section/ with groups.

---

### schema/

Root file. Uses exactly one of two composition keys:

- **`sections:`** — references `section/` types (deep composition)
- **`shallowsections:`** — references `shallowsection/` types (wide composition)

A schema must choose one key. Never both. Draupnir enforces mutual
exclusion at compile time.

Both `sections:` and `shallowsections:` use the same zone pattern as
every other level: `required:`, `optional:`, and `conditional:` sub-maps.
Membership in a zone determines requirement semantics — there are no
separate top-level `required:` or `nullable:` lists.

`nullable` was removed. It was functionally identical to `optional` —
both produce `SomeSection | None = None` in generated Pydantic models,
and TOML (the authoring format) cannot express null.

**Deep schema** (pipeline stages — domain-organized):

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
```

**Wide schema** (render stages — consumption-organized):

```yaml
# schema/agent-universal-render.yaml
title: Agent Universal Render
version: 1
description: Consumption-oriented regrouping of universal-format for rendering
shallowsections:
  required:
    identity: shallowsection/identity
    input: shallowsection/input
    instructions: shallowsection/instructions_reduced
    output: shallowsection/output
    return_format: shallowsection/return_format
    dispatcher: shallowsection/dispatcher
  optional:
    security_boundary: shallowsection/security_boundary
    implicit_grants: shallowsection/implicit_grants
    writing_output: shallowsection/writing_output
    constraints: shallowsection/constraints
    anti_patterns: shallowsection/anti_patterns
    examples: shallowsection/examples_reduced
```

**Schema with conditional sections:**

```yaml
# Conditional sections use the same syntax as other levels
shallowsections:
  required:
    identity: shallowsection/identity
  optional:
    examples: shallowsection/examples_reduced
  conditional:
    writing_output:
      ref: shallowsection/writing_output
      only_when:
        field: .enforcement.enforcement_output_tool.tool_format
        exists: true
```

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | PascalCase name (used for Python class generation) |
| version | yes | Schema version |
| description | yes | Human-readable explanation (used in scaffolding templates) |
| sections | mutual | `required`/`optional`/`conditional` zones — values reference `section/` types (deep composition) |
| shallowsections | mutual | `required`/`optional`/`conditional` zones — values reference `shallowsection/` types (wide composition) |

**`sections` and `shallowsections` are mutually exclusive.** Exactly one
must be present. The key name signals the composition style and constrains
what types may be referenced.

**Schema-level conditionals are always cross-section.** At the schema root,
the "siblings" are sections — complex objects, not leaf fields with matchable
values. Every conditional at the schema level MUST use a dotted path (with
leading dot) to reach into a section to find a field.

---

## Path Type Chain (complete example)

```
pattern/relative                regex: "^\."
pattern/absolute                regex: "^/"
pattern/traversal               regex: "\.\."

atom/path_relative              type: string, pattern: pattern/relative, excludes: [pattern/traversal, ...]
atom/path_absolute              type: string, pattern: pattern/absolute, excludes: [pattern/traversal, ...]

element/path_exists_relative    inherits: atom/path_relative, format: exists
element/path_exists_absolute    inherits: atom/path_absolute, format: exists

union/path_any                  union: [atom/path_relative, atom/path_absolute]
union/path_exists_any           union: [element/path_exists_relative, element/path_exists_absolute]

field/context_path__any         type: union/path_any          (raw stage)
field/context_path__abs         type: atom/path_absolute      (resolved stage)
```

8 types. 4 levels. Each level adds exactly one thing. The field
membrane binds type to meaning. Containers reference the field,
never the raw type.

Pipeline narrowing: definitions accept `field/context_path__any`
(relative or absolute). Resolver absolutizes all paths. Resolved
stage uses `field/context_path__abs`. The looser type drops away.

---

## Compile-Time Enforcement

Draupnir validates at compile time:

1. **Tier boundaries** — every reference points to a level below
2. **No sideways imports** — no same-level references
3. **Mandatory elements** — elements with `required: mandatory` must
   appear somewhere in the schema being compiled
4. **Conditional validity** — `only_when`/`always_when`/`not_when`
   reference fields that exist in the same container (no dot) or
   use leading-dot paths for cross-section root traversal
5. **Type consistency** — reference targets exist and are the correct
   level for their position
6. **Schema mutual exclusion** — exactly one of `sections:` or
   `shallowsections:` must be present
7. **Shallowsection container limit** — max 2 containers total
8. **Zone invariants** — at least one zone must be populated where required
