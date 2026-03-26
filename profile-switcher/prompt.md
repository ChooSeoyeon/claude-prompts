# Claude Code Profile Switcher Setup
`v1.0.0`

Sets up a profile-switching system to use Claude Code with different settings across multiple environments (company, team, personal, etc.).

Step 1 is performed by the user. Claude handles the rest automatically.

---

## Step 1: Define Profiles (user)

Tell Claude the list of profiles you want and the environment variables for each.

Example:
- `work`: ANTHROPIC_BASE_URL=https://my-proxy.example.com, ANTHROPIC_AUTH_TOKEN=sk-xxx
- `personal`: ANTHROPIC_AUTH_TOKEN=sk-yyy

Proceed below when done.

---

## Step 2: Automatic Installation (Claude handles this)

### 2-1. Create ~/.claude/profiles/ folder

### 2-2. Create ~/.claude/profiles/base.json

Common settings file applied to all profiles. Create with the following content:

```json
{}
```

> Add any settings that should apply to all profiles (hooks, plugins, etc.) to this file.

### 2-3. Create profile JSON files

For each profile defined in Step 1, create `~/.claude/profiles/<name>.json`.

Format:
```json
{
  "env": {
    "KEY": "VALUE"
  }
}
```

### 2-4. Create ~/.claude/profiles/switch.sh

Create with the following content and grant execute permission (`chmod +x`):

```bash
#!/bin/bash

PROFILES_DIR="$HOME/.claude/profiles"
SETTINGS="$HOME/.claude/settings.json"

PROFILE="$1"

if [[ -z "$PROFILE" ]]; then
  echo "Usage: switch.sh <profile>"
  echo "Available profiles: $(ls $PROFILES_DIR/*.json 2>/dev/null | xargs -n1 basename | sed 's/\.json//' | grep -v base | tr '\n' ', ' | sed 's/,$//')"
  exit 1
fi

PROFILE_FILE="$PROFILES_DIR/$PROFILE.json"

if [[ ! -f "$PROFILE_FILE" ]]; then
  echo "Profile '$PROFILE' not found at $PROFILE_FILE"
  exit 1
fi

jq -s '.[0] * .[1]' "$PROFILES_DIR/base.json" "$PROFILE_FILE" > "$SETTINGS"

echo "Switched to '$PROFILE' profile"
```

### 2-5. Add aliases to ~/.zshrc

For each profile defined in Step 1, add an alias in the following format:

```
# Claude profile switcher
alias claude-<profile>='~/.claude/profiles/switch.sh <profile> && claude'
```

### 2-6. Print completion instructions

```
── Setup Complete ──
Run `source ~/.zshrc` in terminal before using.

Usage:
  claude-<profile>   Update settings.json to that profile and launch Claude
  claude             Launch Claude with current settings.json as-is

Edit common settings:  ~/.claude/profiles/base.json
Edit profile settings: ~/.claude/profiles/<name>.json
Add new profile:       Create JSON file and add alias
```
