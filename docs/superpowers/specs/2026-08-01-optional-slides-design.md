# Optional slides pipeline — design spec

**Date:** 2026-08-01  
**Status:** approved

## Problem

Slides are on by default. Every digest run pays the cost of:
- ffmpeg frame extraction + pHash dedup (wall time, no LLM tokens)
- `classify_slides_via_grids` vision LLM batches (Haiku, ~$0.04/video)
- `build_deck` pptx generation (no LLM cost)

For most text-heavy/talk-style videos the deck artifact is not the primary value. Audio generation is already off by default (on-demand only).

## Goal

Make slides opt-in. Skip the entire frame extraction + classification block when slides are not requested, saving wall time and the vision classification cost.

## Non-goals

- Audio pipeline changes (already off by default)
- Settings-based default (overkill for personal tool)
- Web UI toggle (slides are generated on-demand via the "Generate Slides" button)

## Design

### 1. CLI flag

Replace `--no-slides` with `--slides` (opt-in).

```
--slides    Build slides.pptx alongside the digest. Off by default.
            Runs frame extraction, pHash dedup, vision classification,
            and deck generation.
```

`deck_path` logic changes from:
```python
elif args.deck_only or not args.no_slides:
```
to:
```python
elif args.deck_only or args.slides:
```

`--no-slide-classification`, `--slide-classifier-model`, `--deck`, and `--deck-only` are unchanged and compose with `--slides` as before.

### 2. Frame extraction gate

Frame extraction is only needed when:
- slides are requested (`deck_path is not None`), OR
- vision picking is enabled and the backend supports vision

```python
vision_capable = getattr(select_backend(for_vision=True), "vision_supported", False)
need_frames = deck_path is not None or (not args.no_vision and vision_capable)

if need_frames:
    # extract_scene_and_interval_frames, dedupe_frames (existing code)
else:
    frames = []
    print("[1-2/5] Frame extraction skipped (no --slides, no vision)")
```

`vision_pick_frames` already falls back to timestamp-based picks when `frames` is empty — no changes needed there. `assign_transcript_to_frames` handles an empty list (returns empty slides data, which is unused when `deck_path is None`).

### 3. Downstream impact

- `digest_video()` spawns `yt2md <url>` as a subprocess with no `--slides` flag — correct, it gets the new off-by-default behavior automatically.
- The watch scheduler uses `digest_video()` — also correct.
- The web "Generate Slides" button calls `build_slides_for_video()` independently — unaffected.
- `--deck-only` still forces `deck_path` on regardless of `--slides`.

## Token savings

| Step | Tokens saved when slides off |
|---|---|
| `classify_slides_via_grids` | ~Haiku vision batches (~$0.04/video) |
| `vision_pick_frames` | 0 (still runs; timestamp fallback if frames=[]) |
| `build_deck` | 0 (no LLM) |
| Frame extraction | 0 (ffmpeg, but saves 30–60s wall time) |

## Migration

`--no-slides` is removed. Anyone using it gets an argparse error on the next run — intentional, since the flag is now meaningless (slides are off by default).
