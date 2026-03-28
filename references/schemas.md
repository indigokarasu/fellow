# Fellow Schemas

## ExperimentProgram
```json
{
  "program_id": "string",
  "target": "string — component identifier",
  "objective": "string — primary metric",
  "benchmark": "string — benchmark identifier",
  "budget": {
    "type": "string — wall_clock|task_count|token_budget|simulation_window",
    "value": "number"
  },
  "constraints": {
    "max_variants": "number",
    "max_cycles": "number",
    "max_parallel_runs": "number",
    "mutation_surface": ["string"],
    "protected_surface": ["string"]
  },
  "promotion": {
    "threshold": "number — minimum improvement (default 0.03)",
    "automatic": "boolean",
    "rollback_on_regression": "boolean"
  },
  "runner": {
    "type": "string — command|function|workflow",
    "entrypoint": "string",
    "timeout_seconds": "number"
  },
  "metric_extractor": {
    "type": "string — json|regex|function|structured_output",
    "source": "string",
    "selector": "string"
  }
}
```

## VariantRecord
```json
{
  "variant_id": "string",
  "cycle_id": "string",
  "parent_variant": "string|null",
  "mutation_method": "string — parameter_sweep|patch_diff|template_substitution|controlled_rewrite|heuristic_mutation",
  "change_summary": "string",
  "validation_status": "string — passed|failed",
  "primary_metric": "number|null",
  "guardrails": "object|null",
  "status": "string — pending|running|completed|failed|rejected"
}
```

## CycleOutput
```json
{
  "cycle_id": "string",
  "target": "string",
  "baseline_score": "number",
  "best_variant_id": "string|null",
  "best_variant_score": "number|null",
  "improvement": "number|null",
  "decision": "string — promote|no_change|abort",
  "artifacts_ref": "string|null",
  "rollback_ref": "string|null",
  "timestamp": "string — ISO 8601"
}
```

## DecisionRecord
Extends shared DecisionRecord from spec-ocas-shared-schemas.md. Fellow-specific types: baseline_established, variant_validated, variant_rejected, cycle_completed, promotion_recommended.
