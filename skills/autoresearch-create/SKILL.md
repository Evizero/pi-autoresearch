---
name: autoresearch-create
description: Set up and run an autonomous experiment loop for any optimization target. Gathers what to optimize, then starts the loop immediately. Use when asked to "run autoresearch", "optimize X in a loop", "set up autoresearch for X", or "start experiments".
---

# Autoresearch — Setup & Run

Set up an autonomous experiment loop and start running immediately.

## Tools

You have two custom tools from the autoresearch extension. **Always use these instead of raw bash for experiments:**

- **`run_experiment`** — pass it a `command` to run. It times execution, captures output, detects pass/fail via exit code.
- **`log_experiment`** — records each experiment's `commit`, `metric`, `status` (keep/discard/crash), and `description`. Persists state, updates the status widget and dashboard (toggle with ctrl+r).

  **On the first call**, always set these to configure the display:
  - `metric_name` — display name for primary metric (e.g. `"total_µs"`, `"bundle_kb"`, `"val_bpb"`)
  - `metric_unit` — unit string that controls number formatting: `"µs"`, `"ms"`, `"s"`, `"kb"`, or `""` for unitless. Integers get comma-separated thousands; fractional values get 2 decimal places.
  - `direction` — `"lower"` (default) or `"higher"` depending on what's better

  **Optional params (any call):**
  - `metrics` — dict of secondary metric name→value for tradeoff monitoring, e.g. `{"parse_µs": 5505, "render_µs": 1440}`
  - `new_baseline` — **rarely used.** Only set this when `autoresearch.sh` changed so substantially that old metric values are no longer comparable (e.g. you changed the optimization metric entirely, switched benchmarks, or altered the workload). This resets the baseline reference point so all future delta % comparisons are against the new measurement. The first experiment is automatically the baseline — you never need to set this during normal operation.

  The inline widget shows the current metric vs baseline as a delta %, e.g. `★ total_µs: 6,945 (-21.2%)`.

## Step 1: Gather Context

Ask the user (propose smart defaults based on the codebase):

1. **Goal** — what are we optimizing? (e.g. "reduce vitest execution time")
2. **Command** — shell command to run per experiment (e.g. `pnpm test:vitest`)
3. **Metric** — what number to measure, and is lower or higher better?
4. **Files in scope** — what can you modify?
5. **Constraints** — hard rules (e.g. "all tests must pass", "don't delete files")

If the user already provided these in their prompt, skip asking and confirm your understanding.

## Step 2: Setup

1. **Create a branch**: `git checkout -b autoresearch/<tag>` (propose a tag based on the goal + date).
2. **Read the relevant files** to understand what you're working with.
3. **Write `autoresearch.md`** and **`autoresearch.sh`** in the working directory. These are committed to the autoresearch branch and allow any agent to resume the work later.

### `autoresearch.md` — The experiment plan

This describes the full context of what we're doing. Write it so that a fresh agent session can read it and continue the work without any other context.

```markdown
# Autoresearch: <goal>

## Objective
<What we're optimizing and why. Be specific about the workload.>

## How to Run
Run `./autoresearch.sh` — it sets up the environment and runs the benchmark,
outputting metrics in the format the agent expects.

## Metrics
- **Primary (optimization target)**: <name> (<unit>, lower/higher is better)
- **Secondary (tradeoff monitoring)**: <name> (<unit>), <name> (<unit>), ...
<Explain what each metric measures and why it matters.>

## Files in Scope
<List every file/directory the agent is allowed to modify.>
- `path/to/file1` — <what it does>
- `path/to/file2` — <what it does>

## Off Limits
<Files/patterns that must NOT be modified.>
- `test/` — tests must continue to pass unchanged
- ...

## Constraints
<Hard rules the agent must respect.>
- All tests must pass
- No new dependencies
- ...

## Baseline
<Filled in after first run: primary + secondary metric values, commit hash.>

## Progress Log
<Append a one-liner for each significant keep: what changed, new metrics, commit.>
```

### `autoresearch.sh` — The benchmark runner

Write a self-contained shell script that:
1. Sets up the environment if needed (build, compile, install deps, etc.)
2. Runs the benchmark/test
3. Outputs metrics in a **parseable format** the agent can extract, e.g.:
   ```
   METRIC total_us=6945
   METRIC parse_us=5505
   METRIC render_us=1440
   METRIC allocations=39847
   ```
4. Exits 0 on success, non-zero on failure

The script should be `chmod +x` and work standalone: `./autoresearch.sh`

The agent uses `run_experiment` with `command: "./autoresearch.sh"` and parses the `METRIC` lines from the output to extract values for `log_experiment`.

**Important**: the script should be fast and deterministic. If the benchmark has variance, run multiple iterations and report the median or minimum.

4. **Commit both files**: `git add autoresearch.md autoresearch.sh && git commit -m "autoresearch: setup experiment plan and runner"`
5. **Run the baseline**: use `run_experiment` with `./autoresearch.sh`, parse the METRIC output, then `log_experiment` to record it. The first experiment automatically becomes the baseline. Set `metric_name`, `metric_unit`, and `direction` on this first call. Include secondary `metrics` from the METRIC lines. Update the Baseline section in `autoresearch.md`.
6. **Start looping** — do NOT wait for confirmation after the baseline. Go.

## Step 3: Experiment Loop

LOOP FOREVER:

1. Think of an experiment idea. Read the codebase for inspiration. Consider:
   - Config changes (parallelism, caching, pooling, environment)
   - Removing unnecessary work (unused setup, redundant transforms)
   - Structural changes (splitting, merging, reordering)
2. Edit files with the idea
3. Use `run_experiment` with `./autoresearch.sh`
4. Parse the `METRIC` lines from the output. Use `log_experiment` to record the result with the primary metric and secondary `metrics` dict. **`log_experiment` automatically commits** with the description as commit message and a `Result: {...}` trailer containing all metrics.
5. If metric improved AND constraints met → keep (status: `keep`). Append to the Progress Log in `autoresearch.md`.
6. If metric worse OR constraints broken → `git reset --hard HEAD~1` (status: `discard` or `crash`)
8. Repeat

**Simplicity criterion**: all else being equal, simpler is better. Removing code for equal results is a win.

**NEVER STOP.** Loop indefinitely until the user interrupts. Do not ask "should I continue?". The user can check progress anytime with ctrl+e.

**Resuming**: If you find `autoresearch.md` and `autoresearch.sh` already exist in the working directory, read them to understand the experiment context. Check the Progress Log for what's been tried. Then continue the loop from where it left off — no need to re-gather context or re-run the baseline.

**Crashes**: if it's a trivial fix (typo, missing import), fix and retry. If fundamentally broken, discard and move on.

## Example Domains

- **Test speed**: metric=seconds ↓, command=`pnpm test`, scope=vitest/jest configs
- **Bundle size**: metric=KB ↓, command=`pnpm build && du -sb dist`, scope=bundler config
- **Build speed**: metric=seconds ↓, command=`pnpm build`, scope=tsconfig + bundler
- **LLM training**: metric=val_bpb ↓, command=`uv run train.py`, scope=train.py
- **Lighthouse score**: metric=perf score ↑, command=`lighthouse --output=json`, scope=components
