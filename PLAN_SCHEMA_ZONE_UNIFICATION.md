# Plan: Unify Schema Level with Zone Pattern + Conditional Improvements

## Three Changes, One Plan

This plan covers three related improvements discovered during the same investigation:

1. **Schema zone unification** — make the schema level use the same
   `required:` / `optional:` / `conditional:` zone pattern as every other level
2. **Presence-based conditionals** — add `exists` predicate to `ConditionalWhen`,
   enabling conditions based on property presence instead of only value matching
3. **Leading-dot cross-section paths** — disambiguate same-object references from
   root-level traversal paths using a leading dot convention

All three touch the conditional system. Doing them together avoids reworking the
same code multiple times.

---

## Part A: Schema Zone Unification

### Problem

The schema level is the only Verdandi level that uses a different structural pattern.
Every other level uses `required:` / `optional:` / `conditional:` zones to declare
membership semantics. Schema uses a flat key-value map with separate `required:` and
`nullable:` lists.

Current:

```yaml
required:          # separate list of names
  - dispatcher
  - identity
nullable:          # another separate list of names
  - implicit_grants
  - examples
shallowsections:   # flat map, no zone semantics
  identity: shallowsection/identity
  implicit_grants: shallowsection/implicit_grants
```

### Analysis: Nullable Is Dead Weight

Investigation confirmed `nullable` adds no value:

1. **Generated Pydantic models collapse nullable and optional.** Regin's
   `generate_structures.py` produces `SomeSection | None = None` for both.
   The processor sees `None` either way.

2. **TOML has no null.** Agent definitions are authored in TOML, which cannot
   express `null`. Nullable is impossible at the entry point.

3. **Pipeline processors don't distinguish.** Every nullable field in the
   generated `AgentUniversalRender` model uses `| None = None` (row 3).
   No `| None` without default (row 2) exists anywhere.

4. **The oneOf wrapping is noise.** `oneOf: [schema, {type: null}]` adds
   structural complexity that nothing consumes.

### Target Shape

```yaml
# Deep composition (sections:)
title: Agent Raw Definition
version: 3
description: ...
sections:
  required:
    indirect: section/indirect
    task: section/task
    security: section/security_raw
    execution: section/execution_raw
    criteria: section/criteria_raw

# Wide composition (shallowsections:)
title: Agent Universal Render
version: 1
description: ...
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

# Conditional sections (when needed)
shallowsections:
  required:
    identity: shallowsection/identity
  optional:
    examples: shallowsection/examples_reduced
  conditional:
    writing_output:
      ref: shallowsection/writing_output
      only_when:
        field: enforcement.enforcement_output_tool.tool_format
        exists: true
```

### Schema-Level Conditional: Always Cross-Section

At every other level, conditionals can reference sibling fields in the same
object. At the schema root level, the "siblings" are sections — complex objects,
not leaf fields with matchable values. Every conditional at the schema level
MUST use a dotted path to reach into a section to find a field.

This is NOT a limitation of JSON Schema. Cross-section conditionals with
`if/then/else` at the document root already work — they're implemented in
`build_cross_section.py`. Schema-level section conditionals are actually the
SIMPLER case because the `then`/`else` targets a top-level property name
directly instead of reaching through nested objects.

The implementation routes all schema-level conditionals through the cross-section
builder (`build_root_conditional` / `build_if_nested`), not the same-section
builder (`build_conditional_entry`).

---

## Part B: Presence-Based Conditionals

### Problem

The current conditional system only supports value matching:

```yaml
only_when:
  field: dispatch_mode
  equals: batch         # value-based: const check

only_when:
  field: output_name_known
  in: [partially, unknown]  # value-based: enum check
```

There is no way to express "if this field/container/section exists, then that
field is required." The `if` clause always checks a value with `const` or `enum`.

This has made schemas cumbersome to write. Many conditions are naturally about
presence, not value.

### JSON Schema Supports This

