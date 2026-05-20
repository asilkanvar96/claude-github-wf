---
name: github-wf
description: 'GitHub workflow automation. Branch, commit, push, PR — EN and TR natural language.'
skill_api_version: 1
context:
  window: isolated
  intent:
    mode: task
---

# GitHub Workflow Skill (Claude Code)

## Purpose
Automate GitHub operations via natural language. Full tool access — runs git, gh CLI, edits files.
Supports English and Turkish.

---

## Startup — BLOCKING, Her Zaman Çalıştır

**ZORUNLU:** Skill her çağrıldığında, kullanıcının isteği ne olursa olsun, önce bu startup adımlarını tamamla. Startup bitmeden hiçbir işlem yapma. "branch aç", "commit et" gibi doğrudan istekler gelse bile önce Step 1→2→3 sırasını tamamla, onay al, sonra işleme geç.

When invoked (`/github-wf`) or user asks "nasıl çalışıyor" / "hangi repo" / "who am I":

**Step 1 — gather state:**
```bash
git config user.name
git config user.email
git remote -v
git rev-parse --abbrev-ref HEAD
gh auth status
git status
git log --oneline -5
git branch -a
```

**Step 2 — present summary:**
```
─────────────────────────────────────────
GitHub Workflow Skill aktif.

Kullanıcı : Ada Lovelace <ada@example.com>
Repo       : origin → https://github.com/owner/repo.git
Branch     : develop
Auth       : ✓ Logged in as adalovelace (github.com)

Son 5 commit:
  abc1234 fix: token expiry check
  def5678 feat: add user settings page

Durum: temiz (değişiklik yok)
─────────────────────────────────────────
Aktif branch stratejisi:

  Korumalı  : main, master  ← direkt push YASAK
  Base       : develop       ← tüm iş buradan başlar
  Dallar     : feat/* | fix/* | chore/* | docs/* | refactor/*

Değiştirmek ister misin?
  "base branch X olsun"  → base branch güncellenir
  "X de korumalı olsun"  → koruma listesine eklenir
  "hayır" / "default"    → olduğu gibi devam
─────────────────────────────────────────
```

**Step 3 — branch config (MANDATORY, her çalıştırmada sor):**

```
Branch stratejisini onaylayalım:

  Korumalı  : main, master
  Base       : develop
  Dallar     : feat/* | fix/* | chore/* | docs/* | refactor/*

Bu oturum için geçerli mi?
  [1] Evet, default devam
  [2] Base branch değiştir
  [3] Korumalı branch ekle/çıkar
```

- Seçim yapılana kadar devam etme
- "1" / "evet" / "default" → onayla, session başlat
- "2" → "Yeni base branch nedir?" sor, güncelle
- "3" → "Hangi branch korumalı olsun / çıkarılsın?" sor, güncelle
- Özelleştirildiyse → "Tamam. Bu oturum: base=X, korumalı=Y,Z" onayla

**Step 4 — contextual kickoff question:**

Mevcut duruma göre bir sonraki adımı öner:

```
Durum: temiz + develop'ta
→ "Yeni bir şeye mi başlayacaksın yoksa mevcut PR var mı?"

Durum: uncommitted changes var
→ "Değişiklikler var ama commit edilmemiş. Kaydetmek ister misin?"

Durum: committed ama push yok
→ "X commit var, push edilmemiş. Push edeyim mi?"

Durum: pushed ama PR yok
→ "Branch push'lu ama PR açılmamış. PR açayım mı?"

Durum: feature branch'teyim
→ "feat/X branch'indesin. Devam mı ediyorsun yoksa PR'a hazır mısın?"
```

If auth missing → print auth setup steps first.
If no remote → warn: "Remote yok. `git remote add origin <url>`"
If on main/master → warn: "main'desin. Yeni branch aç:" + suggest `git checkout -b feat/...`

---

## Session Tracking

**Track these internally during session:**
```
session_actions = []   # her işlem eklenir
action types: branch_created | staged | pushed | pr_opened | synced | undone | cleaned
```

