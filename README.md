# agent-plugins

nttung1211's Claude Code plugins.

## Add this marketplace (once)

In Claude Code:

```
/plugin marketplace add nttung1211/agent-plugins
```

Then install any plugin below with `/plugin install <plugin>@nttung1211`.

## Plugins

| Plugin | Install | What it does |
|---|---|---|
| [u](plugins/u) | `/plugin install u@nttung1211` | `/u:cm` runs a shell command in a chosen folder. `/u:cr` starts Claude sessions in other folders that you can drive from your phone. |

## Add a new plugin (for the maintainer)

1. Create `plugins/<name>/` with `.claude-plugin/plugin.json` (`"name": "<name>"`) and its `skills/`.
2. Add an entry to `.claude-plugin/marketplace.json`:
   ```json
   { "name": "<name>", "source": "./plugins/<name>", "description": "…" }
   ```
3. Add a row to the table above.
4. Run `claude plugin validate .`, then commit and push.

To release a change to a plugin, bump `version` in its `plugin.json`.
Installed copies only update when the version changes.

## License

MIT
