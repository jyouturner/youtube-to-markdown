# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Run / dev / test

- **Bootstrap + launch** (preferred during development): `./run.sh` — checks ffmpeg, installs uv if missing, runs `uv sync`, prompts for `ANTHROPIC_API_KEY` if not in `~/yt2md/.env`, runs `yt2md doctor`, then `yt2md serve`. Idempotent; safe to re-run.
- **Direct CLI** (after `uv sync`): `uv run yt2md <subcommand>` — e.g. `serve`, `digest <url>`, `list`, `read <id>`, `search <q>`, `watch {add,list,remove,run}`, `mcp`, `doctor`, `topics`, `retrofit-topics`, `refresh-pricing` (recalibrate the price table from real Anthropic billing — needs `ANTHROPIC_ADMIN_KEY`).
- **Default web port**: 7682. `--host 0.0.0.0` exposes the library on the LAN (used by the `/listen` QR-code phone flow); default `127.0.0.1` is localhost-only.
- **Slide regression check** (the only "test" — single curated fixture): regenerate the deck for the fixture video via the web reader, then `uv run python tests/compare_slides.py tests/fixtures/igO8iyca2_g.pptx ~/yt2md/digests/igO8iyca2_g/slides.pptx`. Run after touching `extract_*_frames`, `dedupe_frames`, `global_phash_cluster`, `_render_classification_grids`, `classify_slides_via_grids`, `assign_transcript_to_frames`, or `build_deck`. LLM noise — re-run once before treating a failure as a regression.

There is no lint or unit-test suite. The PostToolUse hook in `.claude/settings.json` `ast.parse`s any edited `.py` and blocks (exit 2) on a SyntaxError — so an Edit/Write that leaves `youtube_to_markdown.py` unparseable will fail immediately.

## Architecture

**Single-file design.** Everything ships from `youtube_to_markdown.py` (~10k lines). `pyproject.toml` is configured as `only-include = ["youtube_to_markdown.py"]` — do not split the module without updating the wheel target. The two `[project.scripts]` entry points (`yt2md`, `youtube-to-markdown`) both call `main()` in that file.

**One CLI, many subcommands, dispatched in `main()`:**
- Default (no subcommand): single-video digest flow — `yt2md <url-or-mp4> [srt]` runs the whole pipeline once and writes markdown next to the input.
- `serve` — Flask web reader on `127.0.0.1:7682`. Also runs the in-process scheduler that polls subscriptions. This is the primary UX; most routes live inside `cmd_serve`.
- `watch {add,list,remove,run}` — manage channel subscriptions in `~/yt2md/channels.txt`; `watch run` polls all channels once. On the API backend it defers each video's panel + takeaway (via `--no-panel --no-takeaway`) and runs them as one Message Batch across the whole poll (50% off) — see *Batched background generation* under Architecture.
- `mcp` — start the MCP stdio server. The same library surface (`list_digests`, `read_digest`, `search_library`, `digest_video`, `generate_panel/takeaway/slides`, `job_status`, `*_subscription`) is exposed as `@mcp.tool()` AND as shell subcommands (`list`, `read`, `search`, `digest`, `topics`, `retrofit-topics`) — keep the two surfaces in sync when you add a capability.
- `doctor` — preflight check (Node 20+ for yt-dlp, ffmpeg, cookies, API key, and a **Cost controls** block: admin key, budget-gate thresholds, calibrated-vs-builtin pricing, spend-cap reminder).

