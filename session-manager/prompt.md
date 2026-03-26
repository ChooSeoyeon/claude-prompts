# Claude Session Manager Commands Setup
`v1.0.0`

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
   /import-session <uuid>

   To delete this session (exit Claude with Ctrl+C first, then run in terminal):
   rm ~/.claude/projects/<encoded-pwd>/<uuid>.jsonl
   ```
````

---

## 2. `~/.claude/commands/import-session.md`

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

## 3. `~/.claude/commands/search-session.md`

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

4. Search for the term in those files:
   ```
   grep -rl "<search-term>" <filtered files>
   ```

5. For each matching file, show a snippet:
   ```
   grep -o '.\{0,80\}<search-term>.\{0,80\}' <file> | head -3
   ```
   Parse JSON to extract readable text from `"text":` or `"content":` fields if possible.

6. Display results as a numbered list:
   ```
   1. [Session ID] | [modified date/time]
      Preview: "...matched content..."
      To resume (run in terminal):
      claude --resume <uuid>
   ```

7. If no results: "No results found".
````
