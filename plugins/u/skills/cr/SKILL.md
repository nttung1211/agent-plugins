---
name: cr
description: Create, list, attach to, and remove Claude Code Remote Control sessions in other folders via tmux.
argument-hint: "init | mk <name> <dir> | ls | rm <name> | cl"
disable-model-invocation: true
allowed-tools: Bash(tmux:*), Bash(sleep:*), Bash(command -v:*), Bash(ls:*), Bash(mkdir:*)
---

# /u:cr — Remote Control sessions in tmux

Raw input: `$ARGUMENTS`

Split the input on whitespace. The first word is the subcommand; the rest are its arguments. With no arguments, an unknown subcommand, or missing arguments, reply with the usage line and stop:

`/u:cr init | mk <name> <dir> | ls | rm <name> | cl`

## Rules for every subcommand

- **Always tmux.** Claude needs a real TTY and must outlive this session, so it only ever runs inside a tmux session. Never launch `claude` directly, or with `&`, `nohup`, or `script`.
- **Private server.** Every tmux command uses `tmux -L cr …`, so /u:cr never touches the user's other tmux sessions.
- **Child sessions stay sandboxed.** Every child Claude gets `--settings "${CLAUDE_PLUGIN_ROOT}/skills/cr/spawn-settings.json"`. That file keeps the sandbox on even if the target project turns it off. It also adds ask rules, so the child must ask the user before running anything outside the sandbox, even in auto mode: one for unsandboxed retries, and one for tmux, because a child started in a folder where `/u:cr init` ran inherits the `tmux *` exclusion and tmux can run anything. One gap remains: commands a target project lists in its own `sandbox.excludedCommands` are still approved by the auto-mode classifier.
- **Exact targets.** A plain `-t foo` also matches a session called `foobar`. Target sessions as `-t '=<name>'` (has-session, kill-session) and panes as `-t '=<name>:'` (send-keys, capture-pane).
- **One tmux command per Bash call.** The sandbox exclusion only applies when the whole call is the tmux command. Don't chain with `&&`, `;` or `|`. Don't add redirects like `2>/dev/null`, a `cd`, or `$(…)`. Run `sleep` as its own call.
- **Sandbox failures.** This session runs with the Bash sandbox on, which blocks starting claude, and `/u:cr init` fixes that. If a command other than init fails in a way that looks like the sandbox (`Operation not permitted`, `Permission denied` or `error connecting to` on the tmux socket under `/private/tmp/tmux-…`, sandbox violation notices), tell the user to run `/u:cr init`. Don't retry, and don't use `dangerouslyDisableSandbox`.
- **No-server errors.** Treat tmux's "no server running" / "no sessions" errors as "nothing exists", not as failures.
- **Judge panes by meaning.** Claude Code's on-screen wording changes between versions. When reading a tmux pane, decide what it shows by meaning, never by matching exact text. Quoted texts in this file are examples only.

## /u:cr init (idempotent, safe to rerun)

1. Run `command -v tmux`. If tmux is missing, tell the user to run `brew install tmux` and stop.
2. Add `"tmux *"` to `sandbox.excludedCommands` in `.claude/settings.local.json` in LAUNCH. LAUNCH is the primary working directory this Claude Code session was started in, from the environment info.
   - Format: `excludedCommands` is an array of strings using the same pattern syntax as `Bash(...)` permission rules. A pattern with no wildcard is an exact match, and `tmux *` matches tmux with or without arguments. This was checked against the Claude Code sandboxing docs (code.claude.com/docs/en/sandboxing). If that page now says otherwise, follow the page.
   - Use the Read and Edit/Write tools, not Bash, because the sandbox blocks shell writes to settings files. Read the file first and merge: keep every other key, keep any existing `sandbox` keys and `excludedCommands` entries, and don't add a duplicate. If the file doesn't exist, create it as `{"sandbox": {"excludedCommands": ["tmux *"]}}`. Keep it strict JSON, with no comments and no trailing commas.
   - If the entry is already there, change nothing.
3. Verify with two separate calls:
   - `tmux -L cr new-session -d -s cr-init-test 'sleep 5'`
   - `tmux -L cr kill-session -t '=cr-init-test'`

   If the first call fails, show the error. If it looks like the sandbox (see the rules above), the running session probably hasn't picked up the new setting yet. Tell the user to restart Claude Code in LAUNCH and rerun `/u:cr init`. If `/status` or the docs show the sandbox is admin-required, local settings files are ignored. In that case, offer to put the same entry in `~/.claude/settings.json` instead, and ask before writing it. If the error isn't sandbox-related, say what it points to.
4. Report what changed (file created, entry added) or "already set up", plus the verification result.

## /u:cr mk <name> <dir>

1. **Check the name.** It may contain only letters, digits, `-`, `_` and `/`. It can't start or end with `/` or contain `//`. tmux rejects `.` and `:`, and a leading `/` reads like a path or a slash command. If the name breaks a rule, say why and stop.

   Run `tmux -L cr has-session -t '=<name>'`. If it succeeds, say the session already exists and stop. A "no server" or "can't find session" error means the name is free.
