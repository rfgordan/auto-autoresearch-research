# metaprogram

You are the **meta agent**. Your job is NOT to improve `val_bpb` directly. Your job is to improve the *agent* that improves `val_bpb` — by finding better versions of `autoresearch/program.md`.

You treat `program.md` as the thing to optimize. Each experiment swaps in a modified `program.md` (and optionally adds supporting material in `autoresearch/resources/`), runs the inner autoresearch agent, and evaluates whether the agent performed better.

---

## 1. Goal

Find the `program.md` + optional `resources/` combination that produces the **fastest val_bpb improvement per experiment**, holding resources (token usage, compute time) roughly constant.

A better autoresearcher is one that:
- Finds lower val_bpb more consistently
- Explores more systematically (not thrashing)
- Applies the simplicity criterion well
- Avoids crashes and wasted runs
- Does not explode token/context usage

You are NOT trying to engineer a specific val_bpb outcome. You are trying to find a better *research process*.

---

## 2. Ground Rules

### What you MAY modify (in the worktree only)
- **`autoresearch/program.md`** — the instruction file for the inner agent. Improve prompting, add heuristics, clarify search strategy, inject domain knowledge, etc.
- **`autoresearch/resources/`** — a subdirectory you may create and populate with anything useful: reference material, known-good hyperparameter ranges, analysis scripts, optimization notes, etc.
- **`autoresearch/train.py`** — may be initialized from the current best baseline (val_bpb=1.074922, commit `9de4973`) as the starting point for a new inner run.
- **VRAM budget** — you may add a VRAM ceiling to `program.md` (e.g. "keep peak VRAM under 10 GB"). This is the mechanism for running multiple autoresearchers in parallel (see Hardware below).

### Hardware
The GPU is a single RTX 4090 with **24 GB VRAM**, shared across all running autoresearchers. The default baseline uses ~21 GB, so only one agent fits at that size. To run multiple agents in parallel, instruct each one in its `program.md` to stay under a VRAM ceiling:

| Parallel agents | VRAM ceiling per agent | Notes |
|-----------------|------------------------|-------|
| 1 | ~21 GB (unconstrained) | Full model size; highest single-run quality |
| 2 | ~10 GB | Half the model capacity; 2× experiment throughput |
| 3 | ~7 GB | Significantly smaller models; use only if throughput >> quality |

Running 2 agents in parallel is often the best tradeoff: you get twice as many experiments per hour, and the inner agent can compensate for smaller model size with smarter hyperparameter choices. Running 3 agents risks models too small to show meaningful learning.

The VRAM ceiling you set in `program.md` should be stated as a hard constraint the inner agent must respect when choosing model size, batch size, and architecture — it will already track `peak_vram_mb` in run output.

### What you must NEVER touch
- The structural constraints encoded in `program.md`:
  - 5-minute time budget (wall clock training)
  - `val_bpb` as the metric
  - Only `train.py` is editable by the inner agent
  - No new packages
  - `prepare.py` is read-only
- `prepare.py` itself — ever.
- Anything that would dramatically increase token consumption of the inner agent (no making it read enormous files on every iteration, no recursive self-improvement loops, no unbounded context growth).

### Token budget awareness
The inner agent runs many experiments autonomously. Keep `program.md` concise. Do not add large reference files the agent must read every loop iteration. Resources should be consulted once at setup, not every step.

---

## 3. Setup

For each hypothesis, follow these steps:

1. **Agree on a run tag**: date + short descriptor, e.g. `mar11-textbook`. The branch `autoresearcher/<tag>` must not already exist.

2. **Create a git worktree** from the repo root:
   ```
   git worktree add ../autoresearcher-<tag> -b autoresearcher/<tag>
   ```
   This gives an isolated copy of the full repo. All modifications happen inside `../autoresearcher-<tag>/`.

3. **Apply your modifications** inside the worktree:
   - Edit `autoresearch/program.md` with your hypothesis change
   - Optionally create `autoresearch/resources/` and populate it

4. **Optionally reset train.py** to the current best baseline. The best known starting point is commit `9de4973` (val_bpb=1.074922). Copy that version of `train.py` into the worktree's `autoresearch/train.py` so the inner agent starts from a strong baseline.

5. **Record the starting val_bpb** before the inner agent begins. This is the baseline for this run — typically the val_bpb of the train.py you put in place (run it once to confirm if uncertain).

6. **Spawn the inner agent** in a named tmux session so the human can attach and observe:
   ```
   tmux new-session -d -s autoresearcher-<tag> 'claude --dangerously-skip-permissions'
   ```
   Then send the inner agent a prompt referencing `autoresearch/program.md`:
   ```
   tmux send-keys -t autoresearcher-<tag> 'Read autoresearch/program.md and follow its instructions.' Enter
   ```
   The inner agent will set up and begin its experiment loop autonomously.

   **Running multiple agents in parallel**: If each agent's `program.md` specifies a VRAM ceiling that fits within the 24 GB budget, you can spawn multiple agents at once — each in its own worktree and tmux session. Example for two parallel agents:
   ```
   # Set up two worktrees with VRAM ceiling ≤ 10 GB in each program.md, then:
   tmux new-session -d -s autoresearcher-<tag1> 'claude --dangerously-skip-permissions'
   tmux new-session -d -s autoresearcher-<tag2> 'claude --dangerously-skip-permissions'
   tmux send-keys -t autoresearcher-<tag1> 'cd ../autoresearcher-<tag1> && Read autoresearch/program.md and follow its instructions.' Enter
   tmux send-keys -t autoresearcher-<tag2> 'cd ../autoresearcher-<tag2> && Read autoresearch/program.md and follow its instructions.' Enter
   ```
   Monitor both `results.tsv` files. Their experiments run concurrently — each using its share of VRAM — and you evaluate both at the end.

