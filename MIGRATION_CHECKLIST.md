# Verdandi — Migration Checklist

Maps every toolcompose YAML file to its Verdandi equivalent.
Organized smallest → largest (build order). Check off as migrated.

**Source:** `standards/toolcompose/`
**Target:** `tools/verdandi/`

---

## Key: What Happens to Current Primitives

Most current primitives (68 files) are just `type: string` with a
requirement. In Verdandi, the type lives in atom/ and the requirement
lives in the container (simplegroup/group). A current primitive like
`agent_description` (string, mandatory) becomes a reference to
`atom/string_plain` with `required: mandatory` at the simplegroup level.

Only primitives that carry UNIQUE constraints become atoms, enums, or
elements. The rest collapse into field references.

---

## pattern/ (NEW — extracted from current primitives)

| # | Verdandi file | Extracted from | regex |
|---|--------------|----------------|-------|
| | [x] relative | path patterns | `"^\."` |
| | [x] absolute | path patterns | `"^/"` |
| | [x] traversal | path_forbidden_traversal | `"\.\."` |
| | [x] env | path_forbidden_env | `"(^\\|/)\\.env($\\|/)"` |
| | [x] home | path_forbidden_home | `"~/"` + `"/Users/johnny/"` |
| | [x] root | path_forbidden_root | `"^/$"` |
| | [x] claude | path_forbidden_claude | `"(^\\|/)\\.claude($\\|/)"` |
| | [x] kebab | agent_name | `"^[a-z0-9]+(-[a-z0-9]+)*$"` |
| | [x] snake | param_name | `"^[a-z][a-z0-9_]*$"` |
| | [x] semver | semantic_version | `"^\\d+\\.\\d+\\.\\d+$"` |
| | [x] timestamp_zulu | timestamp_iso_zulu | `"^\\d{4}-\\d{2}-...Z$"` |

**Current files consumed:** path_forbidden_traversal, path_forbidden_env,
path_forbidden_home, path_forbidden_root, path_forbidden_claude,
PATH_BLOCKLIST (composed/)

---

## atom/ (from constrained primitives)

| # | Verdandi file | Source | Constraint |
|---|--------------|--------|------------|
| | [x] string_plain | (new) | type: string, excludes: [pattern/relative, pattern/absolute] |
| | [x] path_relative | path_allowed_* | type: string, pattern: pattern/relative, excludes: [pattern/traversal, pattern/env, pattern/home, pattern/root, pattern/claude] |
| | [x] path_absolute | path_allowed_* | type: string, pattern: pattern/absolute, excludes: [pattern/traversal, pattern/env, pattern/home, pattern/root, pattern/claude] |
| | [x] kebab_string | agent_name | type: string, pattern: pattern/kebab |
| | [x] snake_string | param_name | type: string, pattern: pattern/snake |
| | [x] semver_string | semantic_version | type: string, pattern: pattern/semver |
| | [x] timestamp_zulu | timestamp_iso_zulu | type: string, pattern: pattern/timestamp_zulu |
| | [x] integer | (base type) | type: integer |
| | [x] boolean | (base type) | type: boolean |

**Current files consumed:** agent_name (pattern), param_name (pattern),
path_allowed_absolute, path_allowed_any, semantic_version,
timestamp_iso_zulu

**Collapsed into atom/string_plain** (34 current primitives — no unique
constraints, just string + requirement):
agent_description, context_label, dispatch_input_description,
example_display_headings(!), example_group_name, example_heading,
example_text, failure_definition, failure_evidence(!),
guardrails_anti_pattern, guardrails_constraint, instruction_text,
notify_message, notify_trigger, processing_output_description,
processing_output_file_name_instruction,
processing_output_file_name_template, return_metrics, return_output,
return_status, role_description, role_identity, role_responsibility,
success_definition, success_evidence(!), tempdir_name

(!) = currently typed as array or boolean — verify before collapsing

---

## enum/ (from enum primitives)

