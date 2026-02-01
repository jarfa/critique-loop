# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Critique Loop is a Claude Code plugin that enables a single Claude Code instance to collaborate with a spawned dialogue partner through structured, turn-based dialogues. Designed for planning sessions, code reviews, and pair programming where automated critique is valuable.

**Status:** Implementation complete.

## Architecture

### Core Components

1. **Parent Skill** (`plugins/critique-loop/skills/critique-loop/skill.md`) - Orchestrates the dialogue: gathers user context, creates dialogue file, spawns partner, manages turn-taking via Task tool resume, reports outcomes.

2. **Partner Instructions** (`plugins/critique-loop/dialogue-partner-instructions.md`) - Standalone protocol rules the spawned partner follows. Kept separate to avoid biasing the partner with parent's framing.

3. **Dialogue Files** (`~/.claude/dialogues/<topic>.md`) - Append-only markdown files storing the conversation. Format:
   ```markdown
   # Dialogue: <topic>
   Started: YYYY-MM-DD HH:MM
   Template: planning|review|pair
   Max rounds: 20

   Participants:
     - <role1> @ <directory>
     - <role2> (partner)

   ---

   ## [<role>] Round N — YYYY-MM-DD HH:MM
   <message>
   Status: AWAITING <other-role>|PROPOSING_DONE|DONE|STUCK
   ```

### Coordination Model

- **Task tool orchestration:** Parent spawns partner, partner returns after writing turn, parent resumes partner for next turn
- **Append-only protocol:** Prevents conflicting edits; turns are immutable
- **Status-driven flow:** Last line of each turn declares who acts next
- **Neutral partner spawning:** Partner reads dialogue file itself to avoid bias

### Templates

| Template | Parent Role | Partner Role | Default Rounds |
|----------|-------------|--------------|----------------|
| `planning` | proposer | critic | 20 |
| `review` | author | reviewer | 20 |
| `pair` | lead | partner | 30 |

## Development

### Local Testing

Run Claude Code with the plugin:
```bash
claude --plugin-dir plugins/critique-loop
```

Test:
```bash
/critique-loop --template planning --topic test-topic
```

### Key Files

| Path | Purpose |
|------|---------|
| `plugins/critique-loop/skills/critique-loop/skill.md` | Parent skill orchestration |
| `plugins/critique-loop/dialogue-partner-instructions.md` | Partner protocol rules |
| `plugins/critique-loop/.claude-plugin/plugin.json` | Plugin metadata |

## Protocol Rules

When modifying the skill, preserve these invariants:

1. **Append only** - Never edit previous turns
2. **Status is always last** - Final line of each turn must be `Status: <value>`
3. **Use Edit tool** - All dialogue file writes use Edit, not Bash commands (except initial `mkdir -p`)
4. **Round tracking** - Increment round when both parties have spoken
5. **Neutral spawning** - Partner reads file itself, parent doesn't summarize content
