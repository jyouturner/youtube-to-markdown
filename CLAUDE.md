# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Run / dev / test

- **Bootstrap + launch** (preferred during development): `./run.sh` — checks ffmpeg, installs uv if missing, runs `uv sync`, prompts for `ANTHROPIC_API_KEY` if not in `~/yt2md/.env`, runs `yt2md doctor`, then `yt2md serve`. Idempotent; safe to re-run.
- **Direct CLI** (after `uv sync`): `uv run yt2md <subcommand>` — e.g. `serve`, `digest <url>`, `list`, `read <id>`, `search <q>`, `watch {add,list,remove,run}`, `mcp`, `doctor`, `topics`, `retrofit-topics`, `project-instructions`, `install-skills`.
- **Default web port**: 7682. `--host 0.0.0.0` exposes the library on the LAN (used by the `/listen` QR-code phone flow); default `127.0.0.1` is localhost-only.
- **Wire into Claude Desktop / Cowork**: add to `~/Library/Application Support/Claude/claude_desktop_config.json`: `{"mcpServers": {"yt2md": {"command": "yt2md", "args": ["mcp"]}}}`, then `uv run yt2md install-skills` to write `~/.claude/skills/yt2md-digest-video/SKILL.md`, then restart Desktop. Once both are in place, the user just asks Cowork "digest https://youtu.be/..." and the skill takes over orchestration: `start_video_prep(url)` → poll `job_status_prep(video_id)` → `get_playbook("digest")` → read `get_transcript`, reason → `write_digest` → `get_slide_classifier_grids` → reason → `build_video_deck` → `write_panel` / `write_takeaway`. yt2md does zero LLM in this flow.
- **Slide regression check** (the only "test" — single curated fixture): regenerate the deck for the fixture video via the web reader, then `uv run python tests/compare_slides.py tests/fixtures/igO8iyca2_g.pptx ~/yt2md/digests/igO8iyca2_g/slides.pptx`. Run after touching `extract_*_frames`, `dedupe_frames`, `global_phash_cluster`, `_render_classification_grids`, `classify_slides_via_grids`, `assign_transcript_to_frames`, or `build_deck`. LLM noise — re-run once before treating a failure as a regression.

There is no lint or unit-test suite. The PostToolUse hook in `.claude/settings.json` `ast.parse`s any edited `.py` and blocks (exit 2) on a SyntaxError — so an Edit/Write that leaves `youtube_to_markdown.py` unparseable will fail immediately.

## Architecture

**Single-file design.** Everything ships from `youtube_to_markdown.py` (~10k lines). `pyproject.toml` is configured as `only-include = ["youtube_to_markdown.py"]` — do not split the module without updating the wheel target. The two `[project.scripts]` entry points (`yt2md`, `youtube-to-markdown`) both call `main()` in that file.

**One CLI, many subcommands, dispatched in `main()`:**
- Default (no subcommand): single-video digest flow — `yt2md <url-or-mp4> [srt]` runs the whole pipeline once and writes markdown next to the input.
- `serve` — Flask web reader on `127.0.0.1:7682`. Also runs the in-process scheduler that polls subscriptions. This is the primary UX; most routes live inside `cmd_serve`.
- `watch {add,list,remove,run}` — manage channel subscriptions in `~/yt2md/channels.txt`; `watch run` polls all channels once.
- `mcp` — start the MCP stdio server. The same library surface (`list_digests`, `read_digest`, `search_library`, `digest_video`, `generate_panel/takeaway/slides`, `job_status`, `*_subscription`) is exposed as `@mcp.tool()` AND as shell subcommands (`list`, `read`, `search`, `digest`, `topics`, `retrofit-topics`) — keep the two surfaces in sync when you add a capability.
- `doctor` — preflight check (Node 20+ for yt-dlp, ffmpeg, cookies, API key).

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

**Cowork-as-runtime (Phase 1 — additive, not yet replacing the backends).** A second MCP surface lets Claude Desktop's Cowork drive the pipeline as the agent, with yt2md doing only compute. yt2md makes NO LLM calls in this flow. The split:
- `start_video_prep(url)` spawns a background job (fetch + frames + dedupe + render slide-classifier grids), returns `{video_id, status}` immediately. Writes `digests/<id>/prep_state.json` with all paths the agent needs.
- `job_status_prep(video_id)` for polling — Cowork's 4-minute MCP tool timeout means EVERY long-running stage must be async + poll.
- `get_transcript`, `get_video_meta`, `get_slide_classifier_grids` (returns FastMCP `Image` blocks the model can see), `get_topic_candidate_frames(video_id, start_sec, end_sec)` — read-side tools the agent uses to gather inputs.
- `write_digest`, `write_panel`, `write_takeaway`, `build_video_deck` — write-side tools that persist agent-generated artifacts to the same paths the web reader serves.
- `get_playbook(stage)` returns the existing `DIGEST_SYSTEM_PROMPT` / `PANEL_SYSTEM_PROMPT` / `TAKEAWAY_SYSTEM_PROMPT` plus an output-format spec so the agent's output matches what `write_*` expects.

