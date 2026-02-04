# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Critique Loop is a Claude Code plugin that enables structured, turn-based dialogues between two spawned subagents. The user's Claude instance orchestrates but never writes dialogue turns—keeping all dialogue output hidden from the terminal. Designed for planning sessions, code reviews, and pair programming where automated critique is valuable.

**Status:** Implementation complete.

## Architecture

### Core Components

1. **Orchestrator Skill** (`plugins/critique-loop/skills/critique-loop/skill.md`) - Coordinates the dialogue: gathers user context, creates dialogue file header, spawns two subagents, manages turn-taking via Task tool resume, reports outcomes. Never writes dialogue turns directly.

2. **Participant Instructions** (`plugins/critique-loop/dialogue-partner-instructions.md`) - Standalone protocol rules both subagents follow. Kept separate from orchestrator to ensure unbiased dialogue.

3. **Dialogue Files** (`.dialogues/<topic>.md`) - Append-only markdown files storing the conversation. Format:
   ```markdown
   # Dialogue: <topic>
   Started: YYYY-MM-DD HH:MM
   Template: planning|review|pair
   Max rounds: 5

   Participants:
     - <role1> @ <directory>
     - <role2> (partner)

   ---

   ## [<role>] Round N — YYYY-MM-DD HH:MM
   <message>
   Status: AWAITING <other-role>|PROPOSING_DONE|DONE|STUCK
   ```

### Coordination Model

- **Two-subagent architecture:** Orchestrator spawns Subagent A (user's role) and Subagent B (partner role); both write turns via Write tool in subagent context (hidden from terminal)
- **Task tool orchestration:** Orchestrator spawns, resumes, and coordinates subagents; subagents return after writing each turn
- **Append-only protocol:** Prevents conflicting edits; turns are immutable
- **Status-driven flow:** Last line of each turn declares who acts next
- **Unbiased partner:** Subagent B receives no user context—reads dialogue file fresh for objective critique

### Templates

| Template | Subagent A | Subagent B | Default Rounds |
|----------|------------|------------|----------------|
| `planning` | proposer | critic | 5 |
| `review` | author | reviewer | 5 |
| `pair` | lead | partner | 7 |

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

Verify:
- User sees setup prompts (context gathering, gitignore)
- User sees "Starting dialogue. Spawning proposer and critic..."
- NO Write/Read tool output during dialogue loop (hidden in subagents)
- User sees final summary when done
- Dialogue file contains correct turn history

### Key Files

| Path | Purpose |
|------|---------|
| `plugins/critique-loop/skills/critique-loop/skill.md` | Orchestrator skill (spawns/resumes subagents) |
| `plugins/critique-loop/dialogue-partner-instructions.md` | Protocol rules for both subagents |
| `plugins/critique-loop/.claude-plugin/plugin.json` | Plugin metadata |

## Protocol Rules

When modifying the skill, preserve these invariants:

1. **Append only** - Never edit previous turns
2. **Status is always last** - Final line of each turn must be `Status: <value>`
3. **Use Write tool** - All dialogue file writes use Write (read, append, write back)
4. **Round tracking** - Increment round when both parties have spoken
5. **Orchestrator never writes turns** - Only creates header and coordinates subagents
6. **Unbiased Subagent B** - Partner reads file itself, orchestrator doesn't summarize content to it
