---
name: ocas-fellow
description: >
  Fellow: empirical experimentation engine. Invoked by Mentor to evaluate,
  compare, and promote improvements to OCAS skills, prompts, heuristics, and
  workflows using benchmark-driven experiments. Returns best variant result
  with lineage. Not user-invocable -- called only by Mentor. Trigger phrases:
  'update fellow'.
metadata:
  author: Indigo Karasu
  email: mx.indigo.karasu@gmail.com
  version: "2.6.1"
  hermes:
    tags: [experimentation, benchmarks, evaluation]
    category: evolution
    cron:
      - name: "fellow:update"
        schedule: "0 0 * * *"
        command: "fellow.update"
  openclaw:
    skill_type: system
    visibility: public
    filesystem:
      read:
        - "{agent_root}/commons/data/ocas-fellow/"
        - "{agent_root}/commons/journals/ocas-fellow/"
      write:
        - "{agent_root}/commons/data/ocas-fellow/"
        - "{agent_root}/commons/journals/ocas-fellow/"
    self_update:
      source: "https://github.com/indigokarasu/fellow"
      mechanism: "version-checked tarball from GitHub via gh CLI"
      command: "fellow.update"
      requires_binaries: [gh, tar, python3]
    cron:
      - name: "fellow:update"
        schedule: "0 0 * * *"
        command: "fellow.update"
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


## Ontology types

Fellow observes entity types during experiment execution:
- **Concept/Idea** — hypotheses, experiment designs, potential improvements
- **Thing/DigitalArtifact** — experiment artifacts, baseline runs, variant packages, metric reports
- **Concept/Event** — experiment cycles, baseline establishment events, promotion decisions

Fellow does not emit Signals to Elephas for these observations. Journal entries may include entity observations for internal lineage tracking, but they are not promoted to Chronicle. Elephas consumes Fellow journals for reference only and does not extract Chronicle candidates from them.


## Invocation guard

Fellow is not user-invocable. If triggered directly by a user prompt, respond: "Fellow is an internal engine invoked only by Mentor for benchmark experiments. For skill evaluation, use Mentor."


## Inter-skill interfaces

**Mentor → Fellow (cooperative read):** Fellow reads ExperimentRequest files from `{agent_root}/commons/data/ocas-mentor/experiment-requests/`. Mentor writes the request then invokes `fellow.experiment.run`. Fellow tracks consumed `experiment_id` values in `requests_processed.jsonl`. Fellow does not write to Mentor's directories.

**Fellow → Mentor (cooperative read):** Fellow writes CycleResult files to `{agent_root}/commons/data/ocas-fellow/results/{cycle_id}.json`. Mentor reads from this directory. Fellow does not write to Mentor's directories.

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

1. Read ExperimentRequest files from `{agent_root}/commons/data/ocas-mentor/experiment-requests/`. Track consumed `experiment_id` values in `requests_processed.jsonl`.
2. Persist experiment records, variant results, and cycle output to local files
3. Write CycleResult to `{agent_root}/commons/data/ocas-fellow/results/{cycle_id}.json`. Mentor reads from this directory.
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
{agent_root}/commons/data/ocas-fellow/
  config.json
  experiments.jsonl
  decisions.jsonl
  requests_processed.jsonl
  results/
    {cycle_id}.json
  runs/
    {cycle_id}/
      baseline/
      variant-001/
      variant-002/

{agent_root}/commons/journals/ocas-fellow/
  YYYY-MM-DD/
    {run_id}.json
```


Default config.json:
```json
{
  "skill_id": "ocas-fellow",
  "skill_version": "2.4.0",
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
- Elephas — stores experiment lineage and artifacts via journal signal payloads; journal entity observations consumed during Chronicle ingestion


## Journal outputs

Action Journal — every experiment cycle execution.

When entities are encountered during runs, journals should include the following fields in `decision.payload`:

- `entities_observed` — entities noticed during the run (e.g., Concept/Idea for hypotheses and experiment results, Thing/DigitalArtifact for experiment artifacts, Concept/Event for experiment milestones)
- `relationships_observed` — relationships between observed entities
- `preferences_observed` — any user preferences or behavioral preferences surfaced

Each entity observation must include a `user_relevance` field:
- `user` — the entity is directly related to the user's world (e.g., experiments about user preferences or user-facing features)
- `agent_only` — encountered incidentally as part of system-internal experimentation (most Fellow entities fall here)
- `unknown` — relevance to the user is unclear


## Initialization

On first invocation by Mentor, run `fellow.init`:

1. Create `{agent_root}/commons/data/ocas-fellow/` and subdirectories (`results/`, `runs/`)
2. Write default `config.json` with ConfigBase fields if absent
3. Create empty JSONL files: `experiments.jsonl`, `decisions.jsonl`, `requests_processed.jsonl`
4. Create `{agent_root}/commons/journals/ocas-fellow/`
5. Register cron job `fellow:update` if not already present (check the platform scheduling registry first)
7. Log initialization as a DecisionRecord in `decisions.jsonl`

## Background tasks

| Job name | Mechanism | Schedule | Command |
|---|---|---|---|
| `fellow:update` | cron | `0 0 * * *` (midnight daily) | `fellow.update` |

```
# Task declared in SKILL.md frontmatter metadata.{platform}.cron
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

## Update command

This skill self-updates every 24 hours via:

```bash
openclaw fellow.update
```

This pulls the latest version from GitHub and restarts the skill's background tasks if applicable.
