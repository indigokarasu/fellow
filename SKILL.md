---
name: ocas-fellow
source: https://github.com/indigokarasu/fellow
install: openclaw skill install https://github.com/indigokarasu/fellow
description: Use only when invoked by Mentor to evaluate, compare, and promote improvements to OCAS skills, prompts, heuristics, and workflows via benchmark-driven experiments. Returns best variant result with lineage. Not user-invocable -- called only by Mentor. Trigger phrases: 'update fellow'. Do not use for skill building (use Forge), behavioral pattern analysis (use Corvus), or user-initiated evaluation requests (use Mentor).
metadata: {"openclaw":{"emoji":"🧪"}}
---

# Fellow

Fellow is the system's empirical optimization engine, invoked exclusively by Mentor to determine which implementation of a skill, prompt, heuristic, or workflow actually performs best — not which one looks best on paper. It runs controlled experiments with a fixed benchmark and compute budget, establishes a fresh baseline before testing any variant, and returns the winning result with full mutation lineage so every promotion is traceable and reversible.


## When to use

Fellow is not user-invocable. It is called only by Mentor when:
- A skill's OKR performance has regressed
- A variant proposal needs empirical evaluation
- A prompt, heuristic, or workflow needs optimization
- Mentor needs to compare champion vs challenger implementations


## When not to use

- User-initiated requests — Fellow is Mentor-only
- Skill building or design — use Forge
- Pattern analysis — use Corvus
- Web research — use Sift


## Responsibility boundary

Fellow owns empirical experimentation: baseline establishment, variant generation, benchmark execution, metric extraction, and promotion decisions.

Fellow does not own: deciding what to improve (Mentor), building skill packages (Forge), knowledge persistence (Elephas), behavioral refinement (Praxis).

Mentor provides direction. Fellow provides empirical optimization. Elephas stores lineage and artifacts.


## Invocation guard

Fellow is not user-invocable. If triggered directly by a user prompt, respond: "Fellow is an internal engine invoked only by Mentor for benchmark experiments. For skill evaluation, use Mentor."


## Inter-skill interfaces

Fellow receives ExperimentRequest files from Mentor at: `~/openclaw/data/ocas-fellow/intake/{experiment_id}.json`

Mentor writes the request file, then invokes `fellow.experiment.run`. Fellow reads the file, executes the experiment cycle, and writes a CycleResult to Mentor's intake at: `~/openclaw/data/ocas-mentor/intake/{cycle_id}.json`

After processing, move the request file to `intake/processed/`.

See `spec-ocas-interfaces.md` for schemas and handoff contracts.


## Invocation contract

Fellow is invoked only by Mentor. Invocation payload structure:

```yaml
target: <component_identifier>
program_id: <experiment_program>
objective: <primary_metric>
benchmark: <benchmark_identifier>
budget:
  type: wall_clock | task_count | token_budget | simulation_window
  value: <number>
constraints:
  max_variants: <number>
  max_cycles: <number>
  mutation_surface: [<path_or_field>]
  protected_surface: [<path_or_field>]
promotion:
  threshold: <minimum_improvement>
  automatic: <boolean>
  rollback_on_regression: true
runner:
  type: command | function | workflow
  entrypoint: <runner_entrypoint>
  timeout_seconds: <number>
metric_extractor:
  type: json | regex | function | structured_output
  source: <artifact_source>
  selector: <metric_path>
```


## Experiment lifecycle

1. Establish baseline — run champion against benchmark with identical conditions
2. Generate candidate variants — controlled mutations within allowed surface only
3. Validate variants — syntax, interface, mutation surface, benchmark compatibility
4. Run evaluations — execute each variant against benchmark
5. Extract metrics — primary metric + guardrail metrics
6. Compare against baseline — variant_score >= baseline_score + improvement_threshold
7. Generate follow-up variants — if budget allows and plateau not reached
8. Terminate — on plateau, budget exhaustion, or limit reached

Default limits: max_variants_per_cycle: 20, max_cycles_per_target: 5, max_parallel_runs: 1, plateau_window: 5.


## Baseline protocol

Before testing any variant, a fresh baseline evaluation runs under identical conditions: same benchmark, runtime budget, seed policy, and evaluation harness. Baseline artifacts are immutable during the cycle. Baseline failure aborts the cycle.


## Mutation engine

Supported methods: parameter_sweep, patch_diff, template_substitution, controlled_rewrite, heuristic_mutation.

Constraints: modify only allowed mutation surfaces, never alter protected surfaces, maintain interface compatibility. Every variant records: variant_id, parent_variant, mutation_method, change_summary.


## Promotion rule

A variant is promotable only when: variant_score >= baseline_score + improvement_threshold (default: 0.03). Variants failing guardrails are rejected. Mentor may automatically promote or require manual approval.


## Cycle output

At completion Fellow emits:

```yaml
cycle_id:
target:
baseline_score:
best_variant_id:
best_variant_score:
improvement:
decision: promote | no_change | abort
artifacts_ref:
rollback_ref:
```


