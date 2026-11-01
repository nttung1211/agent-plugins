---
name: cr
description: Create, list, and remove Claude Code Remote Control sessions in other folders via tmux.
argument-hint: "mk <name> <dir> [c] | ls | rm <name> | cl"
disable-model-invocation: true
allowed-tools: Bash(tmux:*), Bash(sleep:*), Bash(command -v:*), Bash(ls:*), Bash(mkdir:*)
---

# /u:cr — Remote Control sessions in tmux

Raw input: `$ARGUMENTS`

Split the input on whitespace. The first word is the subcommand; the rest are its arguments. With no arguments, an unknown subcommand, or the wrong number of arguments, reply with the usage line and stop:

`/u:cr mk <name> <dir> [c] | ls | rm <name> | cl`

A **cr session** is a session that `/u:cr mk` started: a normal `claude --remote-control` session running inside tmux on the private `cr` server. It has the same settings and permission mode as any session the user starts themselves, and it can run `/u:cr` too.

## Rules for every subcommand

- **Check tmux first.** Run `command -v tmux`. If it prints nothing, tell the user to run `brew install tmux` and stop.
- **Always tmux.** Claude needs a real TTY and must outlive this session, so it only ever runs inside a tmux session. Never launch `claude` directly, or with `&`, `nohup`, or `script`.
- **Private server.** Every tmux command uses `tmux -L cr …`, so /u:cr never touches the user's other tmux sessions.
- **Run tmux outside the sandbox.** The Bash sandbox blocks the tmux socket, so run every tmux command with `dangerouslyDisableSandbox: true` from the first try.
- **One tmux command per Bash call.** Don't chain with `&&`, `;` or `|`, and run `sleep` as its own call, so each step is easy to follow.
- **Exact targets.** A plain `-t foo` also matches a session called `foobar`. Target sessions as `-t '=<name>'` (has-session, kill-session) and panes as `-t '=<name>:'` (send-keys, capture-pane).
- **No-server errors.** Treat tmux's "no server running" / "can't find session" errors as "nothing exists", not as failures.
- **This session.** `rm` and `cl` need to know whether the session running this skill is itself a cr session. Run `echo "$TMUX|$TMUX_PANE"`. If the part before `|` contains `/cr,`, this session runs on the cr server: get its name with `tmux -L cr display-message -p -t "$TMUX_PANE" '#S'`. Otherwise this session isn't a cr session.
- **Judge panes by meaning.** Claude Code's on-screen wording changes between versions. When reading a tmux pane, decide what it shows by meaning, never by matching exact text. Quoted texts in this file are examples only.

## /u:cr mk <name> <dir> [c]

The optional third word `c` means "create the folder if it doesn't exist". Any other third word, or more than three words, gets the usage line.

1. **Check the name.** It may contain only letters, digits, `-`, `_` and `/`. It can't start or end with `/` or contain `//`. tmux rejects `.` and `:`, and a leading `/` reads like a path or a slash command. If the name breaks a rule, say why and stop.

   Run `tmux -L cr has-session -t '=<name>'`. If it succeeds, say the session already exists and stop. A "no server" or "can't find session" error means the name is free.