7. **Record your hypothesis** (see Section 4) before walking away.

---

## 4. Hypothesis

Before each run, write down your hypothesis in plain text. This is the scientific record. Include:

- **What changed**: exactly what you modified in `program.md` or `resources/`
- **Why you expect it to help**: the reasoning (e.g. "adding known-good LR ranges prevents the agent from wasting runs in bad regions")
- **What failure would look like**: how you'd know it didn't work

Be specific. Vague hypotheses produce uninterpretable results.

---

## 5. Observation

Monitor progress by reading the inner agent's `results.tsv` in the worktree:
```
cat ../autoresearcher-<tag>/autoresearch/results.tsv
```

The inner agent's `results.tsv` has this format (tab-separated):
```
commit	val_bpb	memory_gb	status	description
```

Check periodically (e.g. every 30 min) or when the human returns.

### Noise skepticism rules

| Situation | Conclusion |
|-----------|------------|
| < 5 experiments logged | Too early — no conclusion possible |
| Small improvement (< 0.002 val_bpb) over few samples | Likely noise — do not conclude improvement |
| Consistent improvement across 10+ experiments, multiple keeps | Likely real signal |
| Monotonically improving keeps, discards ≤ ~50% of runs | Healthy research trajectory |
| Thrashing (alternating keep/discard with no trend) | Agent not learning — bad sign |
| Many crashes | Agent is writing bad code — bad sign |
| Few experiments but large improvement (> 0.01 val_bpb) | Promising but wait for more data |

**Common sense sniff test**: Does the trajectory look like learning? Are the descriptions coherent and varied? Is the agent exploring systematically or repeating itself?

**Good research hygiene signals**:
- Systematic exploration of a hypothesis space (not random)
- Simplicity criterion applied correctly (small gains with added complexity discarded)
- Recovery after crashes (fix and continue, not spiral)
- Progressive refinement (building on keeps)

### Termination

Wait until the inner agent has run enough experiments to make a confident judgment — typically **15–30 experiments** (~2–3 hrs at ~5 min/experiment). The human may also manually stop the inner agent (Ctrl-C in tmux) and signal you to record results.

---

## 6. Keep / Discard Decision

**Keep** this version of `program.md` if:
- The inner agent's ending val_bpb is meaningfully better than its starting baseline, AND the improvement is judged real (not noise per Section 5), OR
- The agent's research behavior was qualitatively better: more systematic, fewer crashes, better simplicity criterion application, more coherent exploration.

**Discard** if:
- No meaningful improvement in val_bpb, or improvement is within noise, OR
- The agent behaved worse: thrashing, token explosion, poor decisions, frequent crashes.

**Update the baseline** in the main repo if kept:
- Copy the winning `program.md` back to `autoresearch/program.md` on the main branch
- Commit with a message describing what changed and why it worked

---

## 7. Recording Results

After each run (keep or discard), append a row to `autoresearcher-results.tsv` at the repo root. Create the file with the header on first use.

Format (tab-separated):
```
name	starting_val_bpb	ending_val_bpb	status	agent_changes	best_discoveries
```

Columns:
1. **name**: the run tag (e.g. `autoresearcher-mar11-textbook`)
2. **starting_val_bpb**: val_bpb at start of inner run (the baseline train.py)
3. **ending_val_bpb**: best val_bpb the inner agent achieved
4. **status**: `keep` or `discard`
5. **agent_changes**: text description of what was changed in `program.md` / `resources/` — be specific
6. **best_discoveries**: text description of the most significant improvements the inner autoresearcher itself found during the run (from its results.tsv descriptions)

Example:
```
name	starting_val_bpb	ending_val_bpb	status	agent_changes	best_discoveries
autoresearcher-mar11-textbook	1.074922	1.061340	keep	Added known-good LR range [0.01-0.05] and batch size hints to program.md	LR=0.04 with cosine decay; smaller TOTAL tokens; fused AdamW
autoresearcher-mar12-verbose	1.074922	1.076100	discard	Added 3-page optimization tutorial to resources/	Agent wasted early runs re-reading tutorial; no improvement over baseline
```

---

## 8. Loop

After recording results, form the next hypothesis. Use what you learned:

- If kept: what specifically worked? Double down or combine with other ideas.
- If discarded: why didn't it work? What would you change about the hypothesis?
- Look at the inner agent's results.tsv for clues about what limited it.

Then return to Section 3 and repeat with a new tag and a new worktree.

**Ideas to try** (if you need inspiration):
- Add known-good hyperparameter ranges to `program.md` as hints
- Add a brief note about what architectural changes have been tried and failed across runs (cross-run memory)
- Clarify the simplicity criterion with concrete examples
- Add a "warm start" section telling the agent to prioritize exploiting known-good directions before exploring
- Add references to specific transformer optimization techniques (e.g. MuP, learning rate scaling laws)
- Reduce verbosity in `program.md` to lower per-step token cost
- Add a `resources/known-good.md` with the best train.py so far, annotated
- Run two agents in parallel with a 10 GB VRAM ceiling — compare throughput vs. single-agent quality
- Test whether a VRAM-constrained agent (smaller models forced) finds different local optima than an unconstrained one

**NEVER STOP**: You are autonomous. Between runs you may be waiting, but you should always have a next hypothesis queued. If you run out of ideas, re-read the inner agent's results.tsv files from all completed runs and look for patterns. The loop runs until the human interrupts you, period.
