---
name: slimsnap
description: Use this skill when the user references something visual on their screen, a layout, a UI element, a design, a broken form, a button, a page, or says things like "fix this", "what's on screen", "the page I'm on", "see what I'm looking at". Reads the user's latest SlimSnap JSON capture, which is a structured description of an annotated screenshot containing bounding boxes, extracted colors, and OCR text. About 700 tokens of JSON vs the API per-image cap of 1,568 tokens on Sonnet/Haiku (up to 4,784 on Opus 4.7+ per Anthropic's vision docs), and the agent acts on coordinates more reliably than re-interpreting pixels. The skill discovers where SlimSnap saves files by reading ~/.slimsnap/config.json (no hardcoded folder).
---

# slimsnap

SlimSnap (https://slimsnap.ai) is a Mac app that converts a screenshot into structured JSON: bounding boxes, extracted colors, OCR text, and user annotations (arrows, rectangles, highlights, callouts) pointing at what matters. This skill lets you read those captures directly so the user does not have to copy and paste.

## Where to find captures

SlimSnap publishes its current save folder in a small config file the user does not manage. Read it on every invocation so the skill follows whatever the user has configured:

1. Read `~/.slimsnap/config.json`. Shape:
   ```json
   {
     "schema_version": "1.0",
     "default_save_folder": "/Users/<name>/Desktop",
     "filename_pattern": "SlimSnap {date} {time}"
   }
   ```
   Use `default_save_folder` as the folder to search.

2. Also check `<project root>/.slimsnap/` if it exists. This is an optional per-project location for users who want captures kept with the codebase. Prefer it over the config-named folder when present.

3. If `~/.slimsnap/config.json` does not exist (SlimSnap was never launched, or the user manually removed it), tell the user to launch SlimSnap once so it can publish its config, and stop. Do not guess at a folder.

Within the chosen folder, pick the most recently modified `.json` file. If the user passes `$ARGUMENTS` with a filename or partial match, use that file instead of the latest.

## How to read a capture

Each SlimSnap JSON conforms to the open MIT schema at https://github.com/bickov/slimsnap-schema. Current version is **v2.0** (`frames[]`); older installs still emit v1.0. Branch on the presence of a top-level `frames` array.

### Schema v2 (current)

- `mode`: `"single"` (a regular screenshot, one frame) or `"scroll"` (a scrollable capture — a whole page, long chat, or feed — split into frames stacked top to bottom).
- `capture`: `{ width_px, height_px }` of the WHOLE capture, all frames stacked.
- `screen` (optional): `{ title, app, url }`. Context about what was captured (browser tab title, app name, URL). Use it to know what kind of code to look for.
- `frames`: the content. Each frame has:
  - `index` (0 = top), `kind` (`"spatial"`), `dims` (`{ width_px, height_px }` of THIS frame), `position` (`{ scroll_y_px }` — the frame's top edge inside the whole capture).
  - `elements`: detected UI elements. Each has `id`, `type` (one of `text`, `button`, `input`, `link`, `image`, `label`, `unknown`), `value` (the OCR text or content), `bbox`, and optional `color` (hex like `#3B82F6`).
  - `annotations`: user-drawn markers. Each has `id`, `type` (`arrow`, `rectangle`, `highlight`, `callout`, `note`), `color`, an optional `intent` (`highlight`, `explain`, `action`, `question`), and geometry depending on type: `from`/`to` for arrows, `bbox` for rectangles and callouts, `position` for point-based notes. Callouts also carry `text`. An annotation may have `target_ref` pointing at an element's `id`.
  - `images` (optional): `[{ file, offset_y_px, height_px }]` — present only when the frame's image was saved to disk alongside the JSON. Saved scroll captures include one PNG per frame in the SAME folder as the JSON; if you need visual confirmation of something the JSON describes, open that frame's file rather than asking the user for a screenshot.
- `estimated_tokens`: approximate token count of this JSON.

**Important: `bbox` is `[x, y, width, height]` normalized to 0-1 relative to the FRAME (`frames[].dims`), not pixels and not the whole capture.** Absolute pixel position of anything: `y_px = position.scroll_y_px + bbox_y * dims.height_px`; `x_px = bbox_x * dims.width_px`. Same for `point` (`[x, y]` normalized 0-1). Element/annotation `id`s are unique across ALL frames, so a `target_ref` near a frame boundary may point into the adjacent frame — search all frames when following one.

For `mode: "scroll"`, treat frame order as reading order: frame 0 is the top of the page. Vertical position within the whole capture comes from `position.scroll_y_px`.

### Schema v1 (older installs)

No `frames` array. Top-level `image` (`{ width_px, height_px, file }`), `elements`, `annotations` — same element/annotation shapes as v2, with all coordinates normalized against `image` dims. Read it as v2 with one implicit frame.

Treat annotations as the user's intent:
- The `intent` field, when present, is the most reliable signal: `highlight` means "look here", `explain` means "the callout text explains what's going on", `action` means "do this", `question` means "I'm asking about this."
- The `text` on a callout is the user's verbal comment. Treat it as part of the prompt.
- The `target_ref` on an annotation links it to a specific element's `id`. Follow it to find what is being marked.
- An arrow uses `from` (the user's hand-drawn start) and `to` (what it points at).
- A rectangle or callout uses `bbox` to mark a region.
- Color is the user's free choice and not semantically fixed. Do not assume "red equals broken." Read `intent` and `text` instead.

## When to act

If the capture is recent (modified within the last few minutes) and the user is asking about something visual, prefer using the capture as primary context rather than asking the user to describe what they see. The capture IS the description.

If no recent capture exists in the configured folder, ask the user to capture a screenshot with SlimSnap (`⌘⇧S` by default), annotate it, and save the JSON, then retry.

## Worked example

User says: "fix this broken sign-up form"

1. Read `~/.slimsnap/config.json` to get `default_save_folder`.
2. List `.json` files in that folder, pick the most recently modified.
3. Check `screen.app`, `screen.url`, and `screen.title` for context so you know what kind of code to look for (React component, HTML page, native view, etc.).
4. Walk `frames[]` in order (v1: treat the whole document as one frame). From each frame's `elements`, identify form fields, buttons, labels by `type` and `value`. Their `bbox` (normalized 0-1 within the frame) tells you layout position.
5. From `annotations` across all frames, find what the user marked. When `target_ref` is present, follow it to the exact element being annotated (search every frame for the id). Use `intent` to interpret the marker: a callout with `intent: "explain"` and `text: "Duplicate Pay button"` is unambiguous, the user is telling you what's wrong.
6. Locate the corresponding source files in the project and propose the fix that addresses the annotated issues specifically.

The agent's edits should be grounded in what the JSON says is wrong, not in a guess about what the user might mean.

## Related

- App: https://slimsnap.ai
- JSON Schema: https://github.com/bickov/slimsnap-schema
- This skill: https://github.com/bickov/slimsnap-skill