| # | Verdandi file | Source primitive | Values |
|---|--------------|-----------------|--------|
| | [x] dispatch_mode | dispatch_mode | [batch, full] |
| | [x] dispatch_input_delivery | dispatch_input_delivery | [tempfile, inline, file, directory] |
| | [x] dispatch_background_mode | dispatch_background_mode | [allowed, required, forbidden] |
| | [x] input_format | file_format | [jsonl, json, text] |
| | [x] output_format_kind | output_format_kind | [jsonl, json, markdown, text] |
| | [x] capabilities | capabilities_all | [read, search, write, edit, append, inspect] |
| | [x] param_type | param_type | [path, string, integer, boolean] |
| | [x] return_mode | return_mode | [status, status-metrics, metrics-output, output] |
| | [x] instruction_mode | instruction_mode | [deterministic, probabilistic] |
| | [x] processing_capability | processing_capability_model | [fast-cheap, standard, analytical, complex-reasoning, max-capability, inherit] |
| | [x] output_name_known | processing_output_file_name_known | [known, partially, unknown] |
| | [x] enforcement_tool_format | enforcement_output_tool_format | [jsonl, json] |
| | [x] write_frequency | enforcement_output_tool_write_frequency + processing_output_write_frequency | [record, batch] |
| | [x] notify_delivery | notify_delivery | [say, notification, log, terminal] |

**Note:** file_format and dispatch_input_format both resolve to [jsonl, json, text].
Unified as `enum/input_format`. enforcement_output_tool_write_frequency and
processing_output_write_frequency both [record, batch] — unified as `enum/write_frequency`.

---

## element/ (from composed/ type adoptions + format)

| # | Verdandi file | Source | inherits | format |
|---|--------------|--------|----------|--------|
| | [x] path_exists_relative | (new) | atom/path_relative | exists |
| | [x] path_exists_absolute | path_exists_absolute | atom/path_absolute | exists |
| | [x] path_exists_schema | path_exists_absolute_schema, path_exists_any_schema | atom/path_absolute | exists_schema |
| | [x] agent_name | agent_name | atom/kebab_string | (mandatory) |
| | [x] agent_description | agent_description | atom/string_plain | (mandatory) |

**Note:** agent_name and agent_description are mandatory elements — they
carry the `required: mandatory` compile-time lock. Whether other fields
need element wrappers TBD.

---

## union/ (from composed/ oneOf + primitives)

| # | Verdandi file | Source | Members |
|---|--------------|--------|---------|
| | [x] path_any | path_allowed_any | [atom/path_relative, atom/path_absolute] |
| | [x] path_exists_any | path_exists_any | [element/path_exists_relative, element/path_exists_absolute] |

---

## tuple/ (from composed/)

| # | Verdandi file | Source | Items |
|---|--------------|--------|-------|
| | [x] dispatch_batch_size | dispatch_batch_size | [atom/integer, atom/integer], constraint: ascending_strict |

---

## simplearray/ (from composed/ array wrappers + array primitives)

| # | Verdandi file | Source composed/ | items |
|---|--------------|-----------------|-------|
| | [x] enforcement_explicit_read | enforcement_explicit_read | element/path_exists_absolute |
| | [x] enforcement_explicit_write | enforcement_explicit_write | element/path_exists_absolute |
| | [x] enforcement_explicit_append | enforcement_explicit_append | element/path_exists_absolute |
| | [x] enforcement_explicit_edit | enforcement_explicit_edit | element/path_exists_absolute |
| | [x] enforcement_explicit_search | enforcement_explicit_search | element/path_exists_absolute |
| | [x] enforcement_explicit_inspect | enforcement_explicit_inspect | element/path_exists_absolute |
| | [x] enforcement_implicit_read | enforcement_implicit_read | element/path_exists_absolute |
| | [x] enforcement_implicit_write | enforcement_implicit_write | element/path_exists_absolute |
| | [x] enforcement_implicit_append | enforcement_implicit_append | element/path_exists_absolute |
| | [x] enforcement_implicit_edit | enforcement_implicit_edit | element/path_exists_absolute |
| | [x] enforcement_implicit_search | enforcement_implicit_search | element/path_exists_absolute |
| | [x] enforcement_implicit_inspect | enforcement_implicit_inspect | element/path_exists_absolute |
| | [x] guardrails_constraints_raw | guardrails_constraints | atom/string_plain |
| | [x] guardrails_anti_patterns_raw | guardrails_anti_patterns | atom/string_plain |
| | [x] security_paths_allowed_read_raw | security_paths_allowed_read | element/path_exists_absolute |
| | [x] security_paths_allowed_write_raw | security_paths_allowed_write | element/path_exists_absolute |
| | [x] security_capabilities_requested | security_capabilities_requested | enum/capabilities |
| | [x] security_capabilities_prohibited | security_capabilities_prohibited | enum/capabilities |
| | [x] role_expertise | role_expertise (primitive!) | atom/string_plain |
| | [x] success_evidence | success_evidence (primitive!) | atom/string_plain |
| | [x] failure_evidence | failure_evidence (primitive!) | atom/string_plain |

