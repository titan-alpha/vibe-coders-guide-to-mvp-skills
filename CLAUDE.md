# Bootstrap protocol — Vibe Coder's Guide to MVP (skills repo)

You are an AI coding agent (Claude Code, Codex, Cursor, etc.) and the user has
asked you to help them ship an MVP using this skill bundle. Follow the protocol
below **strictly from the top, in order, without skipping steps.**

---

## Step 1 — Ask one short question, then act

> *"What should I do — (A) start a new project from scratch, (B) start a new
> project with just the skills, or (C) finish/harden a project you already
> have?"*

Wait for A, B, or C. Then ask **one** follow-up: where on disk to put it (for
A/B) or the path to the existing project (for C).

## Step 2 — Set up the working directory

Pick the matching block. Use the **agent-specific skills directory**:
`.claude/skills/vibe-mvp` for Claude Code, `.codex/skills/vibe-mvp` for Codex,
`.cursor/skills/vibe-mvp` for Cursor, etc. (substituted as `<SKILLS_DIR>` below).

**(A) New project from scratch — uses the project template:**

```bash
git clone https://github.com/titan-alpha/vibe-coders-guide-to-mvp-starter.git <name>
cd <name>
mkdir -p <SKILLS_DIR>
cp -r vibe-mvp/* <SKILLS_DIR>/
```

**(B) New project, skills only:**

```bash
mkdir <name> && cd <name>
git clone https://github.com/titan-alpha/vibe-coders-guide-to-mvp-skills.git skills-tmp
mkdir -p <SKILLS_DIR>
cp -r skills-tmp/vibe-mvp/* <SKILLS_DIR>/
rm -rf skills-tmp
```

**(C) Existing project:**

```bash
cd <existing-project-path>
git clone https://github.com/titan-alpha/vibe-coders-guide-to-mvp-skills.git skills-tmp
mkdir -p <SKILLS_DIR>
cp -r skills-tmp/vibe-mvp/* <SKILLS_DIR>/
rm -rf skills-tmp
```

For **(C)**, after cloning the skills, **read enough of the existing repo**
(package.json, README, top-level structure, recent commits) to understand what's
already built **before** suggesting anything. Do not touch existing code yet.

## Step 3 — Read SKILL.md and follow it strictly from the top

Open `<SKILLS_DIR>/SKILL.md`. **Do not skip the section called
"Mode selection". It is the first action.** That section will tell you to:

1. Offer the user **two paths**: (a) Quick chat dialogue, (b) Visual
   configurator (web UI). **Always offer both** — never assume the user
   wants chat. Many users prefer the visual interface and don't know it
   exists unless you tell them.
2. Once the user picks a path, **lock the mode** (which prescribes the skill
   plan) before starting any sub-skill.
3. Only after mode + path are locked may you begin sub-skill `01-discover.md`.

If you find yourself starting `01-discover.md` without having offered Path A
vs Path B and locked a mode, **stop and back up.**

## Step 4 — Operating rules (apply throughout)

- Store all credentials in `.env.local`. Never hardcode secrets in source.
- Pause and ask before any destructive action, any step that costs money, or
  any change to existing code that the user might not expect.
- Update `STATE.yaml` and `PROJECT.md` at the project root as you go (the
  skills tell you when).

---

That's it. Begin Step 1 now.
