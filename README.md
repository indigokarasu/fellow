# 🧪 Fellow

Fellow is the system's empirical optimization engine, invoked exclusively by Mentor to determine which implementation of a skill, prompt, heuristic, or workflow actually performs best -- not which one looks best on paper. It runs controlled experiments with a fixed benchmark and compute budget, establishes a fresh baseline before testing any variant, and returns the winning result with full mutation lineage so every promotion is traceable and reversible.


Skill packages follow the [agentskills.io](https://agentskills.io/specification) open standard and are compatible with OpenClaw, Hermes Agent, and any agentskills.io-compliant client.

---

## Overview

Fellow exists because "looks better on paper" is not the same as "works better in practice." When Mentor identifies a skill or heuristic that needs improvement, Fellow takes over the empirical work -- establishing a fresh baseline, generating and testing controlled variants within a constrained mutation surface, and returning the winning implementation with full lineage. Every experiment runs against a fixed benchmark and compute budget so results across different targets remain comparable and every promotion is reversible. Fellow is not user-invocable; it is called only by Mentor.

## Commands

| Command | Description |
|---|---|
| `fellow.experiment.run` | Execute an experiment cycle from a Mentor invocation payload |
| `fellow.experiment.status` | Current experiment state if a cycle is in progress |
| `fellow.journal` | Write journal for the current run |
| `fellow.update` | Pull latest from GitHub source (preserves journals and data) |

## Setup

`fellow.init` runs automatically on first invocation by Mentor and creates all required directories, config.json, and JSONL files. No manual setup is required. Fellow is purely reactive -- invoked only by Mentor, with no cron jobs or heartbeat entries.

## Dependencies

**OCAS Skills**
- [Mentor](https://github.com/indigokarasu/mentor) -- sole invoker; provides experiment programs and approves promotions
- [Elephas](https://github.com/indigokarasu/elephas) -- stores experiment lineage and artifacts via journal signal payloads

**External**
- None

## Scheduled Tasks

| Job | Mechanism | Schedule | Command |
|---|---|---|---|
| `fellow:update` | cron | `0 0 * * *` (midnight daily) | Self-update from GitHub source | Invoked only by Mentor.

## Changelog

### v2.5.0 -- April 2, 2026
- Structured entity observations in journal payloads (`entities_observed`, `relationships_observed`, `preferences_observed`)
- `user_relevance` tagging on journal observations (default `agent_only` for experiment entities, `user` for user-preference experiments)
- Elephas journal cooperation in skill cooperation section

### v2.3.0 -- March 27, 2026
- Added `fellow.update` command and midnight cron for automatic version-checked self-updates

### v2.2.0 -- March 22, 2026
- Routing aliases and trigger phrases

### v2.1.0 -- March 22, 2026
- Full experiment lifecycle implementation: baseline, variant generation, benchmark execution, metric extraction, promotion
- Invocation contract schema

### v2.0.0 -- March 18, 2026
- Initial release as part of the unified OCAS skill suite
---

*Fellow is part of the [OCAS Agent Suite](https://github.com/indigokarasu) -- a collection of interconnected skills for personal intelligence, autonomous research, and continuous self-improvement. Each skill owns a narrow responsibility and communicates with others through structured signal files, shared journals, and Chronicle, a long-term knowledge graph that accumulates verified facts over time.*