**Resolved variants** (same structure, different item type):

| # | Verdandi file | Source composed/ |
|---|--------------|-----------------|
| | [x] context_path | context_path | union/path_any |
| | [x] context_path_resolved | context_path_resolved | atom/path_absolute |
| | [x] security_paths_allowed_read_resolved | security_paths_allowed_read_resolved | atom/path_absolute |
| | [x] security_paths_allowed_write_resolved | security_paths_allowed_write_resolved | atom/path_absolute |
| | [x] security_io_input_file | security_io_input_file | union/path_any |
| | [x] security_io_input_file_resolved | security_io_input_file_resolved | atom/path_absolute |
| | [x] security_io_input_directory | security_io_input_directory | union/path_any |
| | [x] security_io_input_directory_resolved | security_io_input_directory_resolved | atom/path_absolute |
| | [x] security_io_output_file | security_io_output_file | union/path_any |
| | [x] security_io_output_file_resolved | security_io_output_file_resolved | atom/path_absolute |
| | [x] security_io_output_directory | security_io_output_directory | union/path_any |
| | [x] security_io_output_directory_resolved | security_io_output_directory_resolved | atom/path_absolute |

**Note:** Many composed/ files that looked like "type adoption" are actually
single-value fields with conditional requirements. In Verdandi, conditionals
live at the simplegroup/group level. These composed/ files don't need
Verdandi equivalents — they collapse into field references with conditionals.

Collapsed: security_input_schema, security_output_schema,
security_paths_workspace, return_schema, security_io_input_tempdir,
security_io_scratch_tempdir (+resolved variants of each)

---

## simplegroup/ (from groups/ — flat objects)

| # | Verdandi file | Source group/ | Fields |
|---|--------------|---------------|--------|
| | [x] parameter_item | parameter_item | param_name, param_type, param_required, param_description |
| | [x] instruction_step | instruction_step | instruction_mode, instruction_text |
| | [x] success_item | success_item | success_definition, success_evidence |
| | [x] failure_item | failure_item | failure_definition, failure_evidence |
| | [x] context_item_raw | context_item | context_path, context_label |
| | [x] context_item_resolved | context_item_resolved | context_path (absolute), context_label |
| | [x] example_entry | example_entry | example_heading, example_text |
| | [x] notify_item | notify_item | notify_trigger, notify_delivery, notify_message |
| | [x] include_instruction | include_instruction (composed/) | include path |
| | [x] include_instruction_resolved | include_instruction_resolved | resolved path |
| | [x] include_success_criteria | include_success_criteria (composed/) | include path |
| | [x] include_failure_criteria | include_failure_criteria (composed/) | include path |
| | [x] include_guardrails_constraints | include_guardrails_constraints (composed/) | include path |
| | [x] include_guardrails_anti_patterns | include_guardrails_anti_patterns (composed/) | include path |
| | [x] include_example | include_example (composed/) | include path |
| | [x] include_example_resolved | include_example_resolved (composed/) | resolved path |
| | [x] include_example_group | include_example_group (composed/) | include path |
| | [x] include_example_group_resolved | include_example_group_resolved (composed/) | resolved path |

---

## group/ (from groups/ + supergroups/ — objects with containers)

| # | Verdandi file | Source | Contains |
|---|--------------|--------|----------|
| | [x] metadata | groups/metadata | element/agent_name, element/agent_description, atom/semver_string |
| | [x] role | groups/role | fields + simplearray/role_expertise |
| | [x] enforcement_explicit_grants | groups/enforcement_explicit_grants | 6 simplearrays |
| | [x] enforcement_implicit_grants | groups/enforcement_implicit_grants | 6 simplearrays |
| | [x] enforcement_output_tool | groups/enforcement_output_tool | fields + conditionals |
| | [x] guardrails_raw | groups/guardrails | simplearray/constraints + simplearray/anti_patterns |
| | [x] guardrails_resolved | groups/guardrails_resolved | resolved variant |
| | [x] guardrails_reduced | groups/guardrails_reduced | reduced variant |
| | [x] security_io_raw | groups/security_io | fields + simplearrays |
| | [x] security_io_resolved | groups/security_io_resolved | resolved variant |
| | [x] security_paths_raw | groups/security_paths | fields + simplearrays |
| | [x] security_paths_resolved | groups/security_paths_resolved | resolved variant |
| | [x] security_schema_raw | groups/security_schema | fields |
| | [x] security_schema_resolved | groups/security_schema_resolved | resolved variant |
| | [x] security_capabilities | groups/security_capabilities | simplearrays |
| | [x] processing | groups/processing | fields + enums |
| | [x] return | groups/return | fields + enums |
| | [x] example_group_raw | supergroups/example_group | fields + array/example_entries |
| | [x] example_group_resolved | supergroups/example_group_resolved | resolved variant |
| | [x] example_group_reduced | supergroups/example_group_reduced | reduced variant |
| | [x] dispatch | supergroups/dispatch | fields + array/dispatch_parameters |
| | [x] communication | supergroups/communication | fields |
| | [x] context_raw | supergroups/context | arrays (context_required, context_available) |
| | [x] context_resolved | supergroups/context_resolved | resolved variant |

