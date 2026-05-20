# claude-discord-fork

Local fork of `claude-plugins-official/discord` (v0.0.4) with Discord-side **slash command support** for remotely operating Claude Code.

## What's added vs upstream

- **`/reload` slash command**: types Claude Code's native `/reload-plugins` into the host tmux pane. Lightweight — no kill/respawn, current session continues uninterrupted, MCP/skill/agent registries refresh in place.
- Slash commands are registered per-guild on `ready` for instant availability (falls back to global registration if the bot is in no guilds).
- Auth: same `access.allowFrom` allowlist as the existing button handler.
- Architecture: detached `bash` pipeline via `spawn({ detached: true, stdio: 'ignore' })` so the `tmux send-keys` survives this plugin process if anything kills it.

## Env vars

| Var | Default | Purpose |
|---|---|---|
| `DISCORD_BOT_TOKEN` | (from `~/.claude/channels/discord/.env`) | Same as upstream |
| `CLAUDE_TMUX_TARGET` | `claude_main` | tmux `session:window` (or `session:window.pane`) to send restart keys to |
| `CLAUDE_PROJECT_KEY` | `-home-aiuser-project` | folder name under `~/.claude/projects/` to read most recent session id from |

## Switchover steps (one-time)

```bash
# 1. Remove upstream registration
claude mcp remove discord

# 2. Add fork
claude mcp add discord -- bun run --cwd /home/aiuser/project/claude-discord-fork start

# 3. Restart Claude Code so the new MCP loads
#    (this final restart is unavoidable — after this, /reload from Discord works)
```

After restart, Discord will see a new `/reload` slash command. Type `/` in any allowed channel to see it.

## Testing checklist

1. After CC restart, in a Discord channel where you're allowlisted:
   - Type `/` — autocomplete should suggest `/reload`
   - Run `/reload` — should reply `♻️ Restarting Claude Code with --continue <sid>... Target tmux pane: claude_main`
2. Watch your tmux pane `claude_main`:
   - ~1.5s later, `C-c` is sent (current CC dies)
   - 1s after that, `claude --continue <sid>` is typed and runs
   - New CC starts, full context preserved
3. After new CC ready, the Discord plugin reconnects automatically as a child of the new CC. Slash command remains available.

## Failure modes

- **`tmux` not running with target pane**: `tmux send-keys` fails silently, CC stays alive. Check `tmux ls` for active session name.
- **Multiple jsonl files with same project key, picks the wrong one**: `latestSessionId()` uses mtime; if a stale session got touched, fix by setting `CLAUDE_PROJECT_KEY` explicitly or pass session id manually.
- **Allowlist edits don't propagate to slash command**: slash command auth reads `access.allowFrom` live (good — `loadAccess()` re-reads every call), no restart needed for permission changes.

## Reverting to upstream

```bash
claude mcp remove discord
claude mcp add discord -- bun run --cwd ~/.claude/plugins/cache/claude-plugins-official/discord/0.0.4 start
# restart CC
```

## Future extensions

The `interactionCreate(isChatInputCommand)` handler is structured as a switch on `interaction.commandName`. To add commands, append to `SLASH_COMMANDS` array + add a branch.

Candidates that howish raised:
- `/status` — print current session id, uptime, MCP server health
- `/sync-settings` — touch `~/.claude/settings.json` to fire CC's native settings hot-reload
- `/restart-mcp <name>` — send signal to specific MCP child without full CC restart (requires CC support — currently doesn't exist)
