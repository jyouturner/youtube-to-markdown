# Optional Slides Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make slides and frame extraction opt-in via `--slides` flag, skipping both by default to save wall time and vision-classification tokens.

**Architecture:** All changes are in `main()` inside `youtube_to_markdown.py`. Two surgical edits: (1) swap `--no-slides` for `--slides` in argparse and flip the `deck_path` condition; (2) gate frame extraction behind a `need_frames` boolean that is true only when slides are requested or vision picking is enabled.

**Tech Stack:** Python 3, argparse, existing ffmpeg + pHash pipeline.

## Global Constraints

- Single-file codebase: all changes in `youtube_to_markdown.py`
- No test suite; the PostToolUse hook auto-runs `ast.parse` on every `.py` edit — a SyntaxError blocks the next action
- No new dependencies
- `--deck-only` must keep working (forces slides on regardless of `--slides`)
- `--no-slide-classification`, `--slide-classifier-model`, `--deck`, `--deck-only` are unchanged

---

### Task 1: Replace `--no-slides` with `--slides` and fix `deck_path` logic

**Files:**
- Modify: `youtube_to_markdown.py:10721-10725` (argparse), `10885-10895` (deck_path)

**Interfaces:**
- Produces: `args.slides` (bool, True when user passes `--slides`)
- Removes: `args.no_slides`

- [ ] **Step 1: Replace the argparse argument**

At line 10721, replace:
```python
    ap.add_argument("--no-slides", action="store_true",
                    help="Skip the PowerPoint deck. By default a slides.pptx file is "
                         "written alongside the digest — the visual layer (intelligently-"
                         "selected frames + transcript snippets) is the tool's main "
                         "differentiator and worth shipping for every digest.")
```
with:
```python
    ap.add_argument("--slides", action="store_true",
                    help="Build slides.pptx alongside the digest. Off by default. "
                         "Runs frame extraction, pHash dedup, vision classification, "
                         "and deck generation.")
```

- [ ] **Step 2: Fix the deck_path comment and condition**

At lines 10885-10895, replace:
```python
    # Slides default-on: write `slides.pptx` next to digest.md so the
    # web reader (and any KB ingester) finds it at a predictable path.
    # --no-slides opts out; --deck path overrides; --deck-only forces it
    # on even when --no-slides was passed (the user explicitly asked
    # for the deck).
    if args.deck and args.deck != "__default__":
        deck_path: Optional[Path] = Path(args.deck)
    elif args.deck_only or not args.no_slides:
        deck_path = digest_path.parent / "slides.pptx"
    else:
        deck_path = None
```
with:
```python
    # Slides off by default. --slides opts in; --deck path overrides;
    # --deck-only forces it on even without --slides.
    if args.deck and args.deck != "__default__":
        deck_path: Optional[Path] = Path(args.deck)
    elif args.deck_only or args.slides:
        deck_path = digest_path.parent / "slides.pptx"
    else:
        deck_path = None
```

- [ ] **Step 3: Verify syntax is clean**

The PostToolUse hook runs `ast.parse` automatically after every Edit/Write — if it passes without blocking, syntax is good. Confirm by checking there's no hook error before proceeding.

- [ ] **Step 4: Smoke-test the flag**

```bash
uv run yt2md --help | grep -A3 "slides"
```
Expected output includes `--slides` and `--deck-only`; `--no-slides` must NOT appear.

- [ ] **Step 5: Commit**

```bash
git add youtube_to_markdown.py
git commit -m "feat: make slides opt-in via --slides (was --no-slides)"
```

---

### Task 2: Gate frame extraction behind need_frames

**Files:**
- Modify: `youtube_to_markdown.py:10897-10990` (frame extraction + slides block in main)

**Interfaces:**
- Consumes: `deck_path` (Optional[Path]) from Task 1, `args.no_vision` (bool)
- Produces: `frames` (List[Tuple[Path, float]]) — empty list when skipped

- [ ] **Step 1: Add need_frames check before frame extraction**

At line 10897 (just before `workdir = Path(tempfile.mkdtemp(prefix="v2d_"))`), insert:

```python
    # Frame extraction is needed only when building slides OR when vision
    # picking is enabled (to supply the frame pool for vision_pick_frames).
    # Compute vision capability now so we can decide before spawning ffmpeg.
    _vision_backend_for_gate = select_backend(for_vision=True)
    _vision_capable = getattr(_vision_backend_for_gate, "vision_supported", False)
    need_frames = deck_path is not None or (not args.no_vision and _vision_capable)
```

- [ ] **Step 2: Wrap the frame extraction block**

