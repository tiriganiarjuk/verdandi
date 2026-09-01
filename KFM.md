<!-- ═══════════════════════════════════════════════════════════════════════ -->
<!-- DO NOT EDIT THE PRELUDE. Before editing ANY part of this file, read     -->
<!-- /Users/johnny/.ai/CONTEXT_MANAGEMENT_SYSTEM.md — the KFM section        -->
<!-- defines what this file is for and why it is not linked from anywhere.   -->
<!--                                                                          -->
<!-- DO NOT LINK THIS FILE. Not from the CONTEXT_MAP, not from CLAUDE.md,    -->
<!-- not from the README. Its name is uninformative on purpose.              -->
<!--                                                                          -->
<!-- Entries are FAILURE SHAPE with concrete facts routed out. An entry that -->
<!-- names files and functions is a clock: it will outlive them and then     -->
<!-- assert something false to a reader who arrived here because something   -->
<!-- had already gone wrong.                                                  -->
<!-- ═══════════════════════════════════════════════════════════════════════ -->

# KFM — Verdandi

Known failure modes. Each entry records something that actually went wrong here, badly
enough or often enough to be worth writing down: a mistake that kept recurring, one that
was tedious to rediscover, or one that cost hours nobody wanted to spend twice.

**This file is deliberately not linked from anything.** Not the CONTEXT_MAP, not CLAUDE.md,
not the README. It has a name that tells you nothing on purpose. That is not tidiness — it
is the point:

> A list of failure modes read *before* you have a model of the project is not a warning,
> it is a set of suggestions. Naming a mistake supplies it. And a vivid catalogue of nine
> concrete failures shadows the orientation gate it sits above — a reader absorbs the
> stories, feels informed about the project, and skips the documents that would actually
> have oriented them.
>
> Read at the right moment — when something has already gone wrong, or when you are about
> to touch a part with a known history — the same entries are worth what they cost. The
> content is not the problem. The timing is.

The mechanism for *when* a reader gets sent here is undecided. Until it is decided, nothing
points here, and that is intentional rather than an oversight.

**If you have landed here anyway:** read it as history, not as instruction. Nothing below
is a description of the tree as it stands now — check anything concrete against the code
before acting on it.

**Maintenance:** an entry retires when its vector is gone. A trap guarding code that no
longer exists is worse than no trap, because it is the rigid layer asserting something
false. This has already happened once. Nothing automatic enforces it.

---

## Scrapped work — what these cost

Previous sessions in this project:

- Added fields to the wrong level (putting container references in Tier 1 field types)
- Used old directory names from stale documentation (`primitives/`, `composed/`, `groups/`)
- Created types that reference peers at the same level (violating the strictly-lower-level rule)
- Edited generated JSON Schema files directly instead of modifying the Verdandi YAML source
- Assumed the hierarchy has 8 or 12 or 14 levels based on stale docs (it has 15, including the field membrane)
- Conflated `group` and `supergroup` terminology (current names: `group`, `superarray`)

None of them felt uncertain while doing it.

---

## You Will Get These Things Wrong

### Editing Generated Schemas Instead of YAML Source
**Detection:** If you find yourself editing files in `{project}/output/*.schema.json`...
**Why it's wrong:** Schemas are generated artifacts. The chain is: Verdandi YAML → Draupnir → `{project}/output/*.schema.json`. Edit the YAML source, run Draupnir, and the schema updates. Direct edits will be silently overwritten on next generation.
**Recovery:** Find the corresponding YAML in `{project}/source/schema/` for the schema you want to change.

### Referencing Types at the Same or Higher Level
**Detection:** If a YAML file in `simplegroup/` references another `simplegroup/` type, or a `group/` references a `section/` type...
**Why it's wrong:** References flow strictly downward through tiers. Tier 2 types reference Tier 1 types. Tier 3 types reference Tier 1 and Tier 2. Tier 4 references Tier 3 (and sometimes Tier 1/2). Same-level and upward references create circular dependencies that break Draupnir's supply-driven loading.
**Recovery:** Read `COMPOSITION_RULES.md` for the formal spec of what each level can reference.

### Using Old Level Names from Stale Documentation
**Detection:** If you find yourself looking for `primitives/`, `composed/`, `groups/`, `supergroups/`...
**Why it's wrong:** These are pre-rebuild directory names from historical design documents. The current 15 levels are: pattern, atom, enum, element, union, tuple, field, simplearray, simplegroup, array, group, superarray, section, shallowsection, schema. Multiple documentation files still reference the old names — the CONTEXT_MAP flags which docs are stale.
**Recovery:** Check the CONTEXT_MAP's Documentation References section for freshness flags before trusting any doc.