**Periyodik özet — her 3 işlemde bir otomatik göster:**
```
─── Oturum Özeti ──────────────────────────
✓ Branch açıldı  : feat/login-fix
✓ 2 staged       : "feat: add login" / "fix: token check" (committed on push)
✓ Push           : origin/feat/login-fix
⏳ PR             : henüz açılmadı
Korumalı branch'e push: YOK ✓
───────────────────────────────────────────
```

**Manuel özet** — "özet", "ne yaptık", "summary" sorularında da göster.

**Oturum sonu özeti** — "bitti" / "done" / cleanup'ta göster:
```
─── Oturum Tamamlandı ─────────────────────
Branch   : feat/login-fix → develop (PR açıldı)
Commitler: 3
Push     : ✓
PR       : #42 — incelemede
Silinen  : feat/login-fix (local)
Koruma   : main'e direkt push → 0 kez ✓
───────────────────────────────────────────
```

---

## Branch Protection — Hard Enforcement

Before ANY push or commit, check target:
```bash
git rev-parse --abbrev-ref HEAD
```

**If current branch = main, master, or any protected branch:**

> **DUR.** main/master'a direkt push yapılmaz. Önce branch aç:
> ```bash
> git checkout -b feat/your-work
> ```
> Sonra push et, PR aç → develop/main'e merge.

Never bypass this check. No exceptions unless user explicitly says "override korumayı" — then warn once more before proceeding.

---

## Authentication Setup

Each person authenticates with their own GitHub account. Skill stores no credentials.

**Browser login (recommended):**
```bash
brew install gh && gh auth login && gh auth status
```

**Token-based (CI, no browser):**
```bash
export GH_TOKEN=ghp_yourtoken   # add to ~/.zshrc to persist
```

**SSH:**
```bash
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub   # paste → GitHub → Settings → SSH keys
git remote set-url origin git@github.com:<owner>/<repo>.git
```

---

## Trigger Phrases → Actions

### Start Work
**EN:** "start working on X", "work on X", "new feature X", "fix X"
**TR:** "X üzerinde çalışmaya başla", "X özelliği ekle", "X hatasını düzelt", "X için branch aç"

```bash
git checkout <base-branch> && git pull
git checkout -b <type>/<slug>
```
- base-branch = session config'deki değer (default: develop)
- Bug/hata → `fix/`, yeni/new → `feat/`
- Slugify (lowercase, hyphens, max 4 words)
- Confirm branch name before creating
- Log: `branch_created`

**Sonrası:** "Branch hazır. Değişiklikleri yaparken 'kaydet' de. Commit mesajını otomatik oluşturacağım."

---

### Save Progress (Stage Only — NO commit)
**EN:** "save", "checkpoint", "stage this"
**TR:** "kaydet", "stage et", "buraya kadar kaydet"

```bash
git add -A && git status
```
- Stage all changes (NO commit yet)
- Generate commit message from diff (Conventional Commits format)
- Save message to `.git/STAGED_NOTES` (append, one line per save):
  ```
  feat: add login form
  fix: button alignment
  ```
- Never stage: `.env`, `*.key`, `*.pem`, `*secret*`
- Log: `staged`

**Sonrası:** "Stage tamam. Commit mesajı kayıt altında: `<generated-message>`. Daha değişiklik var mı? Push etmek istediğinde 'push et' de."

---

### Push (Commit staged + push)
**EN:** "push", "push it"
**TR:** "push et", "gönder"

**First check branch protection.** If on protected branch → block, warn, stop.

**If `.git/STAGED_NOTES` exists (staged but uncommitted changes):**
1. Show accumulated notes
2. Ask: "Tek commit mi, yoksa her kayıt ayrı commit mi olsun?"
   - Tek commit → join all lines as body, first line as subject
   - Ayrı commits → `git commit -m "<line>"` for each note + stage between (not possible retroactively → use first line as main, rest as body)
3. ```bash
   git commit -m "$(head -1 .git/STAGED_NOTES)" -m "$(tail -n +2 .git/STAGED_NOTES)"
   rm .git/STAGED_NOTES
   ```
