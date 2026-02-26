# Verdandi -- Quickstart

## What This Is

Verdandi is a **typed composition hierarchy** expressed as YAML files. It defines the structural vocabulary for agent tool definitions -- what fields exist, what types they have, how they compose into groups, sections, and complete schemas.

Verdandi is **not a script**. It is a data directory. You do not execute it. Other tools consume it.

## How It Works

Verdandi contains ~179 YAML files organized into 14 subdirectories (one per composition level). Each file defines a single reusable type. Higher-level types reference lower-level types by file path (e.g., `atom/kebab_string` means `atom/kebab_string.yaml`).

**Draupnir** (`tools/draupnir/`) is the compiler. It reads all Verdandi YAML files in dependency order, validates them into frozen Pydantic models, and generates JSON Schema files to `schemas/`.

The generated JSON Schemas are then consumed by:
- **Nornir** for runtime validation of agent TOML definitions
- **scaffold_definition.py** to generate TOML authoring scaffolds
- **inspect_schema.py** to generate HTML inspection reports
- **transform_to_universal** to generate Pydantic model modules for the resolution pipeline

## The Type Hierarchy

14 levels in 4 tiers. Each level can ONLY reference levels below it.

### Fields (leaf nodes)

| Level | What It Holds | Example |
|-------|---------------|---------|
| **pattern** | Raw regex | `regex: "^[a-z0-9]+(-[a-z0-9]+)*$"` |
| **atom** | Base type + pattern + exclusions | `type: string, pattern: pattern/kebab` |
| **enum** | Allowed values | `values: [batch, full]` |
| **element** | Atom/enum + runtime contract | `inherits: atom/path_absolute, format: exists` |
| **union** | OneOf composition of fields | `union: [atom/path_relative, atom/path_absolute]` |

### Simple Containers (fields only, no nesting)

| Level | What It Holds | Example |
|-------|---------------|---------|
| **tuple** | Fixed-length sequence of fields | `items: [atom/integer, atom/integer]` |
| **simplearray** | Variable-length array of one field type | `items: atom/string_plain` |
| **simplegroup** | Flat object with required/optional/conditional zones | `required: {name: atom/kebab_string}` |

### Complex Containers (can nest other containers)

| Level | What It Holds | Example |
|-------|---------------|---------|
| **array** | Variable-length array of simplegroups | `items: simplegroup/parameter_item` |
| **group** | Object with fields section + containers section | `fields: {required: ...}, containers: {optional: ...}` |
| **superarray** | Variable-length array of groups | `items: group/example_group_raw` |

### Top Level

| Level | What It Holds | Example |
|-------|---------------|---------|
| **section** | Named complex containers | `components: {role: group/role, ...}` |
| **shallowsection** | Fields + limited containers (max 2) | `required: {title: atom/agent_title}, containers: {...}` |
| **schema** | Root -- named sections/shallowsections | `sections: {indirect: section/indirect, ...}` |

## Raw / Resolved / Reduced Stages

Many types exist in three variants tracking pipeline stages:

- **`_raw`**: What the author writes. Contains relative paths and include references to external files.
- **`_resolved`**: After path resolution. All paths absolutized. Includes still present but with absolute paths.
- **`_reduced`**: After include inlining. External file contents merged in. No more include references.

Example:
```
array/execution_instructions_raw.yaml       -- accepts inline steps + include refs
array/execution_instructions_resolved.yaml  -- same but include paths absolutized
array/execution_instructions_reduced.yaml   -- inline steps only (includes expanded)
```

Schema files at the top compose these variants for each pipeline stage:
```
schema/agent-raw-definition.yaml     -- all _raw sections
schema/agent-universal-format.yaml   -- all _resolved sections
schema/agent-universal-render.yaml   -- _reduced sections for rendering
```

## Conditional Fields

Three keywords control conditional presence:

- **`only_when`**: Required when condition is true. FORBIDDEN when false.
- **`always_when`**: Required when condition is true. No constraint when false.
- **`not_when`**: Forbidden when condition is true. No constraint when false.

```yaml
conditional:
  dispatch_batch_size:
    ref: tuple/dispatch_batch_size
    only_when:
      field: dispatch_mode
      equals: batch
```

## Polymorphic Arrays

Arrays and superarrays support `oneOf` for multiple item types:

```yaml
items:
  oneOf: [simplegroup/instruction_step, simplegroup/include_instruction]
```

This is a typed, known shape -- not arbitrary data.

## What NOT to Do

1. **Do not edit Verdandi files to change generated schemas directly.** Edit the YAML, then run Draupnir to regenerate. The JSON Schema files in `schemas/` are derived artifacts.

2. **Do not create new types without understanding the tier boundaries.** A simplegroup cannot reference another simplegroup. An array cannot reference a group. The hierarchy is strict and Draupnir enforces it.

3. **Do not use name-based references.** Every reference is `level/filename` (without `.yaml`). The file path IS the identity.

4. **Do not add fields to a type that violate its tier.** A simplegroup holds fields only (plus tuple and simplearray). If you need nested containers, use group.

5. **Do not create a shallowsection with more than 2 containers.** Use section + group for deeper nesting.

6. **Do not create inline type declarations above the pattern level.** Everything is a file reference.

7. **Do not confuse raw/resolved/reduced with Verdandi processing stages.** These variants describe stages of **agent definition processing**, not stages of Verdandi itself. Verdandi YAML is static data.

## Where to Get Full Context

- **COMPOSITION_RULES.md** (this directory): Formal specification of all levels, their fields, and the composition rule. The authoritative reference.
- **MIGRATION_CHECKLIST.md** (this directory): Maps every old toolcompose file to its Verdandi equivalent. Useful for understanding provenance.
- **Draupnir CLAUDE.md** (`tools/draupnir/CLAUDE.md`): Architectural constraints for the Draupnir compiler that consumes Verdandi.
- **Generated schemas** (`schemas/agent-*.schema.json`): The JSON Schema output. What Verdandi ultimately produces.
- **Agent definitions** (`definitions/agents/`): TOML files that conform to the generated schemas. Shows how Verdandi types appear in practice.
- **scaffold_definition.py** (`tools/scaffold_definition.py`): Generates TOML scaffolds from schemas. Shows how schema structure maps to authoring experience.
