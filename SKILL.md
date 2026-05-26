---
name: ocas-fellow
description: 'Fellow: empirical experimentation engine. Invoked by Mentor to evaluate,
  compare, and promote improvements to OCAS skills, prompts, heuristics, and workflows
  using benchmark-driven experiments. Returns best variant result with lineage. Not
  user-invocable -- called only by Mentor. Trigger phrases: ''update fellow''.

'
license: MIT
source: https://github.com/indigokarasu/fellow
includes:
  - references/**

metadata:
  author: Indigo Karasu
  version: 2.6.5
---

# Fellow

Fellow is the system's empirical optimization engine, invoked exclusively by Mentor to determine which implementation of a skill, prompt, heuristic, or workflow actually performs best — not which one looks best on paper. It runs controlled experiments with a fixed benchmark and compute budget, establishes a fresh baseline before testing any variant, and returns the winning result with full mutation lineage so every promotion is traceable and reversible.

## When to Use

Fellow is not user-invocable. It is called only by Mentor when:
- A skill's OKR performance has regressed
- A variant proposal needs empirical evaluation
- A prompt, heuristic, or workflow needs optimization
- Mentor needs to compare champion vs challenger implementations

## When NOT to Use

- User-initiated requests — Fellow is Mentor-only
- Skill building or design — use Forge
- Pattern analysis — use Corvus
- Web research — use Sift

## Responsibility boundary

Fellow owns empirical experimentation: baseline establishment, variant generation, benchmark execution, metric extraction, and promotion decisions.

Fellow does not own: deciding what to improve (Mentor), building skill packages (Forge), knowledge persistence (Elephas), behavioral refinement (Praxis).

Mentor provides direction. Fellow provides empirical optimization. Elephas stores lineage and artifacts.

## Ontology Types

Fellow observes entity types during experiment execution (Concept/Idea, Thing/DigitalArtifact, Concept/Event). Fellow does not emit Signals to Elephas for these observations. See `references/schemas.md` for full details.

## Invocation Guard

Fellow is not user-invocable. If triggered directly by a user prompt, respond: "Fellow is an internal engine invoked only by Mentor for benchmark experiments. For skill evaluation, use Mentor."

## Inter-Skill Interfaces

**Mentor → Fellow:** Fellow reads ExperimentRequest files from Mentor's experiment-requests directory. **Fellow → Mentor:** Fellow writes CycleResult files to `{agent_root}/commons/data/ocas-fellow/results/`. See `spec-ocas-interfaces.md` for schemas.

## Experiment Lifecycle

See `references/schemas.md` for the full experiment lifecycle (8 steps), baseline protocol, mutation engine, promotion rule, cycle output schema, run completion procedure, and failure handling.

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

## Journal Outputs

Action Journal — every experiment cycle execution. Entity observations (Concept/Idea, Thing/DigitalArtifact, Concept/Event) may be included for lineage tracking. Each entity observation includes a `user_relevance` field (`user`, `agent_only`, `unknown`). See `references/journal.md` for full schema.

## Initialization

On first invocation by Mentor, run `fellow.init`: create data directories, write default config, create empty JSONL files, register cron. See `references/schemas.md` for full initialization steps.

## Background Tasks

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

See `references/gotchas.md` for all operational pitfalls including invocation guard, baseline failure, entity observation handling, promotion threshold, and Mentor/Fellow responsibility split.

## Support File Map

| File | When to read |
|---|---|
| `references/schemas.md` | Before creating experiments, variants, or cycle outputs |
| `references/journal.md` | Before fellow.journal; at end of every run |
| `references/gotchas.md` | Before any experiment run. Operational pitfalls for invocation, baseline, entities, promotion, and Mentor/Fellow split. |
