# Critique Loop

A Claude Code plugin that enables two Claude Code instances to collaborate through turn-based dialogues with automatic notifications.

## Use Cases

- **Planning/Critiquing** — One instance proposes, the other challenges assumptions
- **Writing/Reviewing** — One authors code or docs, the other reviews
- **Pair Programming** — Instances in separate repos coordinate through dialogue

## Installation

**From GitHub:**
```bash
# Add the marketplace
/plugin marketplace add jarfa/critique-loop

# Install the plugin
/plugin install critique-loop@jarfa
```

**From local source (for development):**
```bash
claude --plugin-dir /path/to/critique-loop
```

## Usage

In the first claude code instance:
```
/critique-loop --template planning --topic api-design
```

In the second claude code instance:
```
/critique-loop --topic api-design
```

The second instance automatically joins as the complementary role.

## Templates

| Template | Roles | Use case |
|----------|-------|----------|
| `planning` | proposer, critic | Planning sessions |
| `review` | author, reviewer | Code/doc reviews |
| `pair` | lead, partner | Pair programming |

Or use arbitrary role names with `--role <name>`. These roles could be descriptive to hint at their role ('speechwriter', 'politician') or not ('alice', 'bob') If you're using `--role`, you need to specify that argument for both claude code instances (unlike `--template` which lets the 2nd instance figure it out from the shared document).
## Configuration

| Environment Variable | Default | Purpose |
|---------------------|---------|---------|
| `DIALOGUE_DIR` | `~/.claude/dialogues` | Where dialogue files are stored |

## License

MIT
