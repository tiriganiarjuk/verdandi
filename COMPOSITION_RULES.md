# Verdandi — Composition Rules

Formal specification for the Verdandi type composition system.
Replaces toolcompose with compile-time enforced hierarchy.

---

## Naming

**Verdandi** — "what is becoming." The types define what data
will become as it flows through the hierarchy.

**System name:** Verdandi
**Location:** `tools/verdandi/`
**Consumer:** Draupnir (schema generator)
**Enforcer:** Draupnir (compile-time validation)

---

## Reference Model

Every reference is a file path: `level/filename` (without `.yaml`).
No name-based lookups. No inline type declarations above pattern level.

The file path IS the identity. At field levels (pattern through union),
the name is derived from the filename. No `name:` field needed.

---

## Composition Categories

### Fields (single items)

```
pattern          raw regex patterns — the lowest building block
atom             constrained base types (string/integer/boolean + pattern + exclusions)
enum             enumerated allowed values
element          atom + runtime format contract
union            oneOf composition of atoms, enums, or elements
```

### Simple Containers (fields only — no containers inside)

```
tuple            fixed-length typed sequence of fields
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
shallowsection   composes fields + containers (simplearray/array/group)
schema           root — references sections and shallowsections
```

---

## Composition Rule

Each level references only levels below it in the load order:

```
simple containers  accept: fields (simplegroup also accepts tuple/simplearray)
array              accepts: simplegroups
group              accepts: fields + simple containers + arrays
superarray         accepts: groups + simplegroups
section            accepts: complex containers (group/array/superarray)
shallowsection     accepts: fields + containers (simplearray/array/group)
```

Draupnir enforces tier boundaries at compile time. Sideways and
upward references are compilation errors.

---

## Level Specifications

### pattern/

The lowest building block. Raw regex patterns reusable by atoms.

```yaml
# pattern/relative.yaml
regex: "^\."
description: Starts with dot (relative path prefix)
```

| Field | Required | Description |
|-------|----------|-------------|
| regex | yes | Regular expression pattern |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Referenced by: atoms (via `pattern:` and `excludes:`)

---

### atom/

Smallest useful constrained type. Name derived from filename.

```yaml
# atom/path_relative.yaml
type: string
pattern: pattern/relative
excludes: [pattern/traversal, pattern/env, pattern/home, pattern/root]
description: Relative path starting with ./ (excludes traversal and sensitive paths)
```

```yaml
# atom/integer.yaml
type: integer
description: Whole number value
```

| Field | Required | Description |
|-------|----------|-------------|
| type | yes | `string`, `integer`, or `boolean` |
| pattern | no | Reference to `pattern/` file (strings only) |
| excludes | no | List of `pattern/` references to reject (strings only) |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Referenced by: enums, elements, unions, simple containers

---

### enum/

Enumerated allowed values. Name derived from filename.

```yaml
# enum/dispatch_mode.yaml
values: [batch, full]
description: How the dispatcher feeds input — all at once or in chunks
```

| Field | Required | Description |
|-------|----------|-------------|
| values | yes | List of allowed values |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Referenced by: elements, unions, simple containers

---

### element/

Adds runtime format contract to an atom or enum. Name derived from filename.

```yaml
# element/path_exists_relative.yaml
inherits: atom/path_relative
format: exists
description: Relative path that must exist on the filesystem
```

| Field | Required | Description |
|-------|----------|-------------|
| inherits | yes | Reference to `atom/` or `enum/` |
| format | yes | Runtime validation keyword (e.g., `exists`) |
| description | yes | Human-readable explanation (used in scaffolding templates) |

`format` bridges schema-time and runtime validation. Schema enforces
pattern + exclusions (from inherited atom). Nornir enforces the
format contract (e.g., filesystem existence).

Optional: `required: mandatory` — compile-time lock. Draupnir verifies
this element appears somewhere in every schema it compiles.

Referenced by: unions, simple containers

---

### union/

Combines fields via oneOf. Name derived from filename.

```yaml
# union/path_any.yaml
union: [atom/path_relative, atom/path_absolute]
```

```yaml
# union/path_exists_any.yaml
union: [element/path_exists_relative, element/path_exists_absolute]
```

