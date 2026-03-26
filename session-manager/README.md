# Claude Session Manager Commands

Three custom commands to search, resume, and import Claude Code sessions across projects.

- `/session-info` — View current session ID + delete command
- `/search-session` — Search session content in current project + resume command
- `/import-session` — Copy a session from another project to the current one

---

## Setup

Paste the contents of [prompt.md](./prompt.md) into Claude Code. Three files will be created automatically.

---

## Usage

### View Current Session

Input:
```
/session-info
```

Output:
```
Session ID: 69d9529b-df37-4745-a92c-5dc384e59576

To resume this session from another project (run inside claude after switching projects):
/import-session 69d9529b-df37-4745-a92c-5dc384e59576

To delete this session (exit Claude with Ctrl+C first, then run in terminal):
rm ~/.claude/projects/-Users-yourname-myrepo/69d9529b-df37-4745-a92c-5dc384e59576.jsonl
```

### Search Session Content

Input:
```
/search-session
→ http server / last 7 days
```

Output:
```
1. 69d9529b | 2026-03-26 17:41
   Preview: "...was looking into how to start an http server..."
   To resume (run in terminal):
   claude --resume 69d9529b-df37-4745-a92c-5dc384e59576
```

### Import Session from Another Project

Input:
```
/import-session 69d9529b-df37-4745-a92c-5dc384e59576
```

Output:
```
✓ Session copied
from: /Users/yourname/other-project
to:   ~/.claude/projects/-Users-yourname-myrepo/

To resume (run in terminal):
claude --resume 69d9529b-df37-4745-a92c-5dc384e59576
```