Lines 10908-10933 currently read (unconditional extraction):
```python
        print(f"[1/5] Extracting frames "
              f"(scene threshold={args.scene_threshold}, interval={args.interval}s, "
              f"single-pass)...")
        _frames_t0 = _time.monotonic()
        scene_frames, interval_frames = extract_scene_and_interval_frames(
            video_path, scene_dir, interval_dir,
            scene_threshold=args.scene_threshold,
            interval=args.interval, duration=duration,
        )
        timings["frames_extract"] = round(_time.monotonic() - _frames_t0, 3)
        frames = merge_frames(scene_frames, interval_frames)
        cap_note = (
            f" (capped from >={SCENE_FRAME_HARD_CAP})"
            if len(scene_frames) == SCENE_FRAME_HARD_CAP else ""
        )
        print(f"      {len(scene_frames)} scene{cap_note} + "
              f"{len(interval_frames)} interval = {len(frames)} candidate frames "
              f"({timings['frames_extract']}s)")

        print(f"[2/5] Deduping consecutive near-identical frames "
              f"(hash distance <= {args.hash_distance})...")
        _dedupe_t0 = _time.monotonic()
        frames = dedupe_frames(frames, args.hash_distance)
        timings["frames_dedupe"] = round(_time.monotonic() - _dedupe_t0, 3)
        print(f"      {len(frames)} unique frames ({timings['frames_dedupe']}s)")
        timings["frames"] = round(_time.monotonic() - _frames_t0, 3)
```

Replace with:
```python
        if need_frames:
            print(f"[1/5] Extracting frames "
                  f"(scene threshold={args.scene_threshold}, interval={args.interval}s, "
                  f"single-pass)...")
            _frames_t0 = _time.monotonic()
            scene_frames, interval_frames = extract_scene_and_interval_frames(
                video_path, scene_dir, interval_dir,
                scene_threshold=args.scene_threshold,
                interval=args.interval, duration=duration,
            )
            timings["frames_extract"] = round(_time.monotonic() - _frames_t0, 3)
            frames = merge_frames(scene_frames, interval_frames)
            cap_note = (
                f" (capped from >={SCENE_FRAME_HARD_CAP})"
                if len(scene_frames) == SCENE_FRAME_HARD_CAP else ""
            )
            print(f"      {len(scene_frames)} scene{cap_note} + "
                  f"{len(interval_frames)} interval = {len(frames)} candidate frames "
                  f"({timings['frames_extract']}s)")

            print(f"[2/5] Deduping consecutive near-identical frames "
                  f"(hash distance <= {args.hash_distance})...")
            _dedupe_t0 = _time.monotonic()
            frames = dedupe_frames(frames, args.hash_distance)
            timings["frames_dedupe"] = round(_time.monotonic() - _dedupe_t0, 3)
            print(f"      {len(frames)} unique frames ({timings['frames_dedupe']}s)")
            timings["frames"] = round(_time.monotonic() - _frames_t0, 3)
        else:
            frames = []
            print("[1-2/5] Frame extraction skipped (no --slides, no vision)")
```

- [ ] **Step 3: Clean up the slides-skipped else branch**

At line 10985-10988, the else branch currently reads:
```python
        else:
            print("[5/5] Slides skipped (--no-slides)")
            # Still need slides_data for downstream code if any consumes it,
            # though currently only build_deck does. Use the rich pool.
            slides_data = assign_transcript_to_frames(frames, segments, duration)
```

Replace with:
```python
        else:
            print("[5/5] Slides skipped (no --slides)")
```

(`slides_data` is unused when `deck_path is None`; `assign_transcript_to_frames` on an empty frame list is pointless.)

- [ ] **Step 4: Verify syntax is clean**

PostToolUse hook auto-validates. Confirm no block error.

- [ ] **Step 5: End-to-end smoke test without --slides**

```bash
# Use a short video or a cached download to avoid a full fetch.
# Verify: no "Extracting frames" line, no slides.pptx written.
uv run yt2md --help  # quick sanity
```

Then inspect the printed output of a real run (or review `logs/oneoff.log` after a web-triggered digest) and confirm:
- Line `[1-2/5] Frame extraction skipped` appears
- No `slides.pptx` is created in the digest directory
- Digest, panel, and takeaway still complete

- [ ] **Step 6: Smoke test with --slides**

```bash
uv run yt2md <url> --slides
```
Expected:
- `[1/5] Extracting frames` appears
- `[5/5] Building slides (N slides)` appears
- `slides.pptx` is written to the digest directory

- [ ] **Step 7: Commit**

```bash
git add youtube_to_markdown.py
git commit -m "feat: skip frame extraction when slides and vision both off"
```
