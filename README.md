# rekor-skill

The official Rekor skill for filesystem-having AI agents. Drives the canonical onboarding flow at [rekor.pro/docs/get-started](https://rekor.pro/docs/get-started).

---

## 1. Install the Rekor skill

One command, works across Claude Code, Codex, Cursor, OpenCode, Cline, GitHub Copilot, and 48 more.

```
mkdir -p .claude && npx skills add wayai-pro/rekor-skill -y
```

Installs the skill for whichever AI agent you have set up on this machine. The `mkdir -p .claude` guarantees Claude Code gets the skill link even in a fresh repo; the `-y` skips prompts so it runs unattended. If your agent is already running, ask it to run the command — it will pick up the skill on its next turn.

## 2. Tell your agent what you want

After install, just ask. The skill walks your agent through CLI install, login, base creation, your first record_type, the preview→production flow, and connecting production agents.

```
Set up Rekor as the data layer for my customer support agent.
```

## 3. Don't have an account yet?

Free plan includes 1,000 operations per month. No credit card required.

[Create a free account](https://rekor.pro)
