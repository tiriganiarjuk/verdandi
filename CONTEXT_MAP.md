<!-- ═══════════════════════════════════════════════════════════════════════ -->
<!-- DO NOT EDIT THIS BLOCK. Before editing ANY part of this file, you MUST  -->
<!-- first read /Users/johnny/.ai/CONTEXT_MANAGEMENT_SYSTEM.md — it defines  -->
<!-- what this file is for and what may NOT go in it.                        -->
<!--                                                                          -->
<!-- This file is a MAP (a router), not a source. Its only job is to tell    -->
<!-- you WHERE the context lives — it never holds the context, state, or     -->
<!-- progress itself. Reading this map does NOT orient you; you are NOT      -->
<!-- oriented until you have read the Orientation-gate docs it points to, in -->
<!-- the order given.                                                         -->
<!--                                                                          -->
<!-- THE WHITELIST RULE: if a doc is not in this map, it is not for           -->
<!-- orientation — do not read it. A stale doc is UNLINKED or DELETED, never -->
<!-- flagged in place. No freshness tags. No "do-not" guards. No state.       -->
<!-- Behavioral rules live in CLAUDE.md, not here.                            -->
<!--                                                                          -->
<!-- This file is LOW-CHURN. It changes only when how-to-get-oriented        -->
<!-- changes — i.e. a doc is added or removed. It is not a worklog and is    -->
<!-- not regenerated per session.                                            -->
<!-- ═══════════════════════════════════════════════════════════════════════ -->

# CONTEXT_MAP — Verdandi

## Purpose

A hand-authored YAML type hierarchy — the source of every shape in the agent pipeline, and the schema by which all other schemas are written.

---

## Orientation gate — read these, in this order, to be oriented

> **Verdandi has nothing to execute.** No package manifest, no entry point, no program.
> Your default frame will send you looking for one; there isn't one, and the twenty minutes
> spent are a known tax. It is data. Draupnir is the only thing that reads it.

1. `~/.ai/smidja/galdr/context/AGENT_BUILD_SYSTEM.md` — why the pipeline exists at all: determinism as the goal, the double gate, and what is being built toward. This is the part that cannot be recovered by reading a tree, and inferring it from the structure has gone wrong every time it has been tried.
2. `~/.ai/smidja/galdr/context/VERDANDI_ELEMENT.md` — Verdandi's own reason for being: why an LLM cannot be trusted to write a schema, why the field membrane is mandatory, why the constraint-bearing type tier is immutable at the filesystem level, and why the levels accreted rather than being designed.
3. `COMPOSITION_RULES.md` — the formal specification: what keys each level accepts and what it may reference. The authority for authoring.

> Read the next one for its *prohibitions*, not its overview. The "What NOT to Do" list is
> the part that carries information you cannot derive.

4. `QUICKSTART.md` — the operational rules, stage variants, and the things that look reasonable and are forbidden.

**Proof:** before authoring or changing any level file, state (a) what the field membrane buys that a direct type reference does not, (b) why references may only flow downward and what that makes impossible, and (c) one thing you would have done that the level system forbids. Generic acknowledgment means you are not oriented — go back and read.

---

## Refresh guide — want X → read Y

| Need | Read |
|------|------|
| what keys a level accepts / may a group contain a group | `COMPOSITION_RULES.md` — note it has drifted from the files in places; where they disagree, the files are what the compiler consumes, and the disagreement is worth reporting |
| the forbidden-but-plausible moves | `QUICKSTART.md` § What NOT to Do |
| why the field membrane exists | `VERDANDI_REDESIGN.md`, and `context/VERDANDI_ELEMENT.md` for the shorter form |
| why schema zones are required/optional/conditional | `PLAN_SCHEMA_ZONE_UNIFICATION.md` |
| what consumes this / what happens downstream | `context/DRAUPNIR_ELEMENT.md`, then `context/NORNIR_GATES_ELEMENT.md` |
| what levels and types exist right now | read live — `ls */source/*/`. Counts are not stored anywhere, because a stored count is wrong by the next commit |
| what a generated schema looks like | `{project}/output/*.schema.json` — derived artifacts; never edit them |
| the full forward cascade after an edit | `context/AGENT_BUILD_SYSTEM.md` § The forward cascade |
| current state / what's left to build | `TODO.md` (+ `git log`) |
