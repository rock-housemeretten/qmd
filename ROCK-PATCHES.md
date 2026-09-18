# Rock patches on `rock/pinned`

This branch is upstream `tobi/qmd` at **`785bbcf`** plus the patches below, in order. Rock Neo installs it pinned by sha (`bun add -g qmd@github:rock-housemeretten/qmd#<sha>`); the human-facing record of the current pin is `✱ LLM/ROCK-REBUILD.md` §qmd. Upstream has since moved to v2.8.x with a different index schema — a version bump is its own project, never `bun update -g`.

| # | commit | date | patch | files |
|---|---|---|---|---|
| 1 | `ef808c1` | 2026-08-20 | **Idle model reclaim** — long-lived `qmd mcp` servers kept llama.cpp weights Metal-resident forever; seven of them exhausted the GPU working set machine-wide. Both MCP entrypoints enable `disposeModelsOnInactivity`; `configureDefaultLlamaCpp()` seam + `QMD_LLM_DISPOSE_MODELS` / `QMD_LLM_IDLE_MS`; the idle timer also arms with models loaded and no live context. | `src/llm.ts` `src/mcp.ts` `src/llm.test.ts` |
| 2 | `ef808c1` | 2026-09-01 | **Bounded KV context** — same commit. | `src/llm.ts` |
| 3 | `6884de6` (fork PR #1) | 2026-09-15 | **Tier 0 chunk snippet** (Rock #266) — the `vsearch` snippet comes from the *matched* chunk, not the document head; `--json` carries `chunkPos` / `chunkSeq`. | `src/store.ts` `src/qmd.ts` `src/store.test.ts` |
| 4 | `f41d6df` (fork PR #2) | 2026-09-16 | **Tier 1 `qmd passages`** (Rock #266) — rank one document's chunks against a query. | `src/qmd.ts` `src/store.ts` `src/store.test.ts` |
| 5 | this branch (fork PR #3) | 2026-09-18 | **Keep `llm_cache` across `update` / `collection add`** (Rock #339) — `updateCollections()` and `indexFiles()` ran `DELETE FROM llm_cache` unconditionally, an Ollama-era leftover. The cache holds query *expansions* keyed on `{query, model}`; index contents are not in the key, so a re-index does not invalidate it. With heartbeat running `qmd update` every 30 min, every variant cache lived ≤ 30 min and `vsearch` regenerated variants at temperature 0.7 after each wipe — the ranking re-rolled twice an hour under every A/B. The explicit `qmd cleanup` still empties it; the 1,000-row LRU trim still bounds it. | `src/qmd.ts` `src/cli.test.ts` |

## Working on this branch

- Scratch clone: `git clone --branch rock/pinned git@github-rock:rock-housemeretten/qmd.git`, then **symlink `node_modules` → `~/.bun/install/global/node_modules`** (the global install's deps; never the package's own dir, never a fresh `bun install` — the native llama build is what you want to reuse).
- PRs: `gh pr create --repo rock-housemeretten/qmd --base rock/pinned` — the fork's default base is upstream `tobi/qmd`; forgetting `--repo` opens a PR against Tobi.
- After merge: re-pin (Alex's call) with the merge sha, update ROCK-REBUILD §qmd. Running `qmd mcp` servers keep the old code until their session restarts.
- Baseline suite quirks (pre-existing, not ours): the MCP HTTP transport tests fail on stateless-transport reuse, and the suite can crash at exit in ggml-metal residency teardown. `cli.test.ts` + `store.test.ts` are the ones that must be green.
