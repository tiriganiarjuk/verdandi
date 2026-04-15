# Verdandi

Typed composition hierarchy for schema-driven agent definitions.

**Part of the agent definition pipeline:** Verdandi → [Draupnir](https://github.com/johnny111272/draupnir) → [Nornir](https://github.com/johnny111272/nornir) Gates → [Regin](https://github.com/johnny111272/regin) → Galdr

## What This Is

Verdandi is the type system for the agent definition pipeline. It consists of approximately 350 YAML files organised into a 15-level composition hierarchy across four tiers:

- **Type tier** — patterns, atoms, enums, elements, unions, tuples
- **Field membrane** — wraps types with per-field descriptions and extensions
- **Container tier** — arrays, groups, superarrays
- **Top level** — sections, schemas

References flow strictly downward — no same-level or upward references. The hierarchy defines the structural vocabulary for agent definitions: what fields are legal, what types they carry, and how they compose.

Verdandi is static data, not executable. It is the source of truth that Draupnir reads to generate JSON Schemas, which in turn become the mechanical enforcement layer for the entire pipeline.

## Key Concepts

**Field membrane.** Containers never reference raw type-tier files directly. The field level wraps types with per-field descriptions and extensions. Skipping it produces types without semantic meaning.

**Pipeline stage variants.** Types exist in variants: `_raw` (authored), `_resolved` (paths absolutised), `_reduced` (includes inlined). These describe stages of agent definition processing in Regin, not Verdandi processing — the YAML itself is static.

**Conditional keywords.** Three conditional keywords control field presence based on other field values: `only_when`, `always_when`, `not_when`.

## Usage

Verdandi is edited by hand. After editing YAML, regenerate schemas through the downstream pipeline:

```bash
cd ../draupnir && uv run draupnir --all       # Regenerate JSON Schemas
cd ../nornir && nornir_deploy --build gates    # Recompile gate modules
```

## Licence

Copyright (c) 2025–2026 John Oker-Blom

This program is free software: you can redistribute it and/or modify it under the terms of the GNU Affero General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

See [LICENSE](LICENSE) for the full licence text.
