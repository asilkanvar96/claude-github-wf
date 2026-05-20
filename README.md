# claude-github-wf

Claude Code skill for GitHub workflow automation via natural language. Supports EN + TR.

## What it does

Branch, stage, push, PR — all via chat. Changes are auto-staged after each file edit; commit + push on demand.

## Install

**1. Install the skill:**
```bash
mkdir -p ~/.claude/skills/github-wf
curl -sL https://raw.githubusercontent.com/asilkanvar96/claude-github-wf/main/SKILL.md \
  > ~/.claude/skills/github-wf/SKILL.md
```

**2. (Optional) Add auto-stage hook to your project:**
```bash
mkdir -p <your-project>/.claude
curl -sL https://raw.githubusercontent.com/asilkanvar96/claude-github-wf/main/settings.json \
  > <your-project>/.claude/settings.json
```
This makes Claude auto-run `git add -A` after every file edit — no need to say "stage this".

**3. Authenticate GitHub CLI:**
```bash
brew install gh && gh auth login
```

## Usage

In Claude Code, type `/github-wf` to start a session.

**Trigger phrases:**
| Action | EN | TR |
|--------|----|----|
| New branch | "work on X", "fix X" | "X için branch aç" |
| Stage | "save" | "kaydet" |
| Push | "push" | "push et" |
| Open PR | "open PR", "ship it" | "PR aç" |
| Sync | "sync" | "sync et" |
| Status | "status" | "durum nedir" |
| Undo | "undo" | "geri al" |
| Done | "done" | "bitti" |

## How it works

- `save` / `kaydet` → stages changes + saves commit message to `.git/STAGED_NOTES`
- `push` / `push et` → commits from STAGED_NOTES + pushes
- Never pushes directly to `main` or `master`