`required` is valid anywhere a schema is accepted, including inside `if` clauses.
This is standard JSON Schema Draft 2020-12:

```json
{
  "if": { "required": ["enforcement_output_tool"] },
  "then": { "required": ["writing_output"] },
  "else": { "properties": { "writing_output": false } }
}
```

Presence-based `if` clauses also work with nested paths for cross-section:

```json
{
  "if": {
    "properties": {
      "enforcement": {
        "required": ["enforcement_output_tool"]
      }
    }
  },
  "then": { "required": ["writing_output"] }
}
```

`if/then/else` is the superset — it handles value-based, presence-based,
same-object, and cross-section conditions. `dependentRequired` is a more
concise spelling for the simple same-object presence case, but `if/then/else`
covers everything.

A previous LLM claimed this was impossible. It is not. It is a basic,
standard feature.

### Target Shape

New `exists` predicate alongside `equals` and `in`:

```yaml
# Value-based (existing)
only_when:
  field: dispatch_mode
  equals: batch

# Value-based set (existing)
only_when:
  field: output_name_known
  in: [partially, unknown]

# Presence-based (new)
only_when:
  field: enforcement_output_tool
  exists: true
```

`exists: true` means "field is present in the object." The `only_when` /
`always_when` / `not_when` keywords handle the directional logic:

- `only_when` + `exists: true` = required when present, forbidden when absent
- `always_when` + `exists: true` = required when present, no constraint when absent
- `not_when` + `exists: true` = forbidden when present, no constraint when absent

`exists: false` is intentionally not supported — use `not_when` with
`exists: true` instead.

### This Applies at Every Level