**Pipeline stages (digest of one video):**
1. `fetch_youtube` → MP4 + SRT (yt-dlp; falls back to Whisper local transcription via `_transcribe_with_whisper` if no captions; needs Node 20+ via `_ensure_js_runtime_available`).
2. `extract_scene_and_interval_frames` → **one ffmpeg subprocess with `filter_complex` split** producing scene frames + periodic frames from a single decode. The two-subprocess version it replaced doubled wall time; do not regress this.
3. `dedupe_frames` (pHash) + `global_phash_cluster` (cross-talk dedup) → candidate frames.
4. `_render_classification_grids` + `classify_slides_via_grids` (vision LLM in batches) → keep only real deck slides.
5. `assign_transcript_to_frames` → pair slides with transcript snippets.
6. `generate_digest` (Sonnet, Pydantic-typed JSON) → topic segmentation.
7. `vision_pick_frames` → per-topic representative frame.
8. `generate_panel_discussion` (Opus by default; the expensive call) → multi-expert critique.
9. `generate_takeaway_prose` → synthesis prose with inline timestamp links.
10. `build_deck` → `slides.pptx`.
11. `generate_audio_from_markdown` → MP3s for the podcast feed (macOS `say` or ElevenLabs).

Every artifact is regeneratable from the cached `downloads/` directory — the per-video page has `Generate X` buttons that retry a single stage.

**LLM backend abstraction.** Three implementations:
- `AnthropicAPIBackend` — uses the Anthropic SDK + `ANTHROPIC_API_KEY`. Supports native vision and prompt caching.
- `ClaudeCodeBackend` — shells out via `claude -p` to a sandboxed local Claude Code install. Vision is opt-in via the `claude_code_vision` setting. **Billing changes Jun 15 2026**: Anthropic moves `claude -p` (and Agent SDK) usage from Pro/Max interactive limits to a separate metered "Agent SDK credit" pool — i.e. starts billing at API rates. Until then, calls draw from the user's Pro/Max plan if the sandbox is OAuth-logged-in; falls back to `ANTHROPIC_API_KEY` if not.
- `ClaudeCodePtyBackend` — drives the user's *primary* `claude` install (resolved via `shutil.which("claude")`) as an interactive REPL under a PTY, using `pyte` to render the TUI to a virtual screen and `\x1b[200~...\x1b[201~` bracketed-paste for long prompts. Stays on Pro/Max subscription billing post-Jun-2026 because the `-p` flag is the billing trigger. No token usage envelope (interactive mode emits none), no native vision, ~10–15s per call (REPL startup). End-of-response detection is the `✻ <verb> for Ns` footer Claude Code prints after each response — the verb rotates ("Brewed", "Cogitated", etc.), match on shape. Strips `ANTHROPIC_API_KEY` from the subprocess env so the REPL can't silently fall back to API billing.

`select_backend()` resolves which one to use from `settings["llm_backend"]` ∈ `{auto, api, claude-code, claude-code-pty}`. Auto prefers API when the key is set, else falls back to Claude Code only when both installed *and* the cached login sentinel is present (so the caller redirects to `/setup` rather than burning a doomed call). `claude-code-pty` is never auto-selected — opt-in only via settings. New LLM calls should accept an optional `backend=` and default to `select_backend()`.

**Hybrid vision routing (built into PTY mode).** PTY can't do vision, so `select_backend(for_vision=True)` transparently returns `AnthropicAPIBackend()` when the primary choice is `claude-code-pty` AND `ANTHROPIC_API_KEY` is set. No new user setting — the routing is implicit. Vision callsites (`classify_slides_via_grids`, `vision_pick_frames` in `main()` and `build_slides_for_video`) all pass `for_vision=True` so PTY users automatically get API-backed frame picking when a key is available, falling back to timestamp-based picks otherwise. Cost impact: image stages (~$0.18/video) keep using API while text/panel/digest/takeaway (~$0.76/video) move to the Pro/Max subscription.

