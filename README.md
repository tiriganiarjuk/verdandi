# Verdandi

A universal schema composition system. Defines complex validated data structures through a 15-level type hierarchy — from regex patterns up through conditional, zone-aware containers to complete schema documents.

**Currently used by the agent definition pipeline:** Verdandi → [Draupnir](https://github.com/johnny111272/draupnir) → [Nornir](https://github.com/johnny111272/nornir) Gates → [Regin](https://github.com/johnny111272/regin) → Galdr

## What This Is

Verdandi is not an agent-specific tool. It is a general-purpose type composition system that happens to have its first project defining agent schemas. The architecture separates universal type primitives (in `core/`) from project-specific compositions (in `agent-builder/` and any future projects).

The system consists of approximately 350 YAML files organised into a 15-level composition hierarchy across four tiers:

- **Type tier** — patterns (regex), atoms (constrained base types), enums, elements (atoms with runtime format contracts), unions (oneOf composition), tuples (fixed-length typed sequences)
- **Field membrane** — the mandatory boundary between types and containers. Every container references field files, never raw types. This is where per-field descriptions, defaults, semantic intent checks, and cross-field constraints attach to structural types
- **Container tier** — simplearrays, simplegroups, arrays, groups, superarrays. Containers compose fields into objects and collections, with required/optional/conditional zones controlling field presence
- **Top level** — sections (deep composition of containers), shallowsections (wide composition mixing fields and containers), and schemas (the root)

References flow strictly downward — no same-level or upward references. Draupnir enforces tier boundaries at compile time.

## Why It Exists Separately

The type system is the foundation that everything else builds on. Agent definitions are one use case. Any domain that needs validated, composable data structures with conditional field presence, cross-section dependencies, and pipeline-stage type narrowing can define a project in Verdandi and get the full compilation pipeline — JSON Schemas, compiled gate validators, and generated Pydantic models — for free.

The separation between `core/` (universal primitives like path types, string constraints, boolean flags) and project directories (domain-specific compositions) means new projects reuse proven type definitions rather than rebuilding them.

## Key Concepts

**Field membrane.** Containers never reference raw type-tier files directly. The field level wraps types with per-field descriptions, defaults, semantic intent checks, and constraint rules. This is what gives structural types their meaning in context — a `string_prose` atom means nothing on its own, but `field/agent_description` with semantic checks and anti-pattern detection carries the full specification.

**Conditional field presence.** Three directional keywords control whether fields appear based on sibling or cross-section field values: `only_when` (required when true, *forbidden* when false), `always_when` (required when true, unconstrained when false), `not_when` (forbidden when true, unconstrained when false). Cross-section references use leading-dot paths for root traversal.

**Pipeline stage narrowing.** Type variants (via double-underscore suffixes) let the same logical field carry different type constraints at different pipeline stages. A path field accepts relative or absolute paths in the raw stage, then narrows to absolute-only after resolution. The looser type drops away as data matures through the pipeline.

## Usage

Verdandi is edited by hand. After editing YAML, regenerate schemas through the downstream pipeline:

```bash
cd ../draupnir && uv run draupnir       # Regenerate JSON Schemas
cd ../nornir && nornir_deploy --build gates    # Recompile gate modules
```

## Licence

Copyright (c) 2025–2026 John Oker-Blom

This program is free software: you can redistribute it and/or modify it under the terms of the GNU Affero General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

See [LICENSE](LICENSE) for the full licence text.