2. **Resolve dir:**

   | dir looks like            | Resolves to            |
   |---------------------------|------------------------|
   | `~` or `~/x`              | `$HOME` / `$HOME/x`    |
   | `/x` (absolute)           | `/x`                   |
   | `.`, `..`, `./x`, `../x`  | `LAUNCH/…`             |
   | anything else, e.g. `dev/project` | `$HOME/dev/project` |

   Strip surrounding quotes, apply the table to get an absolute path, then check it with `ls -ldL '<resolved>'`. The `-L` follows symlinks, so a symlink to a folder counts as a folder.
   - A line starting with `d` means it's a directory. Use it.
   - If it's missing or is a file, don't start anything. Instead:
     - List the deepest parent that exists with `ls '<parent>'`. Suggest up to 5 close matches: case differences, typos 1–2 letters off, and prefix or partial matches.
     - Also check the other plausible reading. For example, if `dev/x` → `~/dev/x` is missing but `LAUNCH/dev/x` exists, offer that.
     - If nothing turns up, search a little wider: `find "$HOME" -maxdepth 4 -type d -iname '*<last path part>*'`. That command isn't pre-approved, so expect a prompt.
     - Show the matches as a numbered list, and add an option to create `<resolved>`. Wait for the user's answer.
     - If they choose to create it, run `mkdir -p '<resolved>'`. If the sandbox blocks that, run it through the tmux server instead: `tmux -L cr start-server ';' run-shell 'mkdir -p "<resolved>"'`. `start-server` is needed because `run-shell` fails when no tmux server is running yet. `run-shell` waits for `mkdir` to finish and returns an error if it fails. Then confirm with `ls -ldL '<resolved>'`.
3. **Start the session.** First run `ls '${CLAUDE_PLUGIN_ROOT}/skills/cr/spawn-settings.json'`. If the file is missing, say so and stop. Never start a child without it. Then start it with one call:

   `tmux -L cr new-session -d -x 200 -y 50 -s <name> -c '<resolved>' 'caffeinate -i claude --remote-control "<name>" --settings "${CLAUDE_PLUGIN_ROOT}/skills/cr/spawn-settings.json"'`

   `caffeinate -i` keeps the Mac from idle-sleeping while the session runs. `--settings` applies the sandbox rules described above.
4. **Get through startup dialogs.** Run `sleep 5`, then `tmux -L cr capture-pane -p -t '=<name>:'`. Judge the pane by meaning:
   - A question about trusting the folder → choose the option that trusts it.
   - A question about enabling or confirming Remote Control → choose the option that enables it.

   Send the keys it needs with `tmux -L cr send-keys -t '=<name>:' …`. That's `Enter`, or `Up`/`Down` and then `Enter`, with each send-keys as its own call. Then `sleep 2` and capture again. Repeat up to 4 times.

   If a dialog fits neither case, show it to the user and ask what to choose. Never pick an option that grants extra permissions, such as bypassing permission prompts, on your own.
5. **Check the result.** Read the pane and judge by meaning whether Claude is running and Remote Control is connected. Today this shows as an "/rc active" label in the bottom bar.
   - Clearly connected → reply in one line: `✅ <name> is up in <resolved>. Open it from the Code tab, or attach on the Mac: tmux -L cr attach -t '=<name>'`
   - Clearly an error → show the relevant lines, then run `tmux -L cr kill-session -t '=<name>'`.
   - Unsure → show the pane, and say the session is still running so the user can check the Code tab on their phone.

## /u:cr ls

Run `tmux -L cr ls -F '#S  #{pane_current_path}  started #{t:session_created}'` and show the output.

If tmux reports no server or no sessions, say "No sessions."

## /u:cr rm <name>

1. Run `tmux -L cr has-session -t '=<name>'`. If it fails, say there is no such session and stop.
2. Send `/exit` with two calls: `tmux -L cr send-keys -t '=<name>:' -l '/exit'`, then `tmux -L cr send-keys -t '=<name>:' Enter`. Then run `sleep 3`.
3. Run `has-session` again. If the session still exists, send `tmux -L cr send-keys -t '=<name>:' C-c`, `sleep 1`, the same C-c again, then `sleep 2`.
4. If it still exists, run `tmux -L cr kill-session -t '=<name>'`.
5. Confirm with `has-session` that it's gone, and say how it closed: /exit, Ctrl+C, or kill. Mention that the user can resume it later by running `claude --continue` in that folder.

## /u:cr cl

1. Run `tmux -L cr ls -F '#S'`. If there's no server or no sessions, say "No sessions." and stop.
2. Close each session the same way as `/u:cr rm`.
3. Run `tmux -L cr ls -F '#S'` again. If anything remains, run `tmux -L cr kill-server`.
4. List what was closed and how each one closed.