4. Then push:
   ```bash
   git push origin <current-branch>    # or -u for first push
   ```

**If no STAGED_NOTES (already committed locally):**
```bash
git push origin <current-branch>
```

- Log: `pushed`

**Sonrası:** "Push tamam. PR açayım mı? `develop`'a base alsın mı, yoksa farklı bir branch mi?"

---

### Open PR
**EN:** "open PR", "ready for review", "ship it"
**TR:** "PR aç", "incelemeye gönder", "review'a aç"

```bash
gh pr create \
  --title "<type>: <description>" \
  --base <base-branch> \
  --body "$(cat <<'EOF'
## What
<summarize changes from commit history>

## Why
<reason from conversation context>

## Test
- [ ] Manual test done
- [ ] No regressions observed
EOF
)"
```
- base = session config'deki base branch
- Log: `pr_opened`

**Sonrası:** "PR açıldı. Review bekleyecek misin yoksa branch'i temizleyelim mi? (merge sonrası temizlenmeli)"

---

### Sync
**EN:** "sync", "get latest", "pull updates"
**TR:** "sync et", "son değişiklikleri çek", "pull yap"

```bash
git fetch origin
git pull --rebase origin <base-branch>
```
- Conflicts → list files, do NOT auto-resolve, ask user
- Log: `synced`

**Sonrası:** "Sync tamam. Yeni commit'ler var mıydı? Devam mı ediyorsun yoksa yeni branch mi açıyoruz?"

---

### Check Status / Özet
**EN:** "what changed", "status", "summary", "what did we do"
**TR:** "ne değişti", "durum nedir", "özet", "ne yaptık"

```bash
git diff && git diff --staged && git log --oneline -10
```
- "özet" / "summary" → show session tracking summary
- "status" / "ne değişti" → show diff summary

---

### Undo
**EN:** "undo", "revert", "go back"
**TR:** "geri al", "hata yaptım", "son commiti iptal et"

Ask: "last commit, staged changes, specific file, or all unstaged?"
- Staged changes (not committed): `git restore --staged .` + delete `.git/STAGED_NOTES`
- Last commit (not pushed): `git reset HEAD~1`
- Specific file: `git checkout -- <file>`
- All unstaged: `git restore .`
- Log: `undone`

> **Warning:** `git reset HEAD~1` removes the last commit. If pushed → requires force push — confirm before proceeding.

---

### Done / Cleanup
**EN:** "done", "clean up", "finished"
**TR:** "bitti", "branch'i sil", "temizle"

Show session summary first, then:
```bash
git checkout <base-branch>
git branch -d <old-branch>
```
- Only after PR is merged
- Confirm before deleting
- Log: `cleaned`

---

## Rules (Always Enforce)

```
NEVER push directly to main, master, or any protected branch
NEVER force push shared branches without explicit confirmation
NEVER commit: .env* | *.key | *.pem | *secret* | credentials*
ALWAYS branch from base-branch (session config, default: develop)
ALWAYS use Conventional Commits format
ALWAYS confirm destructive actions
ALWAYS show session summary every 3 actions
```

---

## Branch Naming

```
feat/<slug>       new feature / yeni özellik
fix/<slug>        bug fix / hata düzeltme
chore/<slug>      maintenance, deps, config
docs/<slug>       documentation
refactor/<slug>   restructure, no behavior change
```

## Commit Types

```
feat      new capability
fix       bug correction
chore     tooling, deps, config
docs      documentation
refactor  restructure without behavior change
test      tests
ci        CI/CD changes
```

---

## Session Flow

```
/github-wf çağrılır
    ↓
startup: user + repo + auth + branch config
    ↓
branch config: default onayla veya özelleştir
    ↓
task alınır (EN veya TR)
    ↓
her işlemde: korumalı branch kontrolü
    ↓
her 3 işlemde: otomatik oturum özeti
    ↓
"kaydet"   → stage only + commit msg → .git/STAGED_NOTES
"push et"  → commit (from STAGED_NOTES) + push (koruma kontrol)
"PR aç"    → PR (base branch = config)
"bitti"    → oturum özeti + cleanup
```
