# u — small Claude Code helpers

A Claude Code plugin with two commands:

- **`/u:cm`**: run a shell command in a chosen folder
- **`/u:cr`**: start Claude sessions in other folders that you can drive from your phone

## Install

In Claude Code:

```
/plugin marketplace add nttung1211/agent-plugins
/plugin install u@nttung1211
```

---

## /u:cm — run a command in a folder

```
/u:cm <command> [$in <folder>]
```

Examples:

```
/u:cm git status                  # in the folder Claude was started in
/u:cm git remote -v $in dev/api   # in ~/dev/api
/u:cm ls $in ./notes              # in ./notes, next to where Claude was started
```

**Folders:**

| You type | It means |
|---|---|
| nothing | the folder Claude was started in |
| `~/x` or `/full/path` | exactly that |
| `./x` or `../x` | relative to the folder Claude was started in |
| `dev/x` (no prefix) | `~/dev/x` |

If the folder or the command looks wrong, `/u:cm` suggests a fix and asks before running.
Risky commands (like `rm -r` or `git push --force`) always ask first.

---

## /u:cr — Claude sessions you can drive from your phone

`/u:cr` starts Claude Code sessions in other folders, each with Remote Control turned on.
Each one runs inside tmux, so it keeps running after you close the window,
and you can use it from the **Code tab in the Claude app** or attach to it on the Mac.

### Setup

Install tmux once:

```sh
brew install tmux
```

That's all. `/u:cr` works in any folder, including from inside a session it started,
so you can create new sessions from your phone.

### Commands

| Command | What it does |
|---|---|
| `/u:cr mk <name> <dir> [c]` | Start a new session called `<name>` in `<dir>`. Add `c` to create the folder if it doesn't exist |
| `/u:cr ls` | List running sessions |
| `/u:cr rm <name>` | Close one session (not the one you're typing in) |
| `/u:cr cl` | Close all sessions, except the one you're typing in |

Examples:

```
/u:cr mk api ~/dev/api      # folder in your home
/u:cr mk notes ./notes      # folder next to where Claude was started
/u:cr mk draft dev/test/writing/cq c   # create ~/dev/test/writing/cq if missing
/u:cr rm api
```

**Names** can use letters, digits, `-`, `_` and `/` (like `api/v2`), but can't start or end with `/`. Use the exact name every time.

**Folders** follow the same rules as `/u:cm`. A folder is required.
If it doesn't exist, `/u:cr` suggests similar folders or offers to create it.
With `c` at the end, it creates the folder straight away, without asking.

### Use a session

**On your phone:** open the Claude app → **Code** tab → pick the session by name.

**On the Mac:** open Terminal and run:

```sh
tmux -L cr attach -t '=<name>'
```

| To… | Press |
|---|---|
| Leave without stopping it | `Ctrl+b`, let go, then `d` |
| Scroll back | `Ctrl+b`, then `[` (press `q` to stop) |
| Send Ctrl+b to Claude | `Ctrl+b` twice |

Closing the Terminal window also just leaves; the session keeps running.

To see sessions from Terminal: `tmux -L cr ls`

### Permissions

Sessions started by `/u:cr` are normal Claude Code sessions with Remote Control on.
They use your usual settings and permission mode, just like a session you open yourself.

The Bash sandbox blocks tmux, so `/u:cr` runs its tmux commands outside the sandbox.
That goes through your normal permission check: in auto mode it runs without stopping you,
in manual mode you may be asked to approve.

### Resume a closed session

After `/u:cr rm`, you can pick the conversation back up in that folder:

```sh
cd <dir>
claude --continue
```

### Troubleshooting

| Problem | Fix |
|---|---|
| Error mentioning `Operation not permitted` or `/private/tmp/tmux-…` | A tmux command ran inside the sandbox. Approve it to run outside the sandbox, or use auto mode |
| `/u:cr` says tmux is missing | `brew install tmux` |
| Session not on your phone | Run `/u:cr ls` to check it's running, then attach on the Mac to see what's on screen |
| `attach` says "no sessions" | Did you include `-L cr`? It's required |
| zsh says `t1 not found` | Keep the quotes: `-t '=t1'` |

### Notes for maintainers

**tmux runs outside the sandbox.** Every tmux call in the skill uses `dangerouslyDisableSandbox: true`,
because the sandbox blocks the tmux socket under `/private/tmp/tmux-<uid>/`.
Earlier versions (0.1.x) used a per-folder `sandbox.excludedCommands` entry from `/u:cr init` instead;
0.2.0 dropped that, so there's no setup per folder.

**Sending user text into a session.** `/u:cr` only sends fixed keys today (`/exit`, `Enter`, `Up`/`Down`, `C-c`).
If you add a feature that sends free text (for example `/u:cr send <name> <msg>`), don't put the text
in `send-keys`, where quotes and backticks get messy. Write it to a temp file and use
`tmux -L cr load-buffer <file>` + `tmux -L cr paste-buffer -t '=<name>:'`.
