---
name: cm
description: Run a shell command in a chosen directory. Usage /u:cm <command> [$in <dir>]. Only use when the user types /u:cm.
argument-hint: <command> [$in <dir>]
disable-model-invocation: true
allowed-tools: Bash
---

# /u:cm — run a command in a directory

Raw input: `$ARGUMENTS`

## 1. Split the input

- Find the **last** standalone token `$in` (surrounded by whitespace or at the end).
  - Everything before it, trimmed → **COMMAND**.
  - Everything after it, trimmed → **DIR** (may contain spaces; strip surrounding quotes).
- No `$in` token → the whole input is COMMAND and DIR is empty.
- `$in` present but nothing after it → treat DIR as empty.
- COMMAND empty → reply with the usage line and stop:
  `/u:cm <command> [$in <dir>]` — e.g. `/u:cm git remote -v $in dev`

## 2. Resolve DIR

LAUNCH = the primary working directory this Claude Code session was started in (from the environment info), not wherever a previous command left the shell.

| DIR looks like            | Resolves to            |
|---------------------------|------------------------|
| empty                     | LAUNCH                 |
| `~` or `~/x`              | `$HOME` / `$HOME/x`    |
| `/x` (absolute)           | `/x`                   |
| `.`, `..`, `./x`, `../x`  | `LAUNCH/…`             |
| anything else, e.g. `dev/project` | `$HOME/dev/project` |

Do it with one Bash call so the rules are applied exactly:

```bash
LAUNCH='<LAUNCH>'; D='<DIR>'
case "$D" in
  "")                  R="$LAUNCH" ;;
  "~")                 R="$HOME" ;;
  "~/"*)               R="$HOME/${D#\~/}" ;;
  /*)                  R="$D" ;;
  .|..|./*|../*)       R="$LAUNCH/$D" ;;
  *)                   R="$HOME/$D" ;;
esac
if [ -d "$R" ]; then echo "OK $(cd "$R" && pwd -P)"; else echo "MISSING $R"; fi
```

## 3. If the directory is wrong

If the result is `MISSING`, or the path exists but is a file, do **not** run anything. Instead:

1. Walk up to the deepest parent that exists and list its subfolders.
2. Pick up to 5 close matches: case differences, typos (1–2 letters off), prefix matches.
3. Also check the other plausible reading — e.g. if `dev/x` → `~/dev/x` is missing but `LAUNCH/dev/x` exists, offer that.
4. If still nothing, search a little wider: `find "$HOME" -maxdepth 4 -type d -iname '*<last path part>*' 2>/dev/null | head -5`.

Show the matches as a short numbered list and ask which one to use. Once the user picks, continue from step 4 with that directory.

## 4. Check the command before running

- Check the program exists: `command -v <first word>`. Run this check inside the resolved directory so local scripts like `./build.sh` are found.
  If not found, suggest close names (e.g. `compgen -c | grep -i '<partial>' | sort -u | head -5`, plus obvious typo fixes like `gti` → `git`). Note that shell aliases from the user's rc files may not exist in this non-interactive shell.
- If you can clearly see a mistake in a subcommand or flag (e.g. `git remove -v` → `git remote -v`), don't run it — show the corrected command and ask.
- In both cases: show **Suggested:** `<fixed command>` in `<dir>`, and run it only after the user says yes.

## 5. Risky commands

Run normal commands immediately. Pause and ask for a yes first if the command could destroy data or is hard to undo, for example:

- `rm -r`/`rm -rf`, `rm` with wildcards, `find … -delete`, `shred`, `dd`, `mkfs`
- `git push --force` / `-f`, `git reset --hard`, `git clean -fd`, `git checkout .`/`git restore .`, `git branch -D`, `git stash clear`
- `chmod -R`/`chown -R`, `sudo …`, `kill -9`, `killall`
- Database drops/truncates, `docker system prune`, `docker volume rm`
- Writes over existing files with `>`

Show exactly what will run and where, then wait.

## 6. Run it

Run in a subshell so the session's working directory doesn't change:

```bash
( cd '<resolved dir>' && <COMMAND> )
```

Pass COMMAND through exactly as typed (pipes, quotes, `&&`, redirections all intact). For long-running commands use a generous timeout or run in the background and say so.

## 7. Report

Keep it short:

- First line: `<COMMAND>` in `<resolved dir>`.
- The output (trim very long output, keeping the start and the end, and say it was trimmed).
- If the exit code is non-zero: give the code, a one-line read of the error, and — if the error points to a fixable mistake (unknown subcommand, bad flag, "did you mean …") — a **Suggested:** corrected command, asking before running it.
