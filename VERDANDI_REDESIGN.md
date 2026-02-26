# Verdandi Redesign — The Field Membrane

**Status:** Design proposal
**Date:** 2026-02-26

---

## The Problem

Verdandi's streamlining from toolcompose achieved type reuse but lost per-field semantic descriptions.

In toolcompose, every field was its own primitive file:

```yaml
# toolcompose/primitives/processing_output_description.yaml
name: processing_output_description
type: string
requirement: required
description: >
  Human-readable description of the output data the agent produces. Describes
  what the output contains and its downstream purpose. The mechanical format
  is captured by processing_output_file_format — this field captures domain
  meaning, but repeating format for human readability is acceptable.
  Example: "One assessment record per input entry with quality scores and failure annotations"
```

Verdandi collapsed those ~60 unique primitives into shared types:

```yaml
# Current Verdandi group — three different fields, one generic type
fields:
  required:
    processing_output_description: atom/string_plain
  optional:
    processing_output_file_name_template: atom/string_plain
    processing_output_file_name_instruction: atom/string_plain
```

The type constraint survived (`string`, not a path). The per-field descriptions — what each field means, what content it expects, what purpose it serves — had nowhere to go. `atom/string_plain` says "Plain string that is not a path." It says nothing about output descriptions vs filename templates vs naming instructions.

The $defs mechanism makes this loss impossible to ignore. All three fields become `{"$ref": "#/$defs/atom__string_plain"}` — identical opaque pointers where there were once three distinct primitives each explaining exactly what content belongs there.

Additionally, x-extensions (`semantic:`, `x-constraint:`, `validation:`) have no home. They are per-field concerns — too specific for shared atoms (which are reused across many fields), too invariant for containers (which vary across pipeline stages). They need a stable per-field identity to attach to.

---

## The Solution: Field as Membrane

Introduce a `field` level that serves as the boundary between types and containers.

**A field is where a type and a description merge into a field value.**

```yaml
# field/processing_output_description.yaml
type: atom/string_plain
description: >
  Human-readable description of the output data the agent produces.
  Describes what the output contains and its downstream purpose.
  The mechanical format is captured by processing_output_file_format —
  this field captures domain meaning.
```

Groups reference fields instead of raw types:

```yaml
# group/processing.yaml (after redesign)
fields:
  required:
    processing_output_description: field/processing_output_description
  optional:
    processing_output_file_name_template: field/processing_output_file_name_template
    processing_output_file_name_instruction: field/processing_output_file_name_instruction
```

---

## The Membrane Property

The field level is the membrane between two tiers with fundamentally different concerns:

```
TYPE TIER (structural constraint)
  pattern → enum → atom → element → union, tuple

                    ┃ field ┃          ← the membrane

CONTAINER TIER (structural arrangement)
  simplearray → simplegroup → array → group →
  superarray → section → shallowsection → schema
```

- **Lowest import level for containers** is `field`
- **Highest inclusion level for types** is also `field`

Everything below field is structural constraint — what values are valid. Everything above field is structural arrangement — how fields compose into objects and arrays. Field itself is where meaning attaches.

---

## Tuples and Unions: Both Type-Level

Tuples are reclassified from Simple Containers to Type Tier.

In practice, tuples are only used as field types — `tuple/dispatch_batch_size` is `[integer, integer]`. A tuple is a type composition (fixed-length typed sequence), not a container that holds named fields. It's the same kind of thing as a union — a way to combine simple types into a richer type.

A field can be typed as `union/path_any` or `tuple/dispatch_batch_size`. Neither is a container; both are types that fields reference.

---

## X-Extensions at the Field Level

The field level is the right place for x-extensions. They are per-field quality and validation rules — too specific for shared atoms, too invariant for container-level placement.

```yaml
# field/processing_output_description.yaml
type: atom/string_plain
description: >
  Human-readable description of the output data the agent produces.
  Describes what the output contains and its downstream purpose.
semantic:
  intent: Describe output content and downstream purpose
  severity: warning
  checks:
    - Must describe data content, not just format
```

Three extension types attach at field level:

| Extension | Purpose | Example |
|-----------|---------|---------|
| `semantic:` | Quality and intent checks (non-schema-validatable) | "Must describe data content, not just format" |
| `constraint:` | Cross-field or domain rules (non-schema-validatable) | "Must reference a field that exists in the same container" |
| `validation:` | Runtime validation hints | "Should not simply restate the file format" |

These travel with the field identity. `processing_output_description` has the same semantic expectations regardless of which group or pipeline stage uses it.

---

## Type Narrowing

Pipeline stages narrow field types using a standard naming convention:

```
field/context_path__path_any         → raw definition stage (relative or absolute)
field/context_path__path_absolute    → after path resolution (absolute only)
```

The double underscore separates field identity from type variant:

- `field/context_path__path_any` — field `context_path`, typed as `union/path_any`
- `field/context_path__path_absolute` — same field, typed as `atom/path_absolute`

