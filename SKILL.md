---
name: slimsnap
description: Use this skill when the user references something visual on their screen, a layout, a UI element, a design, a broken form, a button, a page, or says things like "fix this", "what's on screen", "the page I'm on", "see what I'm looking at". Reads the latest SlimSnap JSON capture, which is a structured description of an annotated screenshot containing bounding boxes, extracted colors, and OCR text. About 700 tokens vs ~8k for a raw image, and the agent acts on coordinates more reliably than re-interpreting pixels.
---

# slimsnap

SlimSnap (https://slimsnap.ai) is a Mac app that converts a screenshot into structured JSON: bounding boxes, extracted colors, OCR text, and user annotations (arrows, rectangles, highlights, callouts) pointing at what matters. This skill lets you read those captures directly so the user does not have to copy and paste.

## Where to find captures

Check these locations in order:

1. `<project root>/.slimsnap/` if it exists (per-project, opt-in)
2. `~/Documents/SlimSnap/` (user-global, the default)

Pick the most recently modified `.json` file in the chosen folder. Files are named with timestamps like `2026-05-28-14-32-07.json`.

If the user passes `$ARGUMENTS` with a filename or partial match, use that file instead of the latest.

## How to read a capture

Each SlimSnap JSON conforms to the open MIT schema at https://github.com/bickov/slimsnap-schema (v1.0). The fields you care about:

- `image`: `{ width_px, height_px, file }`. Pixel dimensions of the original screenshot.
- `screen` (optional): `{ title, app, url }`. Context about what was captured (browser tab title, app name, URL). Use it to know what kind of code to look for.
- `elements`: array of detected UI elements. Each has `id`, `type` (one of `text`, `button`, `input`, `link`, `image`, `label`, `unknown`), `value` (the OCR text or content), `bbox`, and optional `color` (hex like `#3B82F6`).
- `annotations`: user-drawn markers. Each has `id`, `type` (`arrow`, `rectangle`, `highlight`, `callout`, `note`), `color`, an optional `intent` (`highlight`, `explain`, `action`, `question`), and geometry depending on type: `from`/`to` for arrows, `bbox` for rectangles and callouts, `position` for point-based notes. Callouts also carry `text`. An annotation may have `target_ref` pointing at an element's `id`, explicitly linking the annotation to a specific element.
- `estimated_tokens`: approximate token count of this JSON.

**Important: `bbox` is `[x, y, width, height]` normalized to 0-1 relative to the image, not pixels.** Multiply by `image.width_px` / `image.height_px` if you need pixel values. Same for `point` (`[x, y]` normalized 0-1).

Treat annotations as the user's intent:
- The `intent` field, when present, is the most reliable signal: `highlight` means "look here", `explain` means "the callout text explains what's going on", `action` means "do this", `question` means "I'm asking about this."
- The `text` on a callout is the user's verbal comment. Treat it as part of the prompt.
- The `target_ref` on an annotation links it to a specific element's `id`. Follow it to find what is being marked.
- An arrow uses `from` (the user's hand-drawn start) and `to` (what it points at).
- A rectangle or callout uses `bbox` to mark a region.
- Color is the user's free choice and not semantically fixed. Do not assume "red equals broken." Read `intent` and `text` instead.

## When to act

If the capture is recent (modified within the last few minutes) and the user is asking about something visual, prefer using the capture as primary context rather than asking the user to describe what they see. The capture IS the description.

If no recent capture exists in either folder, ask the user to capture a screenshot with SlimSnap (`⌘⇧S` by default), annotate it, and save the JSON, then retry.

## Worked example

User says: "fix this broken sign-up form"

1. Read the latest JSON from `.slimsnap/` (or `~/Documents/SlimSnap/`).
2. Check `screen.app`, `screen.url`, and `screen.title` for context so you know what kind of code to look for (React component, HTML page, native view, etc.).
3. From `elements`, identify form fields, buttons, labels by `type` and `value`. Their `bbox` (normalized 0-1) tells you layout position relative to the image.
4. From `annotations`, find what the user marked. When `target_ref` is present, follow it to the exact element being annotated. Use `intent` to interpret the marker: a callout with `intent: "explain"` and `text: "Duplicate Pay button"` is unambiguous, the user is telling you what's wrong.
5. Locate the corresponding source files in the project and propose the fix that addresses the annotated issues specifically.

The agent's edits should be grounded in what the JSON says is wrong, not in a guess about what the user might mean.

## Related

- App: https://slimsnap.ai
- JSON Schema: https://github.com/bickov/slimsnap-schema
- This skill: https://github.com/bickov/slimsnap-skill