## Run completion

After every experiment cycle:

1. Check `~/openclaw/data/ocas-fellow/intake/` for ExperimentRequest files from Mentor; process and move to `intake/processed/`
2. Persist experiment records, variant results, and cycle output to local files
3. Write CycleResult to `~/openclaw/data/ocas-mentor/intake/{cycle_id}.json`
4. Log material decisions to `decisions.jsonl`
5. Write journal via `fellow.journal`

## Failure handling

- Variant failure: mark failed, preserve logs, continue cycle
- Baseline failure: abort cycle, notify Mentor
- Benchmark failure: invalidate cycle, abort experiment


## Commands

- `fellow.experiment.run` — execute an experiment cycle from Mentor invocation payload
- `fellow.experiment.status` — current experiment state if in progress
- `fellow.journal` — write journal for the current run; called at end of every run
- `fellow.update` — pull latest from GitHub source; preserves journals and data


## Storage layout

```
~/openclaw/data/ocas-fellow/
  config.json
  experiments.jsonl
  decisions.jsonl
  intake/
    {experiment_id}.json
    processed/
  runs/
    {cycle_id}/
      baseline/
      variant-001/
      variant-002/

~/openclaw/journals/ocas-fellow/
  YYYY-MM-DD/
    {run_id}.json
```


Default config.json:
```json
{
  "skill_id": "ocas-fellow",
  "skill_version": "2.3.0",
  "config_version": "1",
  "created_at": "",
  "updated_at": "",
  "defaults": {
    "improvement_threshold": 0.03,
    "max_variants_per_cycle": 20,
    "max_cycles_per_target": 5,
    "max_parallel_runs": 1,
    "plateau_window": 5
  },
  "retention": {
    "days": 90,
    "max_records": 10000
  }
}
```


## OKRs

Universal OKRs from spec-ocas-journal.md apply to all runs.

```yaml
skill_okrs:
  - name: experiment_completion_rate
    metric: fraction of experiment cycles completing without abort
    direction: maximize
    target: 0.90
    evaluation_window: 30_runs
  - name: promotion_rate
    metric: fraction of completed experiments producing a promotable variant
    direction: maximize
    target: 0.30
    evaluation_window: 30_runs
  - name: baseline_stability
    metric: fraction of baselines completing successfully
    direction: maximize
    target: 0.99
    evaluation_window: 30_runs
```


## Optional skill cooperation

- Mentor — sole invoker; provides experiment programs and approves promotions
- Elephas — stores experiment lineage and artifacts via signal intake


## Journal outputs

Action Journal — every experiment cycle execution.


## Initialization

On first invocation by Mentor, run `fellow.init`:

1. Create `~/openclaw/data/ocas-fellow/` and subdirectories (`intake/`, `intake/processed/`, `runs/`)
2. Write default `config.json` with ConfigBase fields if absent
3. Create empty JSONL files: `experiments.jsonl`, `decisions.jsonl`
4. Create `~/openclaw/journals/ocas-fellow/`
5. Ensure `~/openclaw/data/ocas-mentor/intake/` exists (create if missing)
6. Register cron job `fellow:update` if not already present (check `openclaw cron list` first)
7. Log initialization as a DecisionRecord in `decisions.jsonl`

## Background tasks

| Job name | Mechanism | Schedule | Command |
|---|---|---|---|
| `fellow:update` | cron | `0 0 * * *` (midnight daily) | `fellow.update` |

```
openclaw cron add --name fellow:update --schedule "0 0 * * *" --command "fellow.update" --sessionTarget isolated --lightContext true --timezone America/Los_Angeles
```


## Self-update

`fellow.update` pulls the latest package from the `source:` URL in this file's frontmatter. Runs silently — no output unless the version changed or an error occurred.

1. Read `source:` from frontmatter → extract `{owner}/{repo}` from URL
2. Read local version from `skill.json`
3. Fetch remote version: `gh api "repos/{owner}/{repo}/contents/skill.json" --jq '.content' | base64 -d | python3 -c "import sys,json;print(json.load(sys.stdin)['version'])"`
4. If remote version equals local version → stop silently
5. Download and install:
   ```bash
   TMPDIR=$(mktemp -d)
   gh api "repos/{owner}/{repo}/tarball/main" > "$TMPDIR/archive.tar.gz"
   mkdir "$TMPDIR/extracted"
   tar xzf "$TMPDIR/archive.tar.gz" -C "$TMPDIR/extracted" --strip-components=1
   cp -R "$TMPDIR/extracted/"* ./
   rm -rf "$TMPDIR"
   ```
6. On failure → retry once. If second attempt fails, report the error and stop.
7. Output exactly: `I updated Fellow from version {old} to {new}`


## Visibility

public


## Support file map

| File | When to read |
|---|---|
| `references/schemas.md` | Before creating experiments, variants, or cycle outputs |
| `references/journal.md` | Before fellow.journal; at end of every run |
