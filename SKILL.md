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

Each SlimSnap JSON conforms to the open MIT schema at https://github.com/bickov/slimsnap-schema. The fields you care about:

- `elements`: array of detected UI elements. Each has a `bbox` (x, y, width, height in pixels), a dominant `color` (hex), and `text` (OCR result, may be empty).
- `annotations`: user-added markers, each with `type` (arrow, rectangle, highlight, callout), normalized `points` (0 to 1 floats relative to image size), `color`, and optional `text` for callouts.
- `image_size`: `{ width, height }` in pixels of the original screenshot.
- `token_estimate`: approximate token count of this JSON.

Treat annotations as the user's intent:
- An arrow points at the element the user is asking about.
- A red rectangle usually means "this is broken" or "this is what's wrong."
- A callout's `text` is the user's verbal comment, treat it as part of the prompt.
- A highlight marks a region of interest.

## When to act

If the capture is recent (modified within the last few minutes) and the user is asking about something visual, prefer using the capture as primary context rather than asking the user to describe what they see. The capture IS the description.

If no recent capture exists in either folder, ask the user to capture a screenshot with SlimSnap (`⌘⇧S` by default), annotate it, and save the JSON, then retry.

## Worked example

User says: "fix this broken sign-up form"

1. Read the latest JSON from `.slimsnap/` (or `~/Documents/SlimSnap/`).
2. From `elements`, identify form fields, buttons, labels. Their `bbox` coordinates tell you the layout.
3. From `annotations`, find what the user marked: an arrow pointing at the misaligned button, a red rectangle around the email field with overflowing text, a callout that says "this is cut off."
4. Locate the corresponding source files in the project (HTML, JSX, CSS, etc.) and propose the fix that addresses the annotated issues specifically.

The agent's edits should be grounded in what the JSON says is wrong, not in a guess about what the user might mean.

## Related

- App: https://slimsnap.ai
- JSON Schema: https://github.com/bickov/slimsnap-schema
- This skill: https://github.com/bickov/slimsnap-skill
