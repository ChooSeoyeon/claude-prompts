# Claude Code Profile Switcher

A profile-switching system for using Claude Code with different settings across multiple environments (company LLM proxy, team-specific configs, etc.).

- **Separate API endpoints/tokens per environment** — manage company proxy, personal account, etc. independently
- **Common settings in one place** — hooks, plugins, etc. managed in a single base.json
- **Switch with one command** — instant switching via aliases like `claude-work`, `claude-personal`

---

## Setup

Paste the contents of [prompt.md](./prompt.md) into Claude Code and follow the instructions.

---

## Usage

```
claude-<profile>   Update settings.json to that profile and launch Claude
claude             Launch Claude with current settings.json as-is
```

---

## Structure

```
~/.claude/profiles/
├── base.json          # Common settings (hooks, plugins, etc.)
├── <profile>.json     # Per-profile env settings
└── switch.sh          # Switching script
```

`switch.sh` merges `base.json + <profile>.json` and saves as `~/.claude/settings.json`.