**Batched background generation (Message Batches, 50% off).** The panel and takeaway are the two text LLM calls; on the *background* subscription path (`watch run` / scheduler) nobody is waiting on the result, so they go through the Anthropic Message Batches API — a flat 50% token discount — instead of the direct API. `cmd_watch_run` computes `use_batch = _batch_capable()` once, passes `defer_panel=use_batch` to each per-video subprocess (which appends `--no-panel --no-takeaway`), collects the succeeded videos across all channels, then runs `batch_panels_for_videos` + `batch_takeaways_for_videos` (panels first, so takeaways integrate them). The discount is per-request, so even a single new video benefits; fanning out across a poll just saves round-trips. `_batch_capable()` gates the whole thing on the API backend being active — PTY / claude-code have no batch endpoint and keep generating inline in the subprocess. Any video missing from a batch result (per-item error, or the whole batch timing out) falls back to a synchronous `build_panel_for_video` / `build_takeaway_for_video`, so a video is never left without its panel. The panel/takeaway prompts are the single source of truth in `build_panel_request` / `build_takeaway_request`, shared by the sync and batch paths. **Interactive paths stay synchronous** — the web "Generate panel/takeaway" buttons and the one-off digest run on the direct API, since a human is waiting. `AnthropicAPIBackend.text_batch()` submits the batch, polls to `ended` (default 2h cap, then sync-fallback), and returns `{custom_id: (text, usage)}` keyed by `custom_id` (results arrive unordered). Known gap: a manual `yt2md watch run` and `serve`'s in-process scheduler don't share a cross-process lock, so running both at once races on `state.json` — run one at a time.

**Data layout (everything under `~/yt2md/`, override with `YT2MD_DATA`):**
- `.env` — `ANTHROPIC_API_KEY` (mode 0600). Optional `ANTHROPIC_ADMIN_KEY` (org Admin key) unlocks the budget gate's authoritative billing + price calibration. `.env` is also read from the repo CWD and `os.environ` via `load_env_files()`.
- `settings.json` — model choices, language, cookies-from-browser, `llm_backend`, and the budget keys (`budget_warn_usd`, `budget_block_usd`, `budget_workspace_id`, `model_pricing`).
- `pricing_cache.json` — price table calibrated from real billing by `calibrate_pricing_from_billing()` (written by `refresh-pricing` / opportunistically on `serve` start). Primary source for `_model_pricing`; absent → falls back to the hardcoded `DEFAULT_MODEL_PRICING`.
- `channels.txt` — subscribed channels (one URL per line).
- `library.db` — SQLite. Tables include `digest_reads` (read state), `digest_topics` (LLM + user tags), `digest_meta` (saved/dismissed/source), `runs` (per-pipeline cost + status history). Created lazily on first write.
- `digests/<video-id>/` — `digest.md`, `panel.md`, `takeaway.md`, `slides.pptx`, `digest_images/`, `downloads/` (cached source video + transcript; safe to delete to reclaim space).
- `logs/` — job stdout/stderr + the JSONL LLM usage log used by the Activity / cost view.

**Cost tracking.** Every LLM call goes through `record_llm_usage()`, which writes a JSONL line AND a `runs`-table row. `estimate_cost_usd` uses `_model_pricing`. The Activity page reads both. Calls made through the Message Batches API pass `record_llm_usage(batch=True)`, which halves the estimate (batches bill at 50%; `_model_pricing` only knows full rates) and stamps `batch:true` on the record — without this the Activity view would overstate batched calls 2×.