Presence-based conditionals are not schema-specific. They work at every level
that supports conditionals: simplegroup, group (fields and containers),
shallowsection, section (with this plan's additions), and schema.

---

## Part C: Leading-Dot Cross-Section Paths

### Problem

The current system distinguishes same-section from cross-section conditionals
by checking for dots: `"." in when.field`. This works at group level because
sibling fields never have dots in their names. But at the schema root level,
`security` is both a sibling property (a section in the same object) AND a
section you could traverse into. The heuristic breaks.

```yaml
# At schema root level — is this a sibling or a traversal?
only_when:
  field: security         # ambiguous
  exists: true
```

### Solution: Leading Dot Convention

A leading dot means "start from root." No dot means "look in the current object."
Same convention as absolute vs relative paths.

```yaml
# Sibling field in the same object (same-section)
only_when:
  field: dispatch_mode
  equals: batch

# Root-level traversal (cross-section) — leading dot
only_when:
  field: .indirect.dispatch.dispatch_input_delivery
  equals: file

# Schema root level — sibling section (same-object presence check)
only_when:
  field: enforcement_output_tool
  exists: true

# Schema root level — traversal into a section
only_when:
  field: .security.security_io.io_output_file
  exists: true
```

### Current Detection (fragile)

```python
def is_cross_section(when):
    return when is not None and "." in when.field
```

### New Detection (unambiguous)

```python
def is_cross_section(when):
    return when is not None and when.field.startswith(".")
```

### Migration

22 cross-section references across 8 YAML files need a leading dot added:

- `group/security_io.yaml` (6 references)
- `group/security_io_raw.yaml` (6 references)
- `group/enforcement_output_tool.yaml` (5 references)
- `group/security_schema.yaml` (1 reference)
- `group/security_schema_raw.yaml` (1 reference)
- `field/output_schema__x_any.yaml` (1 reference)
- `field/output_schema__x_abs.yaml` (1 reference)

Plus `build_if_nested` strips the leading dot before splitting into segments.

---

## Changes Required

### 1. ConditionalWhen Model (`structures/conditionals.py`)

**Add** `exists` predicate. **Update** validator to accept exactly one of
`equals`, `in`, or `exists`:

```python
class ConditionalWhen(BaseModel):
    model_config = ConfigDict(frozen=True, populate_by_name=True)

    field: str
    equals: str | int | bool | None = None
    in_: tuple[str | int | bool, ...] | None = Field(None, alias="in")
    exists: bool | None = None    # NEW

    @model_validator(mode="after")
    def exactly_one_predicate(self) -> "ConditionalWhen":
        count = sum([
            self.equals is not None,
            self.in_ is not None,
            self.exists is not None,
        ])
        if count != 1:
            msg = "exactly one of 'equals', 'in', or 'exists' must be set"
            raise ValueError(msg)
        if self.exists is not None and self.exists is not True:
            msg = "'exists' must be true (use not_when for the inverse)"
            raise ValueError(msg)
        return self
```

### 2. Same-Section Conditional Builder (`functions/pure/converters/conditional.py`)

**Update** `build_if_condition()` to handle `exists`:

```python
def build_if_condition(when: ConditionalWhen) -> SchemaCondition:
    if when.exists:
        return SchemaCondition(required=(when.field,))
    if when.equals is not None:
        constraint = SchemaConst(const=when.equals)
    else:
        constraint = SchemaEnumConstraint(enum=when.in_)
    return SchemaCondition(properties={when.field: constraint})
```

No other changes to the same-section builder. `build_only_when`, `build_always_when`,
`build_not_when` all work unchanged — the `then`/`else` clauses are independent
of the `if` clause type.

### 3. Cross-Section Conditional Builder (`functions/pure/build_cross_section.py`)

**Update** `build_if_nested()` to handle `exists`:

```python
def build_if_nested(when: ConditionalWhen) -> SchemaNestedPath:
    segments = tuple(when.field.split("."))
    field_name = segments[-1]
    path_segments = segments[:-1]
    if when.exists:
        leaf = SchemaNestedPath(required=(field_name,))
    elif when.equals is not None:
        constraint = SchemaConst(const=when.equals)
        leaf = SchemaNestedPath(
            properties=NestedPathProperties({field_name: constraint}),
        )
    else:
        constraint = SchemaEnumConstraint(enum=when.in_)
        leaf = SchemaNestedPath(
            properties=NestedPathProperties({field_name: constraint}),
        )
    return wrap_nested_path(path_segments, leaf)
```

**Extend** `collect_schema_conditionals()` to also collect conditionals from
the schema-level conditional zone (not just from groups within sections).

### 4. Schema Pydantic Models (`structures/top_level.py`)

**Add** zone container types:

```python
class SchemaSectionConditionalEntry(BaseModel):
    """Conditional section entry for schema section zones."""
    model_config = ConfigDict(frozen=True)
    ref: SectionOnlyRef
    only_when: ConditionalWhen | None = None
    always_when: ConditionalWhen | None = None
    not_when: ConditionalWhen | None = None
    # exactly_one_condition validator (same as ShallowFieldConditionalEntry)

class SchemaShallowConditionalEntry(BaseModel):
    """Conditional shallowsection entry for schema shallowsection zones."""
    model_config = ConfigDict(frozen=True)
    ref: ShallowsectionOnlyRef
    only_when: ConditionalWhen | None = None
    always_when: ConditionalWhen | None = None
    not_when: ConditionalWhen | None = None
    # exactly_one_condition validator

class SchemaSections(BaseModel):
    """Zone container for schema sections: key."""
    model_config = ConfigDict(frozen=True)
    required: dict[str, SectionOnlyRef] | None = None
    optional: dict[str, SectionOnlyRef] | None = None
    conditional: dict[str, SchemaSectionConditionalEntry] | None = None

class SchemaShallowsections(BaseModel):
    """Zone container for schema shallowsections: key."""
    model_config = ConfigDict(frozen=True)
    required: dict[str, ShallowsectionOnlyRef] | None = None
    optional: dict[str, ShallowsectionOnlyRef] | None = None
    conditional: dict[str, SchemaShallowConditionalEntry] | None = None
```

**Modify** Schema model — remove `required` and `nullable`, use zone containers:

```python
class Schema(BaseModel):
    model_config = ConfigDict(frozen=True)
    title: str
    version: int
    description: str
    sections: SchemaSections | None = None
    shallowsections: SchemaShallowsections | None = None
    # KEEP: exactly_one_section_type validator
```

### 5. Section Containers — Add Conditional Zone

While we're unifying, add the `conditional` zone to `SectionContainers` too.
There's no JSON Schema reason sections can't have conditional containers — it
was just never implemented. Same structural pattern as groups:

```python
class SectionContainers(BaseModel):
    model_config = ConfigDict(frozen=True)
    required: dict[str, ComplexContainerRef] | None = None
    optional: dict[str, ComplexContainerRef] | None = None
    conditional: dict[str, SectionContainerConditionalEntry] | None = None  # NEW
```

Update `build_section` to call `collect_conditional` in addition to
`collect_required_optional`.

### 6. Schema YAML Files (13 files in `agent-builder/source/schema/`)

**Deep schemas (11 files) — all sections required:**
- agent-raw-definition, agent-paths-resolved, agent-permissions-resolved,
  agent-instructions-reduced, agent-examples-reduced, agent-guardrails-reduced,
  agent-success-reduced, agent-execution-merged, agent-criteria-merged,
  agent-includes-merged, agent-universal-format

Transform: move flat `sections:` map under `sections: required:`, remove
top-level `required:` list.

**Shallow schemas (2 files) — mixed required/optional:**
- agent-universal-render, agent-anthropic-render

Transform: split flat `shallowsections:` map into `required:` and `optional:`
zones. Remove top-level `required:` and `nullable:` lists.

### 7. Exporter (`functions/impure/exporter.py`)

**Rewrite** `build_schema_properties()` to use zone primitives:

```python
def build_schema_properties(schema, reverse, shared, defs):
    zones = schema.sections or schema.shallowsections
    build_fn = lambda model: build_or_ref(model, reverse, shared, defs)
    properties, required_keys = collect_required_optional(
        zones.required, zones.optional, build_fn,
    )
    cond_props, conditionals = collect_conditional(
        zones.conditional, build_fn,
    )
    properties.update(cond_props)
    return properties, required_keys, conditionals
```

**Update** `export_schema()` to merge schema-level conditionals with
cross-section conditionals for the `all_of` field.

**Remove** nullable wrapping logic and `SchemaNullableObject` /
`SchemaNullType` imports.

### 8. Zone Primitive Compatibility (`functions/pure/converters/zone.py`)

`collect_conditional` takes `dict[str, ConditionalEntry]` where `ConditionalEntry`
is a Protocol in `structures/conditionals.py` with properties: `ref`, `only_when`,
`always_when`, `not_when`.

The new conditional entry types (`SchemaSectionConditionalEntry`,
`SchemaShallowConditionalEntry`, `SectionContainerConditionalEntry`) must
satisfy this protocol. Since they all share the same field names, they
automatically satisfy it. Verify with tests.

**Important:** Schema-level conditionals are always cross-section (dotted paths).
The current `collect_conditional` + `build_conditional_entry` pipeline returns
`None` for cross-section conditionals (they're hoisted separately). Schema-level
conditionals need to go through the cross-section builder path directly.

Options:
a) Process schema-level conditional zone separately from `collect_conditional`,
   routing directly to `build_root_conditional`
b) Extend `collect_conditional` to return cross-section conditionals separately

