# 🔎 Fellow

> **Experimentation engine — runs controlled benchmark experiments to validate skill improvements.**

## Why Fellow?

How do you know if a skill improvement actually made things better? Fellow answers that by running controlled benchmark experiments. When Mentor generates an improvement proposal, Fellow designs and executes the experiment, measures outcomes against baselines, and produces evidence-based results.

Skill packages follow the [agentskills.io](https://agentskills.io/specification) open standard and are compatible with OpenClaw, Hermes Agent, Claude, and any agentskills.io-compliant client.

## Quick Start

```
# Run an experiment
"Run a benchmark comparing the old and new versions of Sands"

# Check results
"What were the results of the last experiment?"
```

## What It Does

Fellow is the empirical testing arm of the OCAS self-improvement loop. It receives experiment requests (typically routed through Mentor), designs controlled benchmarks, executes them, and measures outcomes. Results flow back to Mentor for evaluation and potential promotion.

## Dependencies

- [Mentor](https://github.com/indigokarasu/mentor) — receives experiment requests
- Target skills under evaluation

---

*Fellow is part of the [OCAS Agent Suite](https://github.com/indigokarasu).*