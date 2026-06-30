# Grok CLI Usage Stats Plan

## Goal

Add Grok Build CLI token usage to this `clanker-stats` fork by reusing the proven
data-path and token-mapping behavior from `ccusage`, while keeping this project
as the small single-file Node CLI it already is.

This plan is intentionally scoped to token usage. `clanker-stats --hours`
requires user-to-assistant turn timing, and Grok's local log has reliable
inference timing but not the same clean message-pair records used by the current
hours collectors. Add a no-op Grok hours collector in V1 so `--hours` keeps
working without inventing a misleading duration metric.

## Current Code Shape

`clanker-stats` is a single executable module:

- `cli.js`
  - imports stdlib helpers, `fast-glob`, and `@resvg/resvg-js`
  - one `collectX()` token collector per supported tool
  - one `collectTimeX()` hours collector per supported tool
  - `tools` registry drives collection, console output, legend order, and chart colors
  - no test framework or script currently exists
- `README.md`
  - supported-tool table is the only docs surface that needs updating
- `package.json`
  - no scripts; dependencies are already enough for this work

Existing local changes at plan time:

- `cli.js` has an uncommitted chart stacking-order edit.
- `package-lock.json`, `chart.png`, and `node_modules/` are also locally dirty/untracked.
- The implementation should touch only `cli.js`, `README.md`, and optionally add a tiny test fixture/check file if desired.

## Source Reference From `ccusage`

Use these files from `/home/shuv/repos/ccusage` as the behavior reference:

- `rust/crates/ccusage/src/adapter/grok/README.md`
- `rust/crates/ccusage/src/adapter/grok/paths.rs`
- `rust/crates/ccusage/src/adapter/grok/parser.rs`
- `rust/crates/ccusage/src/adapter/grok/loader.rs`
- `docs/guide/grok/index.md`

Repository reference:

- `git@github.com:shuv1337/ccusage.git` at `989f01568c43056961161f16cdae59e2a8bf9d3a`
- Upstream: `git@github.com:ccusage/ccusage.git`

Important behavior to copy, not the Rust architecture:

- Grok home directory is `process.env.GROK_HOME || join(homedir(), ".grok")`.
- Token log is `${grokHome}/logs/unified.jsonl`.
- Session metadata is `${grokHome}/sessions/<url-encoded-cwd>/<session-id>/summary.json`.
- Include only lines where `msg === "shell.turn.inference_done"`.
- Require `sid`, `ts`, `ctx.loop_index`, `ctx.prompt_tokens`, `ctx.cached_prompt_tokens`, and `ctx.completion_tokens`.
- Token mapping:
  - input tokens: `max(prompt_tokens - cached_prompt_tokens, 0)`
  - cache-read tokens: `cached_prompt_tokens`
  - output tokens: `completion_tokens`
  - reasoning tokens: `reasoning_tokens || 0`
  - total counted by `clanker-stats`: input + cache-read + output + reasoning
- Missing `logs/unified.jsonl` should produce an empty `Map`, not a warning or crash.
- Unknown session metadata is fine; `clanker-stats` only needs date totals.
- `ccusage` counts one row per inference, not one row per user turn. Preserve that behavior.

Local sanity check at plan time:

- `~/.grok/logs/unified.jsonl` exists and has `20083` lines.
- Sample matching events have `ts`, `sid`, `msg: "shell.turn.inference_done"`, and `ctx` token fields.
- `~/.grok/sessions/**/summary.json` exists and includes `info.id`, `info.cwd`, and `current_model_id`.

## Implementation Tasks

### 1. Add small Grok path helpers

- [ ] In `cli.js`, keep helpers near the other top-level helpers.
- [ ] Add `grokHome()`:

```js
function grokHome() {
  return process.env.GROK_HOME?.trim() || join(home, ".grok")
}
```

- [ ] No CLI flag for alternate Grok home in this fork. Existing collectors do not have per-tool flags, and `GROK_HOME` is enough.

### 2. Add the token collector

- [ ] Add `collectGrok()` near the other token collectors.
- [ ] Stream `logs/unified.jsonl` with `createReadStream` + `createInterface`, matching `collectCodex()` and `collectDroid()` style.
- [ ] Use `fg` only if session metadata becomes necessary later; for V1 token totals, unified log alone is enough.
- [ ] Fast-skip lines that do not include `"shell.turn.inference_done"`.
- [ ] Parse JSON defensively inside `try/catch`, matching the existing collector style.
- [ ] Validate required shape before counting:

```js
obj.msg === "shell.turn.inference_done"
obj.sid
obj.ts
obj.ctx?.loop_index != null
obj.ctx?.prompt_tokens != null
obj.ctx?.cached_prompt_tokens != null
obj.ctx?.completion_tokens != null
```

- [ ] Calculate tokens:

```js
const input = Math.max((ctx.prompt_tokens || 0) - (ctx.cached_prompt_tokens || 0), 0)
const tokens = input +
  (ctx.cached_prompt_tokens || 0) +
  (ctx.completion_tokens || 0) +
  (ctx.reasoning_tokens || 0)
```

- [ ] Add positive totals by `isoToDate(obj.ts)`.
- [ ] If `unified.jsonl` does not exist or cannot be opened because the directory is absent, return an empty `Map`.
- [ ] Let unexpected read errors be caught by `main()` and shown as `Grok: skipped (...)`, consistent with existing tools.