Option (a) is simpler — handle the schema conditional zone in the exporter
directly rather than overloading the zone primitive.

### 9. Dead Code Removal

- Remove `SchemaNullableObject` and `SchemaNullType` from `schema_output.py`
  (verify no other consumers first)
- Remove `nullable` handling in exporter

### 10. Tests

**Update existing:**
- Schema loading tests using old flat structure
- Export tests checking for nullable wrapping
- ConditionalWhen tests (add exists predicate cases)

**Add new:**
- Schema model validation with zone structure
- Schema model rejects old flat format (regression guard)
- Conditional schema section entry validation
- Presence-based conditional at simplegroup/group level
- Presence-based conditional at schema level (cross-section)
- Export produces correct `required` array from zone membership
- Export produces correct `allOf` from conditional zone entries
- Export produces correct `if: { required: [...] }` for exists predicates

### 11. Documentation

- `tools/verdandi/COMPOSITION_RULES.md` — update schema level spec, add
  `exists` predicate to conditional docs
- `tools/verdandi/QUICKSTART.md` — update schema examples, add exists examples
- `context/verdandi/` — relevant context docs
- `tools/draupnir/CONTEXT_MAP.md` — update Schema class description
- `tools/verdandi/CONTEXT_MAP.md` — update schema level description

---

