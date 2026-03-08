# Critique Loop

A Claude Code plugin that starts a dialogue between two subagents to critique & improve your plans or code.

## Motivation

When planning or writing code I often want Claude Code to critique its own work. Previously this required manually running two instances and relaying messages between them.

This plugin automates the process: invoke once, and a two subagents are spawned to provide critique, review, or collaboration. The human stays out of the loop until the dialogue reaches a conclusion.

Use this for:
- **Planning/Critiquing** — One agent proposes, the partner challenges assumptions
- **Writing/Reviewing** — One agent presents code or docs, the partner reviews
- **Pair Programming** — Collaborative problem-solving between two partners

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

Just describe what you want in plain text:

```
/critique-loop plan my new caching strategy
```

```
/critique-loop review the auth module with codex
```

The plugin infers appropriate roles (e.g., "proposer" / "critic" for planning, "author" / "reviewer" for reviews) and starts the dialogue automatically. Say something like "with codex" in your description to use Codex as the second partner instead of Claude.

## How It Works

1. You describe what you want to discuss in plain text
2. The orchestrator infers roles, topic, and partner from your description
3. You provide additional context when prompted
4. Two subagents are spawned to start the critique-loop
5. Turns alternate until conclusion (DONE), intervention needed (STUCK), or max rounds reached
6. You receive the summary and a reminder of where to find the full transcript

## Configuration

| Environment Variable | Default | Purpose |
|---------------------|---------|---------|
| `DIALOGUE_DIR` | `.dialogues` | Where dialogue files are stored (project-local) |

## Future Improvements

The dialogue file currently serves as the communication medium between the subagents. A potential optimization: direct communication where the orchestrator passes context in the resume prompt and the partner returns its response directly, eliminating file I/O overhead. The dialogue file would then become optional for debugging purposes.

## License

This project is licensed under the MIT License — see the LICENSE file for details.
