# long-horizon-stage-segmentation

A reusable agent skill for designing per-frame stage prompts, building causal
signal-based automatic trajectory segmentation scripts, and integrating
usable-only auto-segmentation into robotic data collection, trajectory editing,
dataset export, and online inference trackers for long-horizon manipulation
tasks (single-arm or bimanual).

## Contents

- [`SKILL.md`](./SKILL.md) — Five-phase workflow runbook and verification checklist
- [`references/prompt-and-boundary-design.md`](./references/prompt-and-boundary-design.md) — Stage granularity, prompt syntax, causal boundaries, bimanual concurrency, handovers, and cyclic sub-stages
- [`references/pipeline-integration.md`](./references/pipeline-integration.md) — Module architecture, source-frame index schema, two-tier flagging & verdicts, usable-only collection/editor/export hooks, and online tracker parity

## Installation

### Global (all projects on this machine)

```bash
git clone https://github.com/loulouzy/long-horizon-stage-segmentation.git \
  ~/.gemini/config/skills/long-horizon-stage-segmentation
```

### Project-scoped

```bash
git clone https://github.com/loulouzy/long-horizon-stage-segmentation.git \
  .agents/skills/long-horizon-stage-segmentation
```
