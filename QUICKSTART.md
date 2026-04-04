# Verdandi -- Quickstart

## What This Is

Verdandi is a **typed composition hierarchy** expressed as YAML files. It defines the structural vocabulary for agent tool definitions -- what fields exist, what types they have, how they compose into groups, sections, and complete schemas.

Verdandi is **not a script**. It is a data directory. You do not execute it. Other tools consume it.

## How to Use It

### There is no command to run

Verdandi is static data. You edit YAML files, then run Draupnir to compile them into JSON Schemas:

```bash
# Regenerate all schemas from Verdandi YAML
cd /Users/johnny/.ai/smidja/draupnir && uv run draupnir --all

# Check for drift without writing (dry run)
cd /Users/johnny/.ai/smidja/draupnir && uv run draupnir --check

# List available schemas
cd /Users/johnny/.ai/smidja/draupnir && uv run draupnir --list
```

### Input

YAML files in `{project}/source/` organized into 15 subdirectories (one per composition level). Each file defines a single reusable type. Two projects: `agent-builder` (pipeline schemas) and `agent-output` (rendering surface schemas). Shared primitives live in `core/source/` and are symlinked into each project.

### Output

JSON Schema files written to `{project}/output/*.schema.json`. agent-builder produces 13 pipeline schemas; agent-output produces 3 rendering surface schemas. HTML sidecar inspection files are generated alongside.

### After editing Verdandi YAML

The full cascade is:

1. Edit YAML in `{project}/source/`
2. Run Draupnir to regenerate schemas: `cd /Users/johnny/.ai/smidja/draupnir && uv run draupnir --all`
3. Rebuild Nornir gates to embed updated schemas: `nornir_deploy --build gates`
4. Downstream consumers automatically pick up the new schemas

## How It Works

Each project contains YAML files organized into 15 subdirectories (one per composition level). Each file defines a single reusable type. Higher-level types reference lower-level types by file path (e.g., `field/dispatch_mode` means `field/dispatch_mode.yaml`).

**Draupnir** (`/Users/johnny/.ai/smidja/draupnir/`) is the compiler. It reads all Verdandi YAML files in dependency order, validates them into frozen Pydantic models, and generates JSON Schema files to `{project}/output/`.

The generated JSON Schemas are then consumed by:
- **Nornir** for runtime validation of agent TOML definitions at pipeline boundaries
- **Regin** to auto-generate Pydantic models for the pipeline resolution stages
- **scaffold_definition.py** to generate TOML authoring scaffolds
- **inspect_schema.py** to generate HTML inspection reports

## The Type Hierarchy

15 levels in a membrane architecture. Each level can ONLY reference levels below it.

### Type Tier (structural constraint)

| Level | What It Holds | Example |
|-------|---------------|---------|
| **pattern** | Raw regex | `regex: "^[a-z0-9]+(-[a-z0-9]+)*$"` |
| **atom** | Base type + pattern + exclusions | `type: string, pattern: pattern/kebab` |
| **enum** | Allowed values | `values: [batch, full]` |
| **element** | Atom/enum + runtime contract | `inherits: atom/path_absolute, format: exists` |
| **union** | OneOf composition of fields | `union: [atom/path_relative, atom/path_absolute]` |
| **tuple** | Fixed-length sequence of fields | `items: [atom/integer, atom/integer]` |

### Field Membrane (where meaning attaches)

| Level | What It Holds | Example |
|-------|---------------|---------|
| **field** | Type ref + description + x-extensions | `type: atom/string_text, description: "One-line purpose..."` |

### Container Tier (structural arrangement)

| Level | What It Holds | Example |
|-------|---------------|---------|
| **simplearray** | Variable-length array of one field type | `items: field/role_expertise_item` |
| **simplegroup** | Flat object with required/optional/conditional zones (fields + tuples + simplearrays) | `required: {name: field/param_name}` |
| **array** | Variable-length array of simplegroups | `items: simplegroup/parameter_item` |
| **group** | Object with fields section + containers section | `fields: {required: ...}, containers: {optional: ...}` |
| **superarray** | Variable-length array of groups | `items: group/example_group_raw` |

### Top Level

| Level | What It Holds | Example |
|-------|---------------|---------|
| **section** | Named complex containers | `containers: {required: {metadata: group/metadata}}` |
| **shallowsection** | Fields + limited containers (max 3) | `required: {title: field/agent_title}, containers: {...}` |
| **schema** | Root — uses `sections:` key (deep) or `shallowsections:` key (wide), never both. Each key contains `required:`/`optional:`/`conditional:` zones. | `sections: {required: {indirect: section/indirect, ...}}` |

## Raw / Resolved / Reduced Stages

Many types exist in two or three variants tracking pipeline stages:

- **`_raw`**: What the author writes. Contains relative paths and include references to external files.
- **Resolved (no suffix)**: After path resolution. All paths absolutized. Includes still present but with absolute paths.
- **`_reduced`**: After include inlining. External file contents merged in. No more include references.