Expected shape:

```js
async function collectGrok() {
  const counts = new Map()
  const logPath = join(grokHome(), "logs", "unified.jsonl")
  try {
    const rl = createInterface({ input: createReadStream(logPath), crlfDelay: Infinity })
    for await (const line of rl) {
      if (!line.includes('"shell.turn.inference_done"')) continue
      try {
        const obj = JSON.parse(line)
        const ctx = obj.ctx
        if (obj.msg !== "shell.turn.inference_done" || !obj.sid || !obj.ts || !ctx) continue
        if (ctx.loop_index == null || ctx.prompt_tokens == null || ctx.cached_prompt_tokens == null || ctx.completion_tokens == null) continue
        const input = Math.max((ctx.prompt_tokens || 0) - (ctx.cached_prompt_tokens || 0), 0)
        const tokens = input + (ctx.cached_prompt_tokens || 0) + (ctx.completion_tokens || 0) + (ctx.reasoning_tokens || 0)
        if (tokens > 0) add(counts, isoToDate(obj.ts), tokens)
      } catch {}
    }
  } catch (e) {
    if (e?.code !== "ENOENT") throw e
  }
  return counts
}
```

### 3. Add a no-op hours collector

- [ ] Add `collectTimeGrok()`.
- [ ] Return `new Map()` for V1.
- [ ] Add a short comment explaining the ceiling:

```js
async function collectTimeGrok() {
  // ponytail: Grok logs inference durations, not clean user->assistant turns; add hours when a stable turn boundary is confirmed.
  return new Map()
}
```

Do not sum `model_elapsed_ms` or `elapsed_since_turn_start_ms` for `--hours` in V1. That would measure model runtime or loop runtime, while the existing chart says "hours" and approximates wall-clock user-to-last-assistant turn time.

### 4. Register Grok in the chart

- [ ] Add one entry to `tools`.
- [ ] Place it near the other xAI-era coding agents; after `Pi` or before `Mistral Vibe` is fine.
- [ ] Pick a color that does not collide with existing legend colors. Suggested: `#f0f6fc` is too close to the total line, so use `#94a3b8` or `#06b6d4`.

```js
{ name: "Grok", collect: collectGrok, collectTime: collectTimeGrok, color: "#06b6d4" },
```

### 5. Update docs

- [ ] In `README.md`, add Grok to the supported tools table:

```md
| Grok CLI | `~/.grok/logs/unified.jsonl` |
```

- [ ] Mention `GROK_HOME` only if there is already a docs section for env vars. There is not, so skip it unless implementation adds a short note.

### 6. Add the smallest useful check

There is no test framework. Avoid adding one for this feature.

- [ ] Run syntax validation:

```sh
node --check cli.js
```

- [ ] Run the CLI against local data:

```sh
node cli.js
```

- [ ] Confirm output includes a `Grok: ... tokens` line when `~/.grok/logs/unified.jsonl` exists.
- [ ] Confirm `chart.png` is generated.
- [ ] Run hours mode and confirm Grok does not crash it:

```sh
node cli.js --hours
```

- [ ] If a deterministic check is desired without a test framework, add a temporary manual fixture command in the PR notes instead of committing fixture machinery:

```sh
tmp=$(mktemp -d)
mkdir -p "$tmp/logs"
cat > "$tmp/logs/unified.jsonl" <<'JSONL'
{"ts":"2026-06-26T10:00:00.000Z","sid":"s1","msg":"shell.turn.inference_done","ctx":{"loop_index":1,"prompt_tokens":100,"cached_prompt_tokens":30,"completion_tokens":20,"reasoning_tokens":5}}
{"ts":"2026-06-26T11:00:00.000Z","sid":"s1","msg":"shell.turn.prompt_received","ctx":{}}
JSONL
GROK_HOME="$tmp" node cli.js
```

Expected Grok fixture total: `155 tokens`.

## Edge Cases

- [ ] Missing `~/.grok` or missing `logs/unified.jsonl`: collector returns empty counts.
- [ ] Malformed JSONL line: skip it.
- [ ] `cached_prompt_tokens > prompt_tokens`: input is `0`, cache-read still counts.
- [ ] Missing required token fields: skip the line.
- [ ] Missing `reasoning_tokens`: treat as `0`.
- [ ] Multiple inference loops for one user prompt: count every inference line, matching `ccusage`.
- [ ] `GROK_HOME` with whitespace only: ignore it and use `~/.grok`.

## Non-Goals

- [ ] Do not port `ccusage` session reports, pricing, model names, or project grouping.
- [ ] Do not parse `summary.json` in V1. It is useful for session/project/model reports, but this chart only needs date totals.
- [ ] Do not add a test framework.
- [ ] Do not add a `--grok-home` flag unless the project later grows per-tool flags.
- [ ] Do not implement Grok hours until there is a validated turn boundary equivalent to the other tools.

## Done Criteria

- [ ] `cli.js` has `collectGrok()` and `collectTimeGrok()`.
- [ ] `tools` includes Grok.
- [ ] `README.md` lists Grok CLI and its data source.
- [ ] `node --check cli.js` passes.
- [ ] `node cli.js` prints a Grok token total on a machine with Grok data.
- [ ] `node cli.js --hours` still completes.
- [ ] No unrelated local changes are reverted or reformatted.
