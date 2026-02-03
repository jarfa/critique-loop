# Critique Loop

A Claude Code plugin that enables a single Claude Code instance to collaborate with a spawned dialogue partner through turn-based dialogues.

## Motivation

When planning or writing code I often want Claude Code to critique its own work. Previously this required manually running two instances and relaying messages between them.

This plugin automates the process: invoke once, and a dialogue partner is spawned to provide critique, review, or collaboration. The human stays out of the loop until the dialogue reaches a conclusion.

Use this for:
- **Planning/Critiquing** — You propose, the partner challenges assumptions
- **Writing/Reviewing** — You author code or docs, the partner reviews
- **Pair Programming** — Collaborative problem-solving with a partner

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

```
/critique-loop --topic api-design --template planning
```

Or with custom roles:

```
/critique-loop --topic bank-heist --role1 thief --role2 getaway-driver
```

The dialogue runs automatically. You'll see status updates and the final summary when complete.

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `--topic <topic>` | Yes | Dialogue identifier |
| `--template <template>` | Either this... | Predefined role pair |
| `--role1 <role>` + `--role2 <role>` | ...or both of these | Custom role names |
| `--max-rounds <N>` | No | Maximum rounds (default: 20) |

## Templates

| Template | Your Role | Partner Role | Use case |
|----------|-----------|--------------|----------|
| `planning` | proposer | critic | Planning sessions |
| `review` | author | reviewer | Code/doc reviews |
| `pair` | lead | partner | Pair programming |

## How It Works

1. You invoke the skill with a topic and template (or custom roles)
2. You provide context for what you want to discuss
3. The skill creates a dialogue file and writes your opening turn
4. A dialogue partner is spawned to respond
5. Turns alternate until conclusion (DONE), intervention needed (STUCK), or max rounds reached
6. You receive the summary and a reminder of where to find the full transcript

## Configuration

| Environment Variable | Default | Purpose |
|---------------------|---------|---------|
| `DIALOGUE_DIR` | `.dialogues` | Where dialogue files are stored (project-local) |

## Future Improvements

The dialogue file currently serves as the communication medium between parent and partner. A potential optimization: direct communication where the parent passes context in the resume prompt and the partner returns its response directly, eliminating file I/O overhead. The dialogue file would then become optional (e.g., `--log-dialogue` flag) for debugging purposes.

## License

This project is licensed under the MIT License — see the LICENSE file for details.
