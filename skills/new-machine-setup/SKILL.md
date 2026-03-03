---
name: new-machine-setup
description: Use when bootstrapping a new machine — OpenCode is already installed, nothing else is
---

# New Machine Setup

**Announce at start:** "I'm using the new-machine-setup skill to set up this machine."

**Starting assumption:** OpenCode is installed. Nothing else.

---

## Step 1: Install prerequisites

```bash
# Fedora / CentOS Stream
sudo dnf install gh

# Or via Homebrew (Linux)
brew install gh
```

Install `just`:
```bash
mkdir -p ~/.local/bin
wget -qO- "https://github.com/casey/just/releases/download/1.34.0/just-1.34.0-x86_64-unknown-linux-musl.tar.gz" \
  | tar --no-same-owner -C ~/.local/bin -xz just
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
```

Verify:
```bash
just --version
```

---

## Step 2: Authenticate GitHub CLI

```bash
gh auth login
gh auth status
```

---

## Step 3: SSH key for GitHub

**This must complete before cloning the private config repo.**

Verify existing key:
```bash
ssh -T git@github.com
# Expected: "Hi castrojo! You've successfully authenticated..."
```

If missing, generate and add:
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
gh ssh-key add ~/.ssh/id_ed25519.pub --title "$(hostname)"
ssh -T git@github.com
```

---

## Step 4: Clone opencode-config

If `~/.config/opencode` already exists from OpenCode's first-run init:
```bash
cp ~/.config/opencode/agent-memory.json /tmp/agent-memory.json.bak 2>/dev/null || true
rm -rf ~/.config/opencode
```

Clone:
```bash
git clone git@github.com:castrojo/opencode-config.git ~/.config/opencode
```

Verify:
```bash
head -3 ~/.config/opencode/AGENTS.md
ls ~/.config/opencode/memory/
```

**Automatically restored by this clone:**
- `AGENTS.md` — global workflow rules
- `opencode.json` — providers, plugin list (`opencode-agent-memory` auto-fetched at startup)
- `memory/persona.md`, `memory/human.md` — trained agent context, ready immediately
- `agent-memory.json` — journal enabled, custom tags configured
- `agents/` — installed skill files
- `plans/` — project reference docs

---

## Step 5: Clone superpowers and create symlinks

Clone your fork (not upstream — you own customized skills):

```bash
git clone git@github.com:castrojo/superpowers.git ~/.config/opencode/superpowers
```

Set up remote layout (upstream for syncing, never push there):

```bash
cd ~/.config/opencode/superpowers
git remote add upstream git@github.com:obra/superpowers.git
git remote -v
# origin    → castrojo/superpowers  (your fork — push here only)
# upstream  → obra/superpowers      (upstream — fetch only, NEVER push)
```

Check out your customizations branch:

```bash
git checkout feat/castrojo-customizations
```

Create plugin symlink:
```bash
mkdir -p ~/.config/opencode/plugins
ln -s ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js \
      ~/.config/opencode/plugins/superpowers.js
```

Create skills symlink:
```bash
mkdir -p ~/.config/opencode/skills
ln -s ~/.config/opencode/superpowers/skills \
      ~/.config/opencode/skills/superpowers
```

Verify:
```bash
ls -l ~/.config/opencode/plugins/superpowers.js
ls -l ~/.config/opencode/skills/superpowers
```

---

## Step 6: Install npm dependencies

```bash
cd ~/.config/opencode
npm install
```

This restores `@opencode-ai/plugin`, `jsonc-parser`, and `zod`.

---

## Step 7: Reinstall skills

Use the OpenCode skill installer to reinstall all skills into `~/.agents/skills/`:

| Skill | Source |
|---|---|
| `gh-cli` | `github/awesome-copilot` |
| `find-skills` | `vercel-labs/skills` |
| `code-review` | `supercent-io/skills-template` |
| `github-actions-templates` | `wshobson/agents` |
| `centos-linux-triage` | `github/awesome-copilot` |
| `fedora-linux-triage` | `github/awesome-copilot` |
| `shellcheck-configuration` | `wshobson/agents` |
| `container-debugging` | `aj-geddes/useful-ai-prompts` |
| `devops-engineer` | `jeffallan/claude-skills` |
| `git-commit` | `github/awesome-copilot` |
| `bash-linux` | `sickn33/antigravity-awesome-skills` |

Superpowers skills (`~/.config/opencode/skills/superpowers/`) are restored by the symlink in step 5 — no reinstall needed.

---

## Step 8: Verify

```bash
gh auth status                                      # GitHub CLI authenticated
just --version                                      # just available
ssh -T git@github.com                               # SSH key working
head -3 ~/.config/opencode/AGENTS.md               # global rules present
ls ~/.config/opencode/memory/                       # memory blocks present
cat ~/.config/opencode/agent-memory.json            # journal config present
ls ~/.config/opencode/plugins/superpowers.js        # superpowers plugin symlink
ls ~/.config/opencode/skills/superpowers            # superpowers skills symlink
ls ~/.agents/skills/                                # installed skills present
```

All 9 checks must pass before starting any development work.

---

## Synced via git (config repo)

| Path | Notes |
|---|---|
| `AGENTS.md` | Global workflow rules |
| `opencode.json` | Providers, plugin list |
| `memory/persona.md` | Agent persona — trained context |
| `memory/human.md` | Human preferences — trained context |
| `agent-memory.json` | Journal config (tags, enabled flag) |
| `agents/` | Installed skill files |
| `plans/` | Project reference docs |

## Not synced (set up manually)

| Path | Notes |
|---|---|
| `superpowers/` | Clone in step 5 |
| `plugins/` | Symlinks created in step 5 |
| `skills/` | Symlink created in step 5 |
| `journal/` | Runtime — session entries, not config |
| `node_modules/` | `npm install` in step 6 |
| `settings.json` | Runtime state |

---

After setup, use the `onboarding-a-repository` skill for each repo.
