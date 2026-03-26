# Claude Session Manager Commands Setup
`v1.1.0`

Create the following three custom command files in `~/.claude/commands/`.

---

## 1. `~/.claude/commands/session-info.md`

````
Print the current session ID.

1. Find the most recently modified JSONL file in the current project's ~/.claude/projects/ folder:
   ```
   ls -t ~/.claude/projects/<encoded-pwd>/*.jsonl | head -1
   ```
   encoded-pwd: replace `/` with `-` in the current working directory path

2. The filename (without extension) is the current session ID. Output only in this format:
   ```
   Session ID: <uuid>

   To resume this session from another project (run inside claude after switching projects):
   /session-import <uuid>

   To delete this session (exit Claude with Ctrl+C first, then run in terminal):
   rm ~/.claude/projects/<encoded-pwd>/<uuid>.jsonl
   ```
````

---

## 2. `~/.claude/commands/session-import.md`

````
Copy a session file from another project into the current project.

Argument: $ARGUMENTS (session ID or none)

## When no argument is provided

1. List all available projects:
   ```
   ls ~/.claude/projects/
   ```
   Convert `-` back to `/` in folder names for human-readable paths.

2. Ask the user to select a source project.

3. Show recent sessions in the selected project:
   ```
   ls -lt ~/.claude/projects/<selected-project>/*.jsonl | head -10
   ```

4. Ask the user to select a session ID to import.

## When a session ID is provided (or after selection)

1. Compute current project path:
   - Replace `/` with `-` in the result of `pwd`

2. Find source file (search all projects for the session ID):
   ```
   ls ~/.claude/projects/*/<session-id>.jsonl 2>/dev/null
   ```

3. Copy to current project folder:
   ```
   cp <source-path>/<session-id>.jsonl ~/.claude/projects/<current-project>/
   ```

4. Print result:
   ```
   ✓ Session copied
   from: <source project>
   to:   ~/.claude/projects/<current-project>/

   To resume: claude --resume
   ```
````

---

## 3. `~/.claude/commands/session-search.md`

````
Search through conversation files in the current project by content.

1. Ask the user in a single prompt:
   - Search term (required)
   - Time range (optional): "today", "last N days", "after YYYY-MM-DD" — searches all if omitted
   Example: "Enter search term (e.g. 'http server' or 'http server / last 7 days')"

2. Determine the current project folder:
   - encoded-pwd: current working directory with `/` replaced by `-`
   - project dir: `~/.claude/projects/<encoded-pwd>/`

3. Find matching `.jsonl` files in that project dir, filtered by date if specified:
   - If "today": `find ~/.claude/projects/<encoded-pwd>/ -name "*.jsonl" -mtime -1`
   - If "last N days": `find ~/.claude/projects/<encoded-pwd>/ -name "*.jsonl" -mtime -N`
   - If "after YYYY-MM-DD": on macOS use `touch -t YYYYMMDD0000 /tmp/ref_date` then `find ~/.claude/projects/<encoded-pwd>/ -name "*.jsonl" -newer /tmp/ref_date`
   - If omitted: all `.jsonl` files in the project dir

4. Search for the term **only in user messages**. Use python3 to parse JSONL and filter:
   ```
   python3 -c "
   import json, sys, re
   pattern = sys.argv[1]
   for line in open(sys.argv[2]):
       try:
           obj = json.loads(line)
           # Only check user-role messages (not assistant, not thinking, not tool results)
           msg = obj.get('message', {})
           if msg.get('role') != 'user':
               continue
           content = msg.get('content', '')
           if isinstance(content, list):
               text = ' '.join(c.get('text', '') for c in content if isinstance(c, dict) and c.get('type') == 'text')
           else:
               text = str(content)
           if re.search(pattern, text, re.IGNORECASE):
               print(text[:200])
       except: pass
   " "<search-term>" <file>
   ```
   Run this for each candidate file to find matches.

5. For each matching file, show a snippet of the matched user message (the text printed by the python3 command above).

6. Display results as a numbered list:
   ```
   1. [Session ID] | [modified date/time]
      Preview: "...matched content..."
      To resume (run in terminal):
      claude --resume <uuid>
   ```

7. If no results: "No results found".
````
