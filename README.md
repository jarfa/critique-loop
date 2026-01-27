# Dialogue Partner

A Claude Code plugin that enables two Claude Code instances to collaborate through turn-based dialogues with automatic notifications.

## Status

**Design complete, implementation pending.**

See [design document](docs/plans/2026-01-25-dialogue-partner-design.md) for the full specification.

## Use Cases

- **Planning/Critiquing** — One instance proposes, the other challenges assumptions
- **Writing/Reviewing** — One authors code or docs, the other reviews
- **Pair Programming** — Instances in separate repos coordinate through dialogue

## Requirements

- **tmux** — Required for automatic turn notifications
- **fswatch** — File watching on macOS (`brew install fswatch`)

## Installation

**From GitHub:**
```bash
# Add the marketplace
/plugin marketplace add jonathanarfa/dialogue-partner

# Install the plugin
/plugin install dialogue-partner@jonathanarfa
```

**From local source (for development):**
```bash
claude --plugin-dir /path/to/dialogue-partner
```

## Usage

In the first tmux pane:
```
/dialogue-partner --template planning --topic api-design
```

In the second tmux pane:
```
/dialogue-partner --topic api-design
```

The second instance automatically joins as the complementary role.

## Templates

| Template | Roles | Use case |
|----------|-------|----------|
| `planning` | proposer, critic | Planning sessions |
| `review` | author, reviewer | Code/doc reviews |
| `pair` | lead, partner | Pair programming |

Or use arbitrary role names with `--role <name>`.

## Configuration

| Environment Variable | Default | Purpose |
|---------------------|---------|---------|
| `DIALOGUE_DIR` | `~/.claude/dialogues` | Where dialogue files are stored |

## License

MIT
