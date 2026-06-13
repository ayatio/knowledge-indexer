# CLAUDE.md, Knowledge Indexer (PoC)

## Context
We are building a local agent that reads screenshots (tutorials, tools, how to guides) and files each one as a Markdown note in an Obsidian style vault (just a folder of `.md` files). This is the PoC, Phase 1 only. The goal is to prove the core loop on a small sample, so the output is meant to be inspected and tuned, not treated as final. Over extraction is fine for now, we trim later. The extraction prompt is the main tuning knob and must live in ONE place so it is easy to change.

## Paths (never write outside these)
- Input images: `/Users/ayat/repos/knowledge-indexer/poc/images`
- Output vault (PoC, isolated): `/Users/ayat/repos/knowledge-indexer/poc/vault`  (create if missing)
- State, dedupe index, run logs: `/Users/ayat/repos/knowledge-indexer/poc/.state`
- OFF LIMITS: `/Users/ayat/repos/second-brain`. That is the real vault. The PoC must never read or write there.

## Stack
- Python 3, conda base is fine. Keep dependencies minimal: `requests` (or `httpx`), `pyyaml`, `python-dotenv`, `Pillow` optional.
- API key from env var `GEMINI_API_KEY`, also loadable from a `poc/.env`. Never hardcode the key, never print it, never write it to the vault, logs, or repo. Add a `.gitignore` covering `.env`, `.state/`, and `__pycache__/`.

## Model call
- Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- Auth header: `x-goog-api-key: $GEMINI_API_KEY` (confirmed working on this account).
- Send the image inline: a part with `inline_data` containing `mime_type` and `data` (base64 of the file), plus a second part with the text prompt.
- Disable thinking to save tokens and latency: `generationConfig.thinkingConfig.thinkingBudget = 0`. A plain OCR read spent 2000+ thinking tokens in testing, which is wasteful. If the API rejects that field, omit it and continue.
- Response text is at `candidates[0].content.parts[0].text`. It may arrive wrapped in a ```json fence, strip the fence before parsing. Parse defensively: on JSON parse failure, store the raw text in the note body and set `status: needs_review`.

## Extraction prompt (the tuning knob, keep as one constant)
Instruct the model to return ONLY JSON, no prose and no fences, with exactly these fields:
- `title`: short title of the screenshot
- `type`: one of `tutorial`, `tool`, `step`, `UI`, `other`
- `text`: all visible text, verbatim
- `tags`: array of short topic tags
- `tools`: array of tools, apps, or libraries mentioned (may be empty)
- `confidence`: number 0 to 1, the model's own rating of how complete and legible the screenshot is

Keep it lean. We expect to trim fields and reword this after reviewing the first run.

## Note shape (one `.md` per image)
```
---
title: "<title>"
source: photos_local
source_ref: "<original filename>"
image_hash: "<sha256 of the file bytes>"
created: <YYYY-MM-DD>
status: complete | needs_review
confidence: <0..1>
tags: [..]
tools: [..]
enriched_sources: []
---
# <title>

## Extracted
<verbatim text>

## Notes
<optional one or two line summary>
```
- Filename: a slug of the title plus a short hash suffix, eg `build-a-real-time-ai-voice-translator-a1b2c3.md`. Stable and collision safe.
- `status`: `complete` if `confidence >= 0.5` (threshold, tunable in one place), otherwise `needs_review`. Partial or unreadable shots must be `needs_review`, never silently stored as complete.

## Run behaviour
- Process every image in the input folder, but accept a `--limit N` flag, default 20. (There are about 43 images, the free tier is roughly 500 a day and about 10 requests a minute.)
- Rate limit: pace requests to stay under about 10 a minute, and back off on HTTP 429 rather than crashing.
- Dedupe by `image_hash`: if a note for that hash already exists, skip it. Re running must add nothing (idempotent).
- Never delete, move, or modify source images.
- Print an end of run summary: total seen, written, skipped as duplicate, needs_review, errors. Write the same summary plus per item state to a timestamped log in `.state`.

## Acceptance criteria (build and test against these)
1. A run on the images folder produces one `.md` note per new image, in the PoC vault, never in `second-brain`.
2. Every note has frontmatter with `title`, `source_ref`, `image_hash`, `created`, `status`, and at least one tag.
3. Re running the agent adds zero new notes (dedupe by hash holds).
4. A partial or unreadable screenshot is marked `needs_review`, not `complete`.
5. The key is never written to the vault, logs, or repo, and is never printed.
6. The agent stays within the rate limit and does not crash on a 429, it backs off and continues.
7. `--limit` caps the number of images processed in a run.
8. Source images are never modified, moved, or deleted.
9. The summary counts match what is actually on disk.

## Out of scope (later phases, do not build now)
iCloud ingestion, enrichment of partial notes via web or YouTube, grouping multiple screenshots that belong to the same tutorial, any deletion or cleanup, scheduling, and in chat retrieval over the vault.

## Dev loop
Define, design, build, test, review, gate. Write tests derived from the acceptance criteria above, especially the negative ones: no writes to `second-brain`, no key in any output, dedupe adds nothing on a second run. Run the tests on a small labelled handful before the full sample.