```yaml
# union/output_format.yaml
union: [enum/output_format_text, enum/output_format_structured]
```

| Field | Required | Description |
|-------|----------|-------------|
| union | yes | List of `atom/`, `enum/`, or `element/` references |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Pure composition — no constraints added. Members must all be
field-level types (atom, enum, or element).

Referenced by: simple containers

---

### tuple/

Fixed-length typed sequence. Name derived from filename.

```yaml
# tuple/dispatch_batch_size.yaml
items: [atom/integer, atom/integer]
constraint: ascending_strict
```

| Field | Required | Description |
|-------|----------|-------------|
| items | yes | Ordered list of field references (atom/enum/element/union) |
| constraint | no | Validation rule (e.g., `ascending_strict`) |
| description | yes | Human-readable explanation (used in scaffolding templates) |

Length is implicit from `items` count. Positions can reference
different field types.

---

### simplearray/

Variable-length array of one field type.

```yaml
# simplearray/role_expertise.yaml
items: atom/string_plain
```

| Field | Required | Description |
|-------|----------|-------------|
| items | yes | Single field reference (atom/enum/element/union) |
| description | yes | Human-readable explanation (used in scaffolding templates) |

---

### simplegroup/

Flat object with fields organized by requirement.

```yaml
# simplegroup/parameter_item.yaml
required:
  param_name: atom/kebab_string
  param_type: enum/param_type
  param_required: atom/boolean
optional:
  param_description: atom/string_plain
```

```yaml
# simplegroup/example_with_conditional.yaml
required:
  mode: enum/dispatch_mode
optional:
  max_agents: atom/integer
conditional:
  batch_size:
    ref: tuple/dispatch_batch_size
    only_when:
      field: mode
      equals: batch
```

| Section | Required | Description |
|---------|----------|-------------|
| description | yes | Human-readable explanation (used in scaffolding templates) |
| required | no | Fields that must be present |
| optional | no | Fields that may be absent |
| conditional | no | Fields with conditional presence |

At least one of `required`, `optional`, or `conditional` must exist.

Field values reference: atom/, enum/, element/, union/, tuple/,
simplearray/ (any field or simple container type).

#### Conditional keywords

```
only_when      if and only if — required when true, forbidden when false
always_when    required when condition is true (no effect when false)
not_when       forbidden when condition is true (no effect when false)
```

Each conditional field declares its trigger:

```yaml
conditional:
  field_name:
    ref: atom/string_plain
    only_when:
      field: sibling_field
      equals: some_value
```

---

### group/

Object containing fields and containers. Two separate sections
keep fields and containers distinct.

```yaml
# group/enforcement_explicit_grants.yaml
description: Author-defined capability grants — paths per capability bucket
containers:
  required:
    read: simplearray/enforcement_read
    search: simplearray/enforcement_search
    write: simplearray/enforcement_write
    edit: simplearray/enforcement_edit
    append: simplearray/enforcement_append
    inspect: simplearray/enforcement_inspect
```

```yaml
# group/dispatch.yaml
description: Dispatcher configuration — batching, concurrency, execution mode
fields:
  required:
    dispatch_mode: enum/dispatch_mode
    dispatch_input_format: enum/input_format
  conditional:
    dispatch_batch_size:
      ref: tuple/dispatch_batch_size
      only_when:
        field: dispatch_mode
        equals: batch
containers:
  optional:
    dispatch_parameters: array/dispatch_parameters
```

| Section | Required | Description |
|---------|----------|-------------|
| description | yes | Human-readable explanation (used in scaffolding templates) |
| fields | no | `required`/`optional`/`conditional` — references field types (+ tuple in conditionals) |
| containers | no | `required`/`optional`/`conditional` — references simple containers + arrays |

At least one of `fields` or `containers` must exist.

Field values reference: atom/, enum/, element/, union/ (conditionals also accept tuple/).
Container values reference: tuple/, simplearray/, simplegroup/, array/.

---

### array/

Variable-length array of simplegroups.

```yaml
# array/dispatch_parameters.yaml
items: simplegroup/parameter_item
description: Named parameters the agent accepts at dispatch time
```

```yaml
# array/execution_instructions.yaml
items:
  oneOf: [simplegroup/instruction_step, simplegroup/include_instruction]
description: Ordered processing steps — inline or included from file
```