## Execution Sequence

### Phase 1: Leading-Dot Cross-Section Paths

No dependencies. Can land first. Smallest change, broadest impact on
correctness.

1. Update `is_cross_section()` — check `startswith(".")` instead of `"." in`
2. Update `build_if_nested()` — strip leading dot before splitting
3. Add leading dot to all 22 cross-section field references in YAML files
4. Run draupnir to verify output is identical (the dot is stripped before
   emission, so JSON Schema output should not change)
5. Run guardrails and tests

### Phase 2: Presence-Based Conditionals (exists predicate)

Independent of schema zone unification. Can land after Phase 1.

1. Update `ConditionalWhen` model — add `exists` field, update validator
2. Update `build_if_condition` (same-section) — handle `exists`
3. Update `build_if_nested` (cross-section) — handle `exists`
4. Add tests for exists predicate at all levels
5. Run guardrails

### Phase 3: Schema Zone Unification

Depends on Phases 1 and 2.

1. Add zone container types to `structures/top_level.py`
2. Modify Schema model (remove `required`, `nullable`)
3. Add conditional zone to `SectionContainers`
4. Rewrite 13 schema YAML files
5. Verify Draupnir can load them
6. Rewrite exporter to use zone primitives
7. Handle schema-level conditional → cross-section builder routing
8. Update `build_section` to call `collect_conditional`
9. Remove nullable dead code
10. Run guardrails

### Phase 4: Regeneration + Downstream

1. `draupnir --all` — regenerate all schemas
2. Compare output: required sections stay required, former-nullable become
   plain optional (no oneOf wrapping)
3. Regin `generate_structures.py` — regenerate Pydantic models
4. Verify generated models are behaviorally equivalent
5. Run full test suites

### Phase 5: Documentation

1. Update COMPOSITION_RULES.md
2. Update QUICKSTART.md
3. Update context docs and CONTEXT_MAPs

---

## Risk Assessment

**Low risk — exists predicate:** The change to `ConditionalWhen` and the two
`if` clause builders is surgical. The `then`/`else` logic is completely
independent of the `if` clause type. Existing value-based conditionals are
unaffected.

**Low risk — schema zone unification:** nullable → optional produces identical
downstream Pydantic models. The zone pattern is well-established across all
other levels.

**Medium risk — schema-level conditional zone:** New capability with no
immediate consumers. Code path is the cross-section builder, which is
well-tested. But the routing (schema conditional zone → cross-section builder
instead of same-section builder) is new plumbing.

**Watch for:** `collect_conditional` type compatibility with new entry types.
The `ConditionalEntry` Protocol in `structures/conditionals.py` should match
automatically since all entries share the same field names, but verify.

---

## What This Does NOT Change

- Shallowsection and section models — untouched (except adding conditional
  zone to SectionContainers)
- Group models — untouched
- All levels below section — untouched
- The `version` field on Schema — stays
- The `exactly_one_section_type` validator — stays
- Cross-section conditional collection from groups — stays, merged with
  schema-level conditionals
- Fragment export — untouched
- `dependentRequired` — not added (if/then/else with exists covers all cases
  including cross-section; dependentRequired is same-object only)
