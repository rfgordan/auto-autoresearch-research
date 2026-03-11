# auto-autoresearch-research

An agent that iteratively experiments on autoresearch agents.

## Background

[autoresearch](https://github.com/karpathy/autoresearch) (Karpathy, March 2026) is a framework where an AI agent is given a small LLM training setup and experiments autonomously: modifying `train.py`, running 5-minute training jobs, checking if validation loss improved, and repeating indefinitely. The human's lever is `program.md` — a Markdown file that instructs the agent on how to behave: what to try, how to log results, when to keep or discard changes.

The insight is that there are two levels of iteration happening:
- **Inner loop**: the agent modifying `train.py` to improve `val_bpb`
- **Outer loop**: a human modifying `program.md` to improve the agent itself

This repo automates the outer loop.

## What this repo does

Instead of a human hand-tuning `program.md`, an agent here treats `program.md` as the thing to optimize. It runs autoresearch experiments, observes the agent's behavior and outcomes, proposes edits to `program.md`, and measures whether those edits produce better research progress.

The idea: agents are programs. Programs can be iterated on. Why not let an agent do it?

## How agents work (brief)

An **AI agent** is an LLM in a loop: it receives observations (tool outputs, file contents, command results), decides on an action, executes it, and repeats. The key properties that make agents useful for research:

- **Autonomy**: they can run for hours without human input
- **Tool use**: they can read/write files, run shell commands, and check results
- **Self-direction**: given a goal and constraints, they decide *what to try next*

autoresearch exploits all three — the agent reads `program.md` to understand the task, modifies `train.py`, runs training, and decides whether to keep or revert based on results.

This repo adds a second agent layer on top: one that reads the autoresearch outcomes and improves the instructions driving the inner agent.

## Structure

```
autoresearch/       — the autoresearch subproject (inner loop)
  train.py          — model + training loop (inner agent edits this)
  program.md        — agent instructions (outer agent edits this)
  prepare.py        — fixed data prep + eval utilities (never modified)
  results.tsv       — experiment log
README.md           — this file
```

## Relationship to autoresearch

This repo forks from [rfgordan/auto-autoresearch-research](https://github.com/rfgordan/auto-autoresearch-research), which itself builds on [karpathy/autoresearch](https://github.com/karpathy/autoresearch). The inner autoresearch loop is largely unchanged — the contribution here is the outer optimization loop over `program.md`.

## License

MIT
