# slimsnap-skill

A [Claude Code](https://claude.com/claude-code) skill for [SlimSnap](https://slimsnap.ai). Lets Claude Code automatically read structured JSON captures of your screen, so you can say "fix this layout" instead of pasting a screenshot every turn.

About 700 tokens per capture instead of ~8k for a raw image, with structured bounding boxes, extracted colors, OCR text, and your annotations.

## Install

Globally, available in every project:

```bash
mkdir -p ~/.claude/skills/slimsnap
curl -L https://raw.githubusercontent.com/bickov/slimsnap-skill/main/SKILL.md \
  -o ~/.claude/skills/slimsnap/SKILL.md
```

Or scoped to a single project:

```bash
mkdir -p .claude/skills/slimsnap
curl -L https://raw.githubusercontent.com/bickov/slimsnap-skill/main/SKILL.md \
  -o .claude/skills/slimsnap/SKILL.md
```

## Use

1. Capture a screenshot with SlimSnap (`⌘⇧S`).
2. Annotate what matters: arrows, rectangles, callouts.
3. Hit "Save JSON" (or rely on the auto-save to `~/Documents/SlimSnap/`).
4. In Claude Code, say what you want: "fix the broken sign-up layout I just captured."
5. Claude Code reads the latest JSON automatically and acts on it.

## Why

Pasting a raw screenshot to a coding agent costs hundreds to thousands of vision tokens per turn and the agent re-interprets pixels every time. A SlimSnap JSON costs about 700 tokens, is structured (the agent acts on coordinates), and is reusable across turns without re-paying the cost. Over a long Claude Code session, the difference in context and token spend is real.

## Where it looks

The skill reads from these locations in order:

1. `<project root>/.slimsnap/` (per-project, opt-in)
2. `~/Documents/SlimSnap/` (user-global, the default SlimSnap output)

Most recent `.json` by modification time wins. You can also pass a specific filename as an argument.

## Spec

The JSON format is an open MIT spec: https://github.com/bickov/slimsnap-schema

## License

MIT.