| Field | Required | Description |
|-------|----------|-------------|
| items | yes | Reference to simplegroup/ — or oneOf list for polymorphic arrays |
| description | yes | Human-readable explanation (used in scaffolding templates) |

---

### superarray/

Variable-length array of groups.

```yaml
# superarray/execution_examples.yaml
items: group/example_group
description: Example groups with entries demonstrating expected behavior
```

| Field | Required | Description |
|-------|----------|-------------|
| items | yes | Reference to group/ — or oneOf list for polymorphic arrays |
| description | yes | Human-readable explanation (used in scaffolding templates) |

---

### section/

Composes complex containers only. No direct fields.

```yaml
# section/execution.yaml
components:
  instructions: array/execution_instructions
  examples: superarray/execution_examples
description: How the agent processes work — steps and examples
```

| Field | Required | Description |
|-------|----------|-------------|
| components | yes | Named references to group/, array/, superarray/ |
| description | yes | Human-readable explanation (used in scaffolding templates) |

---

### shallowsection/

Composes fields alongside containers. Same structure as group but
with field refs at top level (not nested under `fields:` key).

```yaml
# shallowsection/identity.yaml
description: Agent identity — who it is, what tier it runs at
required:
  title: atom/agent_title
  description: element/agent_description
  role_identity: atom/string_plain
  role_responsibility: atom/string_plain
  capability_tier: enum/processing_capability
optional:
  role_description: atom/string_plain
containers:
  optional:
    role_expertise: simplearray/role_expertise
```

| Section | Required | Description |
|---------|----------|-------------|
| description | yes | Human-readable explanation (used in scaffolding templates) |
| required | no | Field references (atom/enum/element/union) |
| optional | no | Field references |
| conditional | no | Conditional field references |
| containers | no | `required`/`optional` — references simplearray/, array/, group/ |

At least one of `required`, `optional`, `conditional`, or `containers`
must exist.

**Container limit:** Maximum 2 containers per shallowsection. This
prevents abuse — shallowsections are for mostly-flat structures with
occasional nesting, not an excuse to avoid proper deep composition.
If you need more than 2 containers, use section/ with groups.

---

### schema/

Root file. References sections OR shallowsections — not both.
A schema must choose deep (sections) or wide (shallowsections).

```yaml
# schema/agent-universal-render.yaml
version: 1
description: Consumption-oriented regrouping of universal-format for rendering
required:
  - identity
  - input
  - instructions
  - output
  - return_format
  - dispatcher
nullable:
  - security_boundary
  - implicit_grants
  - examples
  - writing_output
  - constraints
  - anti_patterns
sections:
  identity: shallowsection/identity
  security_boundary: shallowsection/security_boundary
  implicit_grants: shallowsection/implicit_grants
  input: shallowsection/input
  instructions: section/instructions
  examples: section/examples
  output: shallowsection/output
  writing_output: section/writing_output
  constraints: shallowsection/constraints
  anti_patterns: shallowsection/anti_patterns
  return_format: shallowsection/return_format
  dispatcher: shallowsection/dispatcher
```

| Field | Required | Description |
|-------|----------|-------------|
| version | yes | Schema version |
| description | yes | Human-readable explanation (used in scaffolding templates) |
| required | yes | List of section names that must be present |
| nullable | no | List of section names that may be null/absent |
| sections | yes | Named references to section/ OR shallowsection/ (no mixing) |

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
```

6 types. 3 levels. Each level adds exactly one thing. Unions compose
atoms OR elements — always referencing below, no sideways.

Pipeline narrowing: definitions accept `union/path_any` (relative or
absolute). Resolver absolutizes all paths. Resolved stage uses
`atom/path_absolute` directly. The looser type drops away.

---

## Compile-Time Enforcement

Draupnir validates at compile time:

1. **Tier boundaries** — every reference points to a level below
2. **No sideways imports** — no same-level references
3. **Mandatory elements** — elements with `required: mandatory` must
   appear somewhere in the schema being compiled
4. **Conditional validity** — `only_when`/`always_when`/`not_when`
   reference fields that exist in the same container
5. **Type consistency** — reference targets exist and are the correct
   level for their position