Example:
```
array/execution_instructions_raw.yaml       -- accepts inline steps + include refs
array/execution_instructions.yaml           -- same but include paths absolutized
array/execution_instructions_reduced.yaml   -- inline steps only (includes expanded)
```

Schema files at the top compose these variants for each pipeline stage:
```
schema/agent-raw-definition.yaml            -- all _raw sections
schema/agent-universal-format.yaml          -- all resolved/reduced sections
schema/agent-universal-render.yaml          -- reduced sections for rendering
schema/agent-anthropic-render.yaml          -- vendor-specific Anthropic format
```

## Field Files

The `field/` directory (138 files) is the membrane between types and containers. Each field file carries:

- **title**: PascalCase name (used for Python class generation)
- **type**: Reference to the type tier (atom/, enum/, element/, union/, or tuple/)
- **description**: Per-field semantic meaning
- **semantic** (optional): Quality/intent checks
- **constraint** (optional): Cross-field/domain rules
- **validation** (optional): Runtime validation hints

Type narrowing uses double underscore: `field/context_path__any` (raw, accepts relative or absolute) vs `field/context_path__abs` (resolved, absolute only).

## Conditional Fields

Three keywords control conditional presence:

- **`only_when`**: Required when condition is true. FORBIDDEN when false.
- **`always_when`**: Required when condition is true. No constraint when false.
- **`not_when`**: Forbidden when condition is true. No constraint when false.

Each condition uses exactly one predicate -- `equals`, `in`, or `exists` -- they are mutually exclusive.

Value-based conditions match against a field's value:
```yaml
conditional:
  dispatch_batch_size:
    ref: field/dispatch_batch_size
    only_when:
      field: dispatch_mode
      equals: batch          # const check — single value
```

```yaml
conditional:
  some_field:
    ref: field/some_field
    always_when:
      field: output_format
      in: [json, jsonl]      # enum check — any of several values
```

Presence-based conditions match on whether a field exists in the object:
```yaml
conditional:
  resource_timeout:
    ref: field/resource_timeout
    only_when:
      field: resource_url
      exists: true           # presence check — field is present
```

`exists: false` is not supported. Use `not_when` with `exists: true` for the inverse.

Cross-section root traversal (leading dot -- field is in a different section):
```yaml
conditional:
  some_field:
    ref: field/some_field
    only_when:
      field: .indirect.dispatch.dispatch_input_delivery
      equals: streaming
```

The leading dot means "start from the schema root." Without a dot, the path is a sibling in the same object. This distinction matters when a conditional in one section depends on a value defined in another section.

## Polymorphic Arrays

Arrays and superarrays support `oneOf` for multiple item types:

```yaml
items:
  oneOf: [simplegroup/instruction_step, simplegroup/include_instruction]
```

This is a typed, known shape -- not arbitrary data.

## What NOT to Do

1. **Do not edit generated schemas directly.** Edit the YAML, then run Draupnir to regenerate. The JSON Schema files in `schemas/` are derived artifacts.

2. **Do not create new types without understanding the tier boundaries.** A simplegroup cannot reference another simplegroup. An array cannot reference a group. The hierarchy is strict and Draupnir enforces it.

3. **Do not use name-based references.** Every reference is `level/filename` (without `.yaml`). The file path IS the identity.

4. **Do not add fields to a type that violate its tier.** A simplegroup holds fields, tuples, and simplearrays — anything below it in the hierarchy. If you need nested containers (arrays, groups), use group.

5. **Do not create a shallowsection with more than 3 containers.** Use section + group for deeper nesting.

6. **Do not create inline type declarations above the pattern level.** Everything is a file reference.

7. **Do not confuse raw/resolved/reduced with Verdandi processing stages.** These variants describe stages of **agent definition processing**, not stages of Verdandi itself. Verdandi YAML is static data.

8. **Do not reference types at the same or higher level.** References flow strictly downward. A group cannot reference a section. A simplegroup cannot reference another simplegroup. Draupnir rejects these at compile time.

9. **Do not skip the field level.** Containers (simplegroup, group, shallowsection) reference `field/` files, not raw type-tier files. The field membrane is where descriptions and x-extensions live.

## Where to Get Required Context

Read these files first, in this order, when working with Verdandi:

| Priority | File | What It Gives You |
|----------|------|-------------------|
| 1 | `CLAUDE.md` | Context refresh protocol -- common mistakes and how to avoid them |
| 2 | `COMPOSITION_RULES.md` | Formal specification of all 15 levels, their fields, and the composition rule. The authoritative reference. |
| 3 | `CONTEXT_MAP.md` | Multi-project architecture, schema references, consuming tools, directory inventory |
| 4 | `VERDANDI_REDESIGN.md` | Field membrane design rationale |
| 5 | `/Users/johnny/.ai/smidja/draupnir/CLAUDE.md` | Architectural constraints for the Draupnir compiler that consumes Verdandi |

### Quick orientation files

- **Generated schemas** (`{project}/output/*.schema.json`): The JSON Schema output. What Verdandi ultimately produces.
