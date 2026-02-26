# Verdandi — Context Refresh Protocol

## STOP. Read this before modifying any YAML files.

You have context from a compaction summary or from earlier in this session. You feel like you understand the composition hierarchy well enough to start editing. **You are almost certainly wrong.**

After compaction, you have a summary — not understanding. The summary preserves level names and file counts but loses the composition rules, the tier containment constraints, and the specific ways each level's YAML structure differs from the others. You will produce YAML that looks right, passes a casual review, and silently violates the hierarchy because the summary told you WHAT levels exist but not HOW they compose.

**This has happened before.** Previous sessions working with this hierarchy have:

- Added fields to the wrong level (putting container references in Tier 1 field types)
- Used old directory names from stale documentation (`primitives/`, `composed/`, `groups/`)
- Created types that reference peers at the same level (violating the strictly-lower-level rule)
- Edited generated JSON Schema files directly instead of modifying the Verdandi YAML source
- Assumed the hierarchy has 8 or 12 levels based on stale docs (it has 14)
- Conflated `group` and `supergroup` terminology (current names: `group`, `superarray`)

None of them felt uncertain while doing it.

**The test:** Can you name all 14 levels in load order and state which tier each belongs to? Can you state what a `simplegroup` can contain that a `group` cannot, and vice versa? If not, you don't have enough context. Read the CONTEXT_MAP below before proceeding.

---
<!-- EVERYTHING ABOVE THIS LINE IS STANDARDIZED (except the failure examples) -->
<!-- EVERYTHING BELOW THIS LINE IS PROJECT-SPECIFIC                           -->
---

## You Will Get These Things Wrong

### Editing Generated Schemas Instead of YAML Source
**Detection:** If you find yourself editing files in `schemas/*.schema.json`...
**Why it's wrong:** Schemas are generated artifacts. The chain is: Verdandi YAML → Draupnir → `schemas/*.schema.json`. Edit the YAML source, run Draupnir, and the schema updates. Direct edits will be silently overwritten on next generation.
**Recovery:** Find the corresponding YAML in `tools/verdandi/` for the schema you want to change.

### Referencing Types at the Same or Higher Level
**Detection:** If a YAML file in `simplegroup/` references another `simplegroup/` type, or a `group/` references a `section/` type...
**Why it's wrong:** References flow strictly downward through tiers. Tier 2 types reference Tier 1 types. Tier 3 types reference Tier 1 and Tier 2. Tier 4 references Tier 3 (and sometimes Tier 1/2). Same-level and upward references create circular dependencies that break Draupnir's supply-driven loading.
**Recovery:** Read `tools/verdandi/COMPOSITION_RULES.md` for the formal spec of what each level can reference.

### Using Old Level Names from Stale Documentation
**Detection:** If you find yourself looking for `primitives/`, `composed/`, `groups/`, `supergroups/`...
**Why it's wrong:** These are pre-rebuild directory names from historical design documents. The current 14 levels are: pattern, enum, atom, element, union, tuple, simplearray, simplegroup, array, group, superarray, section, shallowsection, schema. Multiple documentation files still reference the old names — the CONTEXT_MAP flags which docs are stale.
**Recovery:** Check the CONTEXT_MAP's Documentation References section for freshness flags before trusting any doc.

### Adding Wrong Fields to a Level
**Detection:** If you add a field like `containers:` to a `simplegroup` or `fields:` to an `array`...
**Why it's wrong:** Each level has a specific set of allowed fields. Simplegroups have required/optional/conditional zones but no nested containers. Arrays hold simplegroups but not groups. Groups have both fields and containers sections. The COMPOSITION_RULES.md spec defines exactly what each level accepts.
**Recovery:** Read `tools/verdandi/COMPOSITION_RULES.md` for the level you're editing.

### Forgetting to Run Draupnir After Changes
**Detection:** If you edit Verdandi YAML but don't regenerate schemas...
**Why it's wrong:** Verdandi is consumed by Draupnir. Changes to YAML only take effect when Draupnir regenerates the JSON Schemas. And after schema regeneration, Nornir must be rebuilt to embed the updated schemas. The cascade is: YAML → Draupnir → schemas → Nornir rebuild → pipeline uses new schemas.
**Recovery:** After any YAML edit: `draupnir --all`, then rebuild Nornir if validation gates are affected.

_This section grows from experience. If you are corrected on an error that could recur, append it here immediately._

---
<!-- ═══════════════════════════════════════════════════════════════════════ -->
<!-- EVERYTHING ABOVE THIS LINE IS BEHAVIORAL (human-maintained, empirical) -->
<!-- EVERYTHING BELOW THIS LINE IS DYNAMIC ORIENTATION (agent-regenerated)  -->
<!-- ═══════════════════════════════════════════════════════════════════════ -->
---

## Dynamic Orientation

@./CONTEXT_MAP.md

<!-- CONTEXT_MAP.md provides: purpose, primary references, documentation references
     with freshness flags, schema references, consuming tools, directory inventory,
     context refresh guide, known issues / active work.
     Regenerated by summarizer agents. Do not edit CONTEXT_MAP.md manually. -->
