# slimsnap-skill

A [Claude Code](https://claude.com/claude-code) skill for [SlimSnap](https://slimsnap.ai). Lets Claude Code automatically read structured JSON captures of your screen, so you can say "fix this layout" instead of pasting a screenshot every turn.

About 700 tokens per capture vs the API per-image cap of 1,568 tokens on Sonnet/Haiku (up to 4,784 on Opus 4.7+ per [Anthropic's vision docs](https://platform.claude.com/docs/en/build-with-claude/vision)). Structured bounding boxes, extracted colors, OCR text, and your annotations.

Works with **scroll captures** too (schema v2): a whole page, a long chat, an endless feed arrives as `frames[]` — page-sized units the agent reads in order, each with its own elements and annotations. Vision models can't read one tall stitched screenshot (they downscale it until the text is gone); frames JSON has no such limit.

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

1. Launch SlimSnap at least once so it publishes its config to `~/.slimsnap/config.json`.
2. Capture a screenshot with SlimSnap (`⌘⇧S`), or a scrolling capture of a whole page (`⌘⇧L` — scroll yourself or press Autoscroll).
3. Annotate what matters: arrows, rectangles, callouts.
4. Hit **Save JSON** (autosave on by default writes straight to your configured folder).
5. In Claude Code, say what you want: "fix the broken sign-up layout I just captured."
6. Claude Code reads the latest JSON automatically and acts on it.

## Why

Pasting a raw screenshot to a coding agent costs hundreds to thousands of vision tokens per turn and the agent re-interprets pixels every time. A SlimSnap JSON costs about 700 tokens, is structured (the agent acts on coordinates), and is reusable across turns without re-paying the cost. Over a long Claude Code session, the difference in context and token spend is real.

## How discovery works

The skill is **not hardcoded** to a specific folder. SlimSnap publishes a tiny config at `~/.slimsnap/config.json` naming its current default save folder. The skill reads that file on every invocation and looks in whatever folder you have configured in the SlimSnap Settings window. Change the folder in SlimSnap and the skill follows.

If you prefer keeping captures with a codebase, create a `<project>/.slimsnap/` folder and the skill will prefer it when present.

## Spec

The JSON format is an open MIT spec: https://github.com/bickov/slimsnap-schema

## License

MIT.