**Pricing resolution (don't just edit the hardcoded table).** `_model_pricing(model)` resolves in order: `pricing_cache.json` (billing-calibrated, merged over the hardcoded fields) → `settings["model_pricing"][model]` override → `DEFAULT_MODEL_PRICING` (fallback). The hardcoded table is now a *fallback + sanity-bound*, not the source of truth — the real rates come from `calibrate_pricing_from_billing()`, which derives effective $/Mtok from the Admin Usage + Cost reports (rejecting any line that diverges >3× from the fallback). So when Anthropic changes prices, run `yt2md refresh-pricing` rather than hand-editing the table. The hardcoded `DEFAULT_MODEL_PRICING` must still be kept roughly correct because it's the no-admin-key fallback and the divergence bound. (History: the table once carried Opus 4.7 at the retired $15/$75 tier vs the real $5/$25, overstating panel cost 3×.)

**Cost governance (Admin API).** A thin Admin-API client — `_admin_api_key()` (resolves the org `ANTHROPIC_ADMIN_KEY` from env/repo-`.env`/`~/yt2md/.env`), `_admin_get()`, `_admin_get_pages()` — backs two features; both **degrade gracefully to no-ops without an admin key**, so non-org users are unaffected:
- *Self-calibrating pricing* (above): `calibrate_pricing_from_billing()` + `fetch_usage_by_model()` / `fetch_cost_by_description()`, joined by `_norm_model()`.
- *Budget gate*: `check_budget()` refuses to **start** a new digest once month-to-date workspace spend ≥ `budget_block_usd` (warns at `budget_warn_usd`); wired at the top of `digest_video()` (returns `{status:"blocked"}`) and in `main()`'s digest path (`sys.exit`), so CLI/MCP/web/scheduler are all gated; never mid-pipeline. `workspace_month_to_date_cost()` reads authoritative spend from the Cost API (5-min cache) and falls back to summing the local usage log. `budget_status()` auto-detects + persists `budget_workspace_id` by matching the runtime key to a workspace. These are *soft, app-side* controls — the hard backstop is a Console spend cap, which the Admin API can read but **cannot set** (Console-only). See `docs/spikes/2026-05-30-ai-cost-governance.md` for the full three-layer model (in-app / gateway / provider-native) and why a real-time hard cutoff needs a gateway (LiteLLM), not the Admin API.

**Prompt caching is for reuse, not one-shot calls.** Cache-write bills at 1.25× input; a block written but never read is a net 25% loss. The single-shot digest/panel/takeaway calls therefore pass `cache=False` (do NOT add `cache=True` back without a reader in the same 5-min window). The backend's `cache_control` plumbing is left intact for a future shared-prefix cache.

## Things that bite

- **Don't break syntax.** The PostToolUse hook will block your next action if `youtube_to_markdown.py` is unparseable after an edit. Fix it before doing anything else.
- **Two surfaces, one library.** The web (`cmd_serve`), the CLI (`list/read/search/digest`), and the MCP server (`cmd_mcp`) all call the *same* underlying library functions. When you add a capability, wire it through the shared function — don't duplicate logic per surface.
- **Backend-agnostic LLM calls.** Accept an optional `backend` parameter, default to `select_backend()`. Don't hardcode `anthropic.Anthropic(...)`.
- **PTY backend is TUI-coupled.** `ClaudeCodePtyBackend` parses Claude Code's REPL output via `pyte` and looks for specific markers (`⏺` for response start, `✻ <verb> for Ns` for completion). Each Claude Code upgrade is a potential break — re-run a smoke call after bumping `claude`. Pin a known-good version in any deployment.
- **Frame extraction is one ffmpeg call by design.** The single-decode `extract_scene_and_interval_frames` is a performance fix; reverting to two subprocesses ~doubles pipeline time on slide-heavy talks.
- **The panel is the cost.** The Opus panel call is the single biggest line (~$0.20 of ~$0.46/video at corrected pricing — ~40%). Be conservative about adding Opus calls; swapping the panel to Sonnet is the main lever. The background subscription path (`watch run` / scheduler) also runs the panel + takeaway through the Message Batches API at 50% off — see *Batched background generation* under Architecture — but that only helps where nobody's waiting; interactive panel generation stays full-price.
- **Budget gate / pricing are admin-key-gated and fail open.** `_admin_*` helpers return `None` without `ANTHROPIC_ADMIN_KEY`, and `check_budget()` allows the action when spend is unknown — by design (non-org users unaffected). Don't make the pipeline hard-depend on them.
- **Windows is unverified.** Code paths are platform-aware but no one has run end-to-end. macOS is the supported path; Linux mostly works.