Worker is `_do_video_prep(url, ...)` running via `start_local_job(f"{video_id}:prep", ...)`. State persistence is a single `prep_state.json` per video — paths only, no LLM output. The classic three-backend flow (API/Code/PTY) remains intact; Cowork is the third user-facing surface alongside web + CLI. End-state plan: when Cowork integration is proven, the legacy backends + `select_backend()` + `~/yt2md/.env`/cost-tracking are deleted in a follow-up phase.

**Phase 2 — bundled skill + install command.** `_BUNDLED_SKILLS` (module-level dict) holds the markdown body for `yt2md-digest-video`, a Cowork playbook that tells the agent how to orchestrate the Phase-1 toolkit end-to-end. `yt2md install-skills` writes it to `~/.claude/skills/yt2md-digest-video/SKILL.md`; the file follows the [agentskills.io](https://agentskills.io) standard (frontmatter: `name`, `description`, `allowed-tools`; markdown body for instructions). The same dir is read by both Cowork (Claude Desktop) and Claude Code, so this skill works in both surfaces. To update the skill, edit `_BUNDLED_SKILLS["yt2md-digest-video"]` in the Python source and re-run `install-skills` — there's no separate skills file in the repo, the Python source is the single source of truth.

Constraints worth remembering (load-bearing for any change):
- Cowork tool-call timeout: hard 4-min cap, flaky past 60s. Anything compute-heavy MUST be start-job + poll. The `:prep` job pattern is the template.
- Cowork tool-result size: ~500 KB ceiling per result. Image blocks must be downscaled (slide-classifier grids are 1200×675 JPEG q85 ≈ 150-200 KB — safe; vision-pick candidates use `_encode_frame_for_vision`'s 1024-long-edge default).
- Cowork is NOT headless. Subscriptions/scheduler can't run inside Cowork. Local scheduler (`_scheduler_loop` in `cmd_serve`) keeps using the API/PTY backends until we migrate to Desktop Scheduled Tasks invoking a Cowork playbook (Phase 3).

**Data layout (everything under `~/yt2md/`, override with `YT2MD_DATA`):**
- `.env` — `ANTHROPIC_API_KEY` (mode 0600).
- `settings.json` — model choices, language, cookies-from-browser, `llm_backend`.
- `channels.txt` — subscribed channels (one URL per line).
- `library.db` — SQLite. Tables include `digest_reads` (read state), `digest_topics` (LLM + user tags), `digest_meta` (saved/dismissed/source), `runs` (per-pipeline cost + status history). Created lazily on first write.
- `digests/<video-id>/` — `digest.md`, `panel.md`, `takeaway.md`, `slides.pptx`, `digest_images/`, `downloads/` (cached source video + transcript; safe to delete to reclaim space).
- `logs/` — job stdout/stderr + the JSONL LLM usage log used by the Activity / cost view.

**Cost tracking.** Every LLM call goes through `record_llm_usage()`, which writes a JSONL line AND a `runs`-table row. `estimate_cost_usd` uses `_model_pricing` — update that table when adding a model. The Activity page reads both.

## Things that bite

- **Don't break syntax.** The PostToolUse hook will block your next action if `youtube_to_markdown.py` is unparseable after an edit. Fix it before doing anything else.
- **Two surfaces, one library.** The web (`cmd_serve`), the CLI (`list/read/search/digest`), and the MCP server (`cmd_mcp`) all call the *same* underlying library functions. When you add a capability, wire it through the shared function — don't duplicate logic per surface.
- **Backend-agnostic LLM calls.** Accept an optional `backend` parameter, default to `select_backend()`. Don't hardcode `anthropic.Anthropic(...)`.
- **PTY backend is TUI-coupled.** `ClaudeCodePtyBackend` parses Claude Code's REPL output via `pyte` and looks for specific markers (`⏺` for response start, `✻ <verb> for Ns` for completion). Each Claude Code upgrade is a potential break — re-run a smoke call after bumping `claude`. Pin a known-good version in any deployment.
- **Frame extraction is one ffmpeg call by design.** The single-decode `extract_scene_and_interval_frames` is a performance fix; reverting to two subprocesses ~doubles pipeline time on slide-heavy talks.
- **The panel is the cost.** Opus calls dominate (~70%+ of per-video spend). Be conservative about adding more.
- **Windows is unverified.** Code paths are platform-aware but no one has run end-to-end. macOS is the supported path; Linux mostly works.