Both variants are concrete YAML files. No implicit narrowing, no inference. The narrowed variant keeps the same semantic description while constraining the type:

```yaml
# field/context_path__path_any.yaml
type: union/path_any
description: >
  Filesystem path to a context resource.
  Auto-derives read permission at resolution time.

# field/context_path__path_absolute.yaml
type: atom/path_absolute
description: >
  Filesystem path to a context resource (absolutized).
  Auto-derives read permission at resolution time.
```

Groups at different pipeline stages reference the appropriate variant:

```yaml
# simplegroup/context_item_raw.yaml
required:
  context_label: field/context_label
  context_path: field/context_path__path_any

# simplegroup/context_item_resolved.yaml
required:
  context_label: field/context_label
  context_path: field/context_path__path_absolute
```

---

## Draupnir Disassembly

Draupnir cracks open each field reference and produces both layers in the JSON Schema output:

```json
"processing_output_description": {
  "$ref": "#/$defs/atom__string_plain",
  "description": "Human-readable description of the output data the agent produces..."
}
```

- The `$ref` carries the full structural constraint (type, pattern, `not` exclusions) from the underlying atom/enum/element/union
- The `description` carries the per-field semantic meaning from the field definition
- X-extensions are emitted as sibling properties (`x-semantic`, `x-constraint`, `x-validation`)

The type-level $defs deduplication still works. `atom/string_plain` remains a single $def referenced by many fields. But each field's property in the schema now carries its own description and extensions — the information that toolcompose had and Verdandi lost.

---

## Field YAML Structure

```yaml
# Minimal field — type + description
type: atom/string_plain              # required: reference to type tier
description: >                       # required: per-field semantic meaning
  What this field represents and what content belongs here.

# Field with x-extensions
type: atom/string_plain
description: >
  What this field represents and what content belongs here.
semantic:                            # optional: quality/intent checks
  intent: What this field should achieve
  severity: warning
  checks:
    - Specific quality check
  anti_patterns:
    - What to avoid
constraint:                          # optional: cross-field/domain rules
  rule: relationship_description
  severity: error
  reason: Why this constraint exists
validation:                          # optional: runtime validation hints
  instruction: What to verify at runtime
  severity: warning
```

| Field | Required | Description |
|-------|----------|-------------|
| type | yes | Reference to type tier: atom/, enum/, element/, union/, or tuple/ |
| description | yes | Per-field semantic meaning — what this field represents |
| semantic | no | Quality and intent checks (emitted as `x-semantic`) |
| constraint | no | Cross-field or domain rules (emitted as `x-constraint`) |
| validation | no | Runtime validation hints (emitted as `x-validation`) |

---

## Revised Hierarchy

### Before (14 levels, 4 tiers)

```
Tier 1 Fields:      pattern → enum → atom → element → union
Tier 2 Simple:      tuple → simplearray → simplegroup
Tier 3 Complex:     array → group → superarray
Tier 4 Top:         section → shallowsection → schema
```

### After (15 levels, membrane architecture)

```
Type Tier:          pattern → enum → atom → element → union, tuple
Field Membrane:     field
Container Tier:     simplearray → simplegroup → array → group →
                    superarray → section → shallowsection → schema
```

**Changes:**
- `tuple` moves from Simple Containers to Type Tier (alongside `union`)
- `field` inserted as new level between types and containers
- Containers reference `field/` instead of raw type references
- Fields carry descriptions and x-extensions that were previously lost

---

## Migration Impact

### Verdandi YAML files

- New `field/` directory with one YAML per field identity (+ narrowing variants)
- Existing containers updated to reference `field/` instead of atom/enum/element/union
- Type-tier files unchanged
- Estimated field count: ~60-80 files (roughly matching old toolcompose primitive count, plus narrowing variants)

### Draupnir

- New `field` entry in Level enum (load order between tuple and simplearray)
- New Pydantic model for field level (type ref + description + optional x-extensions)
- Builder disassembles field into `$ref` + `description` + x-extension siblings
- Existing $defs mechanism unchanged — types still deduplicate at the atom/enum level

### Downstream consumers

- Generated JSON Schemas gain per-field descriptions (strictly additive)
- inspect_schema.py — field descriptions appear in HTML sidecars automatically
- scaffold_definition.py — field descriptions improve generated TOML scaffolds
- Nornir — recompile to embed updated schemas (standard procedure)

---

## What This Recovers

| Capability | Toolcompose | Current Verdandi | After Redesign |
|-----------|-------------|------------------|----------------|
| Type reuse | No (each primitive unique) | Yes ($defs) | Yes ($defs) |
| Per-field descriptions | Yes (in primitive files) | Lost | Yes (in field files) |
| X-extensions | Scattered | Lost (no home) | Yes (in field files) |
| Type narrowing | Implicit (different primitives) | Implicit (different type refs) | Explicit (naming convention) |
| Compile-time enforcement | No | Yes (Draupnir) | Yes (Draupnir) |
