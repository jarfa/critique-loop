# Critique Loop

A Claude Code plugin that enables two Claude Code instances to collaborate through turn-based dialogues with automatic notifications.

## Motivation
When planning or writing code I often spin up multiple instances of Claude Code and use one to critique the other. This has radically reduced the number of errors and poor design choices that made it into plans or code before my own final review.

However, it also required me to act as their intermediary and manually relay feedback back and forth between them.

Hence the need for a formal skill to simplify the process and take the human (mostly) out of the loop until the machines reach a conclusion.

You could use this for:  
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

## Roles
The `--template` flag (shown above) simply tells each instance what their role is. If those defaults don't work for you, you can also name the roles whatever you want by skipping that flag and using `--role`. For example
```
/critique-loop --topic bank-heist --role thief
```
And for the second instance:
```
/critique-loop --topic bank-heist --role defense-lawyer
```
Note that if you're using `--role`, you have to specify it for both instances, not just the first one.

## Configuration

| Environment Variable | Default | Purpose |
|---------------------|---------|---------|
| `DIALOGUE_DIR` | `~/.claude/dialogues` | Where dialogue files are stored |

## License

This project is licensed under the MIT License — see the LICENSE file for details.
