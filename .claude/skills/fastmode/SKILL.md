---
name: fastmode
description: Run a browser task with the jev-ultrafast agent (Browser Use × TypeSafe). Invoke with /fastmode followed by a natural-language goal, optionally a --url. Example: /fastmode --url https://www.google.com/travel/flights find one-way flights Zurich to London on 2026-09-20 for one adult in economy. The agent drives a real Chrome via Browser Harness, so it only truly runs on a machine with Chrome and the required API keys (your desktop), not the cloud side panel.
---

# fastmode — run the jev-ultrafast browser agent

This skill runs [jev-ultrafast](https://github.com/browser-use/jev-ultrafast), a browser
agent with a dynamic, indexed action space (Browser Use × TypeSafe). One natural-language
goal in; the agent picks an operation + target per cycle and drives a real Chrome.

## What the user passes

Everything after `/fastmode` is the request. Parse it into:

- **goal** (required): the natural-language task, e.g. `find one-way flights Zurich to London on 2026-09-20 for one adult in economy, stop when flight options are visible`.
- **--url** (optional): the starting page. If omitted, infer a sensible start URL from the
  goal, or ask the user for one. Do NOT invent site-specific action scripts — jev takes a
  goal and a start URL, nothing more.
- **--keep-open** (optional): leave the browser open after the run.
- **--inspector** (optional): launch the local web inspector instead of a headless-style run.

## Step 0 — Environment reality check (do this FIRST)

jev-ultrafast needs a **real Chrome** (connected via Browser Harness), **`uv`**, and two
**paid API keys**. It cannot run in an environment that lacks these.

Run the preflight below. If any check fails, STOP and report exactly what is missing and how
to fix it. Do not pretend a run happened. In particular, a cloud / web side-panel session
typically has no Chrome and no keys — in that case say so plainly and point the user to run
`/fastmode` from Claude Code on their desktop.

```bash
# uv present?
command -v uv >/dev/null || echo "MISSING: uv (install: https://docs.astral.sh/uv/)"
```

## Step 1 — Locate (or fetch) jev-ultrafast

Resolve the jev-ultrafast checkout in this order, using the first that exists:

1. `$JEV_ULTRAFAST_DIR` if set.
2. A sibling of the current repo: `../jev-ultrafast`.
3. `~/jev-ultrafast`.
4. Otherwise clone it: `git clone https://github.com/browser-use/jev-ultrafast.git <dir>`
   (default `<dir>` = `~/jev-ultrafast`). Tell the user where you cloned it.

Set `JEV_DIR` to the resolved path for the remaining steps.

```bash
JEV_DIR="${JEV_ULTRAFAST_DIR:-}"
for cand in "$JEV_ULTRAFAST_DIR" ../jev-ultrafast ~/jev-ultrafast; do
  [ -n "$cand" ] && [ -f "$cand/pyproject.toml" ] && JEV_DIR="$cand" && break
done
if [ -z "$JEV_DIR" ]; then
  JEV_DIR=~/jev-ultrafast
  git clone https://github.com/browser-use/jev-ultrafast.git "$JEV_DIR"
fi
echo "Using jev-ultrafast at: $JEV_DIR"
```

## Step 2 — Install + keys

```bash
cd "$JEV_DIR"
uv sync                        # installs deps incl. Browser Harness
[ -f .env ] || cp .env.example .env
```

Then confirm `.env` contains real values for both keys. If either is a placeholder or empty,
STOP and ask the user to add them (do not print the key values back):

- `TYPESAFE_API_KEY` — TypeSafe.
- `TEXT_MODEL_API_KEY` — the text model (OpenRouter key in the example config; the demo uses
  `inception/mercury-2.5` with reasoning disabled).

Check Chrome connectivity (needs remote debugging; allow it in Chrome if prompted):

```bash
cd "$JEV_DIR" && uv run browser-harness --doctor
```

If the doctor reports no browser, STOP: there is no Chrome to drive. This is the expected
failure in a cloud/headless session — report it and suggest running on the desktop.

## Step 3 — Run

**Headless-style task run** (default):

```bash
cd "$JEV_DIR" && uv run --env-file .env python examples/run.py \
  --url "<START_URL>" \
  --goal "<GOAL>"
```

**Local inspector** (`--inspector`): launches the web UI on http://127.0.0.1:8766, where the
user clicks Start demo → Run automatically, or Choose next to step through.

```bash
cd "$JEV_DIR" && uv run jev
```

Pass `--keep-open` through to `examples/run.py` when requested.

## Step 4 — Report honestly

jev's own `DONE` is not proof of success. Report what actually happened: final URL, the
agent's status/elapsed_ms from the state stream, and whether the goal's verifiable outcome was
reached. If the agent returned `BLOCKED`, say what blocked it. If you could not verify the
outcome, say that rather than claiming success.

## Hard rules (from the project's AGENTS.md — do not violate)

- Input is one natural-language goal. No site-specific plans, no hardcoded field values.
- Never let the model emit selectors, coordinates, shell commands, or executable JavaScript.
- Never retry a browser mutation.
- Keep credentials server-side; never print key values or commit `.env`.
- Tests must not call paid APIs. Live runs DO make paid API calls — only run when the user
  asked for an actual browser task.