2. **Resolve dir.** LAUNCH is the primary working directory this Claude Code session was started in, from the environment info.

   | dir looks like            | Resolves to            |
   |---------------------------|------------------------|
   | `~` or `~/x`              | `$HOME` / `$HOME/x`    |
   | `/x` (absolute)           | `/x`                   |
   | `.`, `..`, `./x`, `../x`  | `LAUNCH/…`             |
   | anything else, e.g. `dev/project` | `$HOME/dev/project` |

   Strip surrounding quotes, apply the table to get an absolute path, then check it with `ls -ldL '<resolved>'`. The `-L` follows symlinks, so a symlink to a folder counts as a folder.
   - A line starting with `d` means it's a directory. Use it. A `c` changes nothing here.
   - If it's missing and `c` was given, create it right away, as described under **Create the folder**, with no suggestions and no question.
   - If it exists but is a file and `c` was given, say so and stop. Never overwrite or rename a file.
   - Otherwise, when it's missing or is a file, don't start anything. Instead:
     - List the deepest parent that exists with `ls '<parent>'`. Suggest up to 5 close matches: case differences, typos 1–2 letters off, and prefix or partial matches.
     - Also check the other plausible reading. For example, if `dev/x` → `~/dev/x` is missing but `LAUNCH/dev/x` exists, offer that.
     - If nothing turns up, search a little wider: `find "$HOME" -maxdepth 4 -type d -iname '*<last path part>*'`.
     - Show the matches as a numbered list, and add an option to create `<resolved>`. Wait for the user's answer.
     - If they choose to create it, create it as described under **Create the folder**.
   - **Create the folder:** run `mkdir -p '<resolved>'`. If the sandbox blocks it (`Operation not permitted`), run the same command again with `dangerouslyDisableSandbox: true`. Then confirm with `ls -ldL '<resolved>'`. If it still isn't a directory, show the error and stop.
3. **Start the session** with one call:

   `tmux -L cr new-session -d -x 200 -y 50 -s '<name>' -c '<resolved>' 'caffeinate -i claude --remote-control "<name>"'`

   `caffeinate -i` keeps the Mac from idle-sleeping while the session runs.
4. **Get through startup dialogs.** Run `sleep 5`, then `tmux -L cr capture-pane -p -t '=<name>:'`. Judge the pane by meaning:
   - A question about trusting the folder → choose the option that trusts it.
   - A question about enabling or confirming Remote Control → choose the option that enables it.

   Send the keys it needs with `tmux -L cr send-keys -t '=<name>:' …`. That's `Enter`, or `Up`/`Down` and then `Enter`, with each send-keys as its own call. Then `sleep 2` and capture again. Repeat up to 4 times.

   If a dialog fits neither case, show it to the user and ask what to choose. Never pick an option that grants extra permissions, such as bypassing permission prompts, on your own.
5. **Check the result.** Read the pane and judge by meaning whether Claude is running and Remote Control is connected. Today this shows as an "/rc active" label in the bottom bar.
   - Clearly connected → reply in one line: `✅ <name> is up in <resolved>. Open it from the Code tab, or attach on the Mac: tmux -L cr attach -t '=<name>'`. If this run created the folder, write `(new folder)` right after `<resolved>`.
   - Clearly an error → show the relevant lines, then run `tmux -L cr kill-session -t '=<name>'`.
   - Unsure → show the pane, and say the session is still running so the user can check the Code tab on their phone.

## /u:cr ls

Run `tmux -L cr ls -F '#S  #{pane_current_path}  started #{t:session_created}'` and show the output.

If tmux reports no server or no sessions, say "No sessions."

## /u:cr rm <name>

1. If `<name>` is this session (see **This session**), say: "Can't close the session you're typing in. Close it from another session, or attach on the Mac and type /exit." Then stop.
2. Run `tmux -L cr has-session -t '=<name>'`. If it fails, say there is no such session and stop.
3. Send `/exit` with two calls: `tmux -L cr send-keys -t '=<name>:' -l '/exit'`, then `tmux -L cr send-keys -t '=<name>:' Enter`. Then run `sleep 3`.
4. Run `has-session` again. If the session still exists, send `tmux -L cr send-keys -t '=<name>:' C-c`, `sleep 1`, the same C-c again, then `sleep 2`.
5. If it still exists, run `tmux -L cr kill-session -t '=<name>'`.
6. Confirm with `has-session` that it's gone, and say how it closed: /exit, Ctrl+C, or kill. Mention that the user can resume it later by running `claude --continue` in that folder.

## /u:cr cl

1. Run `tmux -L cr ls -F '#S'`. Leave out this session if it's a cr session (see **This session**), without mentioning it. If nothing is left, say "No sessions." and stop.
2. Close each remaining session the same way as `/u:cr rm`.
3. Run `tmux -L cr ls -F '#S'` again. Kill anything other than this session that remains with `tmux -L cr kill-session -t '=<name>'`.
4. List what was closed and how each one closed.