---

## array/ (from arrays/ — items are simplegroups)

| # | Verdandi file | Source array/ | Items |
|---|--------------|---------------|-------|
| | [x] dispatch_parameters | dispatch_parameters | simplegroup/parameter_item |
| | [x] execution_instructions_raw | execution_instructions | oneOf: [simplegroup/instruction_step, simplegroup/include_instruction] |
| | [x] execution_instructions_resolved | execution_instructions_resolved | resolved variant |
| | [x] execution_instructions_reduced | execution_instructions_reduced | reduced variant |
| | [x] success_criteria_raw | success_criteria | oneOf: [simplegroup/success_item, simplegroup/include_success_criteria] |
| | [x] success_criteria_resolved | success_criteria_resolved | resolved variant |
| | [x] success_criteria_reduced | success_criteria_reduced | reduced variant |
| | [x] failure_criteria_raw | failure_criteria | oneOf: [simplegroup/failure_item, simplegroup/include_failure_criteria] |
| | [x] failure_criteria_resolved | failure_criteria_resolved | resolved variant |
| | [x] failure_criteria_reduced | failure_criteria_reduced | reduced variant |
| | [x] context_required_raw | context_required | simplegroup/context_item |
| | [x] context_required_resolved | context_required_resolved | resolved variant |
| | [x] context_available_raw | context_available | simplegroup/context_item |
| | [x] context_available_resolved | context_available_resolved | resolved variant |
| | [x] example_entries_raw | example_entries | oneOf: [simplegroup/example_entry, simplegroup/include_example] |
| | [x] example_entries_resolved | example_entries_resolved | resolved variant |
| | [x] example_entries_reduced | example_entries_reduced | reduced variant |
| | [x] notify | notify | simplegroup/notify_item |

---

## superarray/ (from superarrays/)

| # | Verdandi file | Source | Items |
|---|--------------|--------|-------|
| | [x] execution_examples_raw | execution_examples | oneOf: [group/example_group, simplegroup/include_example_group] |
| | [x] execution_examples_resolved | execution_examples_resolved | resolved variant |
| | [x] execution_examples_reduced | execution_examples_reduced | reduced variant |

---

## section/ (from sections/ — complex containers only)

| # | Verdandi file | Source section/ | Components |
|---|--------------|-----------------|------------|
| | [x] task | task | group/role, group/communication, group/return, group/processing |
| | [x] execution_raw | execution | array/execution_instructions, superarray/execution_examples |
| | [x] execution_resolved | execution_resolved | resolved variant |
| | [x] execution_reduced | execution_reduced | reduced variant |
| | [x] criteria_raw | criteria | group/guardrails, array/success_criteria, array/failure_criteria |
| | [x] criteria_resolved | criteria_resolved | resolved variant |
| | [x] criteria_reduced | criteria_reduced | reduced variant |
| | [x] security_raw | security | group/security_io, group/security_paths, group/security_schema, group/security_capabilities |
| | [x] security_resolved | security_resolved | resolved variant |
| | [x] enforcement | enforcement | group/enforcement_explicit_grants, group/enforcement_implicit_grants, group/enforcement_output_tool |
| | [x] indirect | indirect | group/metadata, group/dispatch, group/context, section/enforcement |

**Cross-stage sections** (from sections/ — combinatorial variants):