### Adding Wrong Fields to a Level
**Detection:** If you add a field like `containers:` to a `simplegroup` or `fields:` to an `array`...
**Why it's wrong:** Each level has a specific set of allowed fields. Simplegroups have required/optional/conditional zones but no nested containers. Arrays hold simplegroups but not groups. Groups have both fields and containers sections. The COMPOSITION_RULES.md spec defines exactly what each level accepts.
**Recovery:** Read `COMPOSITION_RULES.md` for the level you're editing.

### Forgetting to Run Draupnir After Changes
**Detection:** If you edit Verdandi YAML but don't regenerate schemas...
**Why it's wrong:** Verdandi is consumed by Draupnir. Changes to YAML only take effect when Draupnir regenerates the JSON Schemas. And after schema regeneration, Nornir must be rebuilt to embed the updated schemas. The cascade is: YAML → Draupnir → schemas → Nornir rebuild → pipeline uses new schemas.
**Recovery:** After any YAML edit: `cd /Users/johnny/.ai/smidja/draupnir && uv run draupnir`, then `nornir_deploy --build gates` if validation gates are affected.

### Using the Old Schema Format (flat map + nullable)
**Detection:** If you write a schema YAML with top-level `required:` list, `nullable:` list, or a flat `sections:`/`shallowsections:` map without zones...
**Why it's wrong:** Schema level now uses `required:` / `optional:` / `conditional:` zones inside `sections:` or `shallowsections:`, same as every other level. `nullable` is removed — it was functionally identical to `optional`. See `PLAN_SCHEMA_ZONE_UNIFICATION.md`.
**Recovery:** Read the plan's "Target Shape" section for the new YAML format.

### Writing Cross-Section Conditionals Without Leading Dot
**Detection:** If you write `field: indirect.dispatch.dispatch_mode` without a leading dot...
**Why it's wrong:** Cross-section (root-traversal) paths use a leading dot: `field: .indirect.dispatch.dispatch_mode`. No dot means "sibling in the same object." This disambiguates same-object references from root traversal, which is critical at the schema root level where siblings are sections.
**Recovery:** Add leading dot to all dotted conditional field paths.

_This section grows from experience. If you are corrected on an error that could recur, append it here immediately._

## You Will Get These Things Wrong (salvaged)

### Union Fields Without List Defaults (agent-output)
**Detection:** If you author a `field/` whose type is a union like `union/format_or_pair` and you give it a scalar default (or no default) where the consumer expects a list...
**Why it's wrong:** Union fields such as `format_or_pair` need `list[str]` defaults. Supplying the list default is a required Draupnir extension — without it the generated schema/model defaults are wrong for these union-typed fields.
**Recovery:** Give union fields their `list[str]` default explicitly.

### Shortening Property Names So They No Longer Match TOML Keys
**Detection:** If you rename a schema property to a shorter/cleaner form (e.g. dropping a suffix like `_postscript_visible`)...
**Why it's wrong:** Identity (and other) property names must match the authored TOML keys **exactly** — e.g. `expertise_is_strictly_limited_postscript_visible`, not a shortened variant. A mismatch produces silent key mismatches between the TOML definition and the generated schema.
**Recovery:** Keep property names byte-for-byte identical to the TOML keys; verify by comparing TOML keys against schema property names per section.

### Choosing simplegroup vs group for Variant Sub-Tables (agent-output)
**Detection:** If you are modelling a content-surface variant sub-table and reach for the wrong container level...
**Why it's wrong:** In agent-output, variant sub-tables are **simplegroups when flat** (one level of TOML sub-table) and **groups when nested** (2+ levels). E.g. `security_boundary.framing_variant` and `failure_criteria.abort_stance_variant` use groups (nested TOML sub-tables); `constraints` and `success_criteria` variant sub-tables are direct simplegroups in the shallowsection (flat TOML sub-tables).
**Recovery:** Count the TOML sub-table nesting depth — 1 level → simplegroup, 2+ levels → group.

---
<!-- ═══════════════════════════════════════════════════════════════════════ -->
<!-- EVERYTHING ABOVE THIS LINE IS BEHAVIORAL (human-maintained, empirical) -->
<!-- EVERYTHING BELOW THIS LINE IS DYNAMIC ORIENTATION (agent-regenerated)  -->
<!-- ═══════════════════════════════════════════════════════════════════════ -->
---
