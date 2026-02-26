# Extension Migration Crossings

Container files in `standards/toolcompose/` that reference type-tier files bearing `semantic` or `x-constraint` extensions.

**Format:** `source.field_name.target  [extension_type]`

**Filter:** source NOT IN type-tier; target has `semantic` or `x-constraint` block

**Source:** `standards/toolcompose/` (159 YAML files scanned)

---

## semantic (17 crossings)

```
groups/context_item.context_label.primitives/context_label  [semantic]
groups/context_item_resolved.context_label.primitives/context_label  [semantic]
groups/failure_item.failure_definition.primitives/failure_definition  [semantic]
groups/failure_item.failure_evidence.primitives/failure_evidence  [semantic]
groups/guardrails.guardrails_anti_patterns.composed/guardrails_anti_patterns  [semantic]
groups/guardrails.guardrails_constraints.composed/guardrails_constraints  [semantic]
groups/guardrails_reduced.guardrails_anti_patterns.composed/guardrails_anti_patterns  [semantic]
groups/guardrails_reduced.guardrails_constraints.composed/guardrails_constraints  [semantic]
groups/guardrails_resolved.guardrails_anti_patterns.composed/guardrails_anti_patterns  [semantic]
groups/guardrails_resolved.guardrails_constraints.composed/guardrails_constraints  [semantic]
groups/instruction_step.instruction_text.primitives/instruction_text  [semantic]
groups/metadata.agent_description.primitives/agent_description  [semantic]
groups/role.role_description.primitives/role_description  [semantic]
groups/role.role_identity.primitives/role_identity  [semantic]
groups/role.role_responsibility.primitives/role_responsibility  [semantic]
groups/success_item.success_definition.primitives/success_definition  [semantic]
groups/success_item.success_evidence.primitives/success_evidence  [semantic]
```

## x-constraint (9 crossings)

```
groups/security_io.security_io_input_tempdir.composed/security_io_input_tempdir  [x-constraint]
groups/security_io.security_io_output_directory.composed/security_io_output_directory  [x-constraint]
groups/security_io.security_io_output_file.composed/security_io_output_file  [x-constraint]
groups/security_io.security_io_scratch_tempdir.composed/security_io_scratch_tempdir  [x-constraint]
groups/security_io_resolved.security_io_input_tempdir.composed/security_io_input_tempdir  [x-constraint]
groups/security_io_resolved.security_io_scratch_tempdir.composed/security_io_scratch_tempdir  [x-constraint]
groups/security_schema.security_output_schema.composed/security_output_schema  [x-constraint]
groups/security_schema_resolved.security_output_schema.composed/security_output_schema_resolved  [x-constraint]
supergroups/dispatch.dispatch_batch_size.composed/dispatch_batch_size  [x-constraint]
```

---

## Summary

| Extension | Crossings | Distinct targets | Destination |
|-----------|-----------|------------------|-------------|
| `semantic` | 17 | 12 unique targets | `field/*.yaml` as `semantic:` block |
| `x-constraint` | 9 | 7 unique targets | `field/*.yaml` as `constraint:` block |
| **Total** | **26** | **19** | |
