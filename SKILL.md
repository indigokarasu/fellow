---
name: ocas-fellow
description: 'Fellow: empirical experimentation engine. Invoked by Mentor to evaluate,
  compare, and promote improvements to OCAS skills, prompts, heuristics, and workflows
  using benchmark-driven experiments. Returns best variant result with lineage. Not
  user-invocable -- called only by Mentor. Trigger phrases: ''update fellow''.

'
license: MIT
metadata:
  author: Indigo Karasu
  version: 2.6.5
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

Fellow is invoked only by Mentor. See the full invocation payload schema in `references/schemas.md` → **Invocation contract**.

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

At completion Fellow emits a result payload. See the full schema in `references/schemas.md` → **Cycle output**.

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

## Recovery Behavior

This skill implements the recovery contract from `spec-ocas-recovery.md`.

- **Evidence**: Every experiment run writes evidence to `{agent_root}/commons/data/ocas-fellow/evidence.jsonl`, including no-op runs with mandatory `not_activity_reason`.
- **Gap detection**: On every wake, checks evidence log for most recent run. If gap exceeds 24h for update cron, logs `gap_detected`.
- **Degraded mode**: When Mentor or experiment harness unavailable, logs `degraded: <dependency>` and queues work for retry.
- **Log compaction**: Evidence logs older than 30 days (no-op) or 90 days (error/gap) compacted. Escalation records never auto-deleted. Last 7 days retained.

## Storage layout

See `references/schemas.md` → **Storage layout** and **Default config.json**.

## OKRs

Universal OKRs from spec-ocas-journal.md apply to all runs. See `references/schemas.md` → **OKRs** for the full `skill_okrs` definition.

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

`fellow.update` pulls the latest package from the `source:` URL in this file's frontmatter. Runs silently — no output unless the version changed or an error occurred. See `references/schemas.md` → **Self-update procedure** for the full step-by-step.

## Visibility

public

## Gotchas

- **Fellow is not user-invocable** — If triggered directly by a user prompt, Fellow responds with a redirect to Mentor. Only Mentor can invoke `fellow.experiment.run` with a valid ExperimentRequest payload.
- **Baseline failure aborts the entire cycle** — If the champion baseline evaluation fails (benchmark error, harness crash), the whole experiment cycle is aborted. No variants are tested until the baseline issue is resolved.
- **Entity observations are never promoted to Chronicle** — Fellow's journals may include entity observations for lineage tracking, but Elephas does not extract Chronicle candidates from Fellow journals. Experiment artifacts stay internal.
- **Promotion threshold is strict** — A variant must beat the baseline by at least `improvement_threshold` (default 0.03) to be promotable. Variants failing guardrails are rejected even if they beat the baseline.
- **Mentor provides direction, Fellow provides optimization** — Fellow never decides what to improve; it only evaluates what Mentor sends. If Mentor's experiment program is flawed, Fellow will faithfully optimize the wrong thing.

## Support file map

| File | When to read |
|---|---|
| `references/schemas.md` | Before creating experiments, variants, or cycle outputs |
| `references/journal.md` | Before fellow.journal; at end of every run |

## Update command

This skill self-updates every 24 hours via `fellow.update`, which pulls the latest version from GitHub and restarts the skill's background tasks if applicable.