| # | Verdandi file | Source |
|---|--------------|--------|
| | [x] examples_resolved_instructions_reduced | examples_resolved_instructions_reduced |
| | [x] examples_reduced_instructions_resolved | examples_reduced_instructions_resolved |
| | [x] success_failure_resolved_guardrails_reduced | success_failure_resolved_guardrails_reduced |
| | [x] success_failure_reduced_guardrails_resolved | success_failure_reduced_guardrails_resolved |

---

## shallowsection/ (NEW — for render schemas)

These don't exist in toolcompose. Built fresh from UNIVERSAL_RENDER_SCHEMA_DESIGN.md.

| # | Verdandi file | Purpose |
|---|--------------|---------|
| | [x] identity | agent identity fields + capability_tier |
| | [x] security_boundary | workspace_path + capability grants |
| | [x] implicit_grants | derived grants for hooks |
| | [x] input | input format + parameters + context arrays |
| | [x] output | output format + file location + naming |
| | [x] writing_output | output tool config |
| | [x] constraints | guardrail rules |
| | [x] anti_patterns | failure patterns |
| | [x] return_format | return mode + criteria |
| | [x] dispatcher | dispatch config for launcher |

---

## schema/ (from schema/)

| # | Verdandi file | Source schema/ | Type |
|---|--------------|----------------|------|
| | [x] agent-raw-definition | agent-raw-definition | raw input |
| | [x] agent-universal-format | agent-universal-format | resolved output |
| | [x] agent-includes-resolved | agent-includes-resolved | intermediate |
| | [x] agent-paths-resolved | agent-paths-resolved | intermediate |
| | [x] agent-permissions-resolved | agent-permissions-resolved | intermediate |
| | [x] agent-execution-merged | agent-execution-merged | intermediate |
| | [x] agent-criteria-merged | agent-criteria-merged | intermediate |
| | [x] agent-instructions-reduced | agent-instructions-reduced | intermediate |
| | [x] agent-examples-reduced | agent-examples-reduced | intermediate |
| | [x] agent-guardrails-reduced | agent-guardrails-reduced | intermediate |
| | [x] agent-sf-reduced | agent-sf-reduced | intermediate |
| | [x] agent-universal-render | (NEW) | render format |

---

## Migration Summary

| Verdandi level | New files | From toolcompose | Notes |
|---------------|-----------|-----------------|-------|
| pattern/ | ~11 | 5 primitives + PATH_BLOCKLIST | Extracted regex patterns |
| atom/ | ~9 | ~6 primitives (rest collapse) | Smallest useful constrained types |
| enum/ | ~14 | 16 primitives (some unified) | Extracted enum values |
| element/ | ~5 | ~5 composed + primitives | atom + format/mandatory |
| union/ | ~2 | 2 primitives/composed | oneOf compositions |
| tuple/ | 1 | 1 composed | Fixed-length sequences |
| simplearray/ | ~33 | ~21 composed + 3 primitives | Arrays of fields |
| simplegroup/ | ~19 | ~10 groups + ~9 composed | Flat objects |
| group/ | ~23 | ~15 groups + 7 supergroups | Objects with containers |
| array/ | ~18 | 18 arrays | Arrays of simplegroups |
| superarray/ | 3 | 3 superarrays | Arrays of groups |
| section/ | ~15 | 15 sections | Complex containers |
| shallowsection/ | ~10 | (NEW) | Render format sections |
| schema/ | ~12 | 11 schemas + 1 new | Root files |

**Total Verdandi files:** ~175
**Current toolcompose files:** ~204 (68 prim + 57 composed + 25 group + 18 array + 7 supergroup + 3 superarray + 15 section + 11 schema)

~34 current primitives collapse into field references (no individual file needed).
~20 composed files collapse into field references with conditionals.
composed/ disappears entirely — split across element/, union/, tuple/, simplearray/, simplegroup/.

---

## Build Order

1. **pattern/** — no dependencies
2. **atom/** — depends on pattern/
3. **enum/** — no dependencies (parallel with atom/)
4. **element/** — depends on atom/
5. **union/** — depends on atom/, enum/, element/
6. **tuple/** — depends on atom/, enum/, element/, union/
7. **simplearray/** — depends on all fields
8. **simplegroup/** — depends on all fields + simple containers
9. **group/** — depends on simplegroup/, simplearray/, tuple/
10. **array/** — depends on simplegroup/
11. **superarray/** — depends on group/
12. **section/** — depends on group/, array/, superarray/
13. **shallowsection/** — depends on all levels
14. **schema/** — depends on section/, shallowsection/
