# Vibe Coder's Guide to MVP — Agent Skills

A portable bundle of markdown skills that walks an AI coding agent (Claude Code, Codex) from nothing or an existing prototype all the way to a deployed, MVP-quality web app.

## Quick start (one-line copy/paste)

Paste this into your AI coding agent (Claude Code, Codex, Cursor, …):

```
Help me ship an MVP. Bootstrap from https://github.com/titan-alpha/vibe-coders-guide-to-mvp-skills — read AGENTS.md and follow it strictly.
```

The agent will ask you A/B/C, clone what's needed, drop the skills into the
right place for your platform, and walk you through everything else.

> Want to read what the agent reads?
> [`AGENTS.md`](./AGENTS.md) (bootstrap protocol) → [`vibe-mvp/SKILL.md`](./vibe-mvp/SKILL.md) (the methodology).

## What this is

Fifteen numbered sub-skills plus a `SKILL.md` entry point. Your agent reads
`SKILL.md` first, then walks the numbered files in order. Each sub-skill has
two modes of work:

- **DIALOGUE** — the agent asks you questions (one or two at a time).
- **AUTONOMOUS** — the agent does the work itself, reporting at the end.

The skills cover: discovery, design, auth, AI integration, chatbot, admin
dashboard, compliance (GDPR/CCPA/TOS/Privacy), accessibility, security,
performance, deploy, domain, end-to-end testing, ship checklist, and founder
deliverables (pitch deck, one-pagers, ads, launch copy).

## How to use

### With Claude Code

```bash
git clone https://github.com/titan-alpha/vibe-coders-guide-to-mvp-skills.git
mkdir -p .claude/skills
cp -r vibe-coders-guide-to-mvp-skills/vibe-mvp .claude/skills/
# Then in Claude Code:
#   "Read .claude/skills/vibe-mvp/SKILL.md and follow it."
```

### With Codex

```bash
git clone https://github.com/titan-alpha/vibe-coders-guide-to-mvp-skills.git
mkdir -p .codex/skills
cp -r vibe-coders-guide-to-mvp-skills/vibe-mvp .codex/skills/
```

### Or use the starter project

If you haven't started a project yet, the companion starter repo includes this
skill bundle pre-installed: **[vibe-coders-guide-to-mvp-starter](https://github.com/titan-alpha/vibe-coders-guide-to-mvp-starter)**.

## The pitch

You can already vibe-code something that *almost* works. What you're missing
isn't talent — it's the confidence that what you're building is actually
shippable, and a checklist of the things a senior engineer wouldn't skip.

That's what this is. A stack of skills you hand your agent. It asks the right
questions and builds the right pieces — so you ship a real MVP, not a demo.

**Believe you can. You can.**

## License

MIT.
