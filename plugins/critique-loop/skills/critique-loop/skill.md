---
name: critique-loop
description: "Start a dialogue with a spawned partner. Enables planning, reviewing, and pair programming with automated turn-taking."
---

# Critique Loop

You orchestrate a structured dialogue between two spawned subagents. Follow this protocol exactly.

## Parse Arguments

Extract from the invocation:
- `--topic <topic>` (REQUIRED): The dialogue identifier
- `--template <template>`: Predefined role pair (`planning`, `review`, or `pair`)
- `--role1 <role>` + `--role2 <role>`: Custom role names (both required if not using template)
- `--max-rounds <N>`: Maximum rounds (default: 5, or template default)
- `--partner <type>`: Agent for Subagent B. Options: `claude` (default), `codex`

**Validation:**
- `--topic` is required. If missing, ask the user for it.
- Must provide EITHER `--template` OR BOTH `--role1` and `--role2`
- If `--partner codex`: run `codex --version` via the **Bash tool** to verify it's installed. If the command fails, tell the user: "Codex CLI not found. Install it or use `--partner claude` (default)." and stop.
- If validation fails, explain the requirement and ask for correct arguments

## Templates

If `--template` was provided, use these role pairs:

| Template | Role A | Role B | Max Rounds |
|----------|--------|--------|------------|
| `planning` | proposer | critic | 5 |
| `review` | author | reviewer | 5 |
| `pair` | lead | partner | 7 |

If using custom roles: Subagent A takes `--role1`, Subagent B takes `--role2`.

**Note:** When using custom roles, avoid role names that are substrings of each other (e.g., don't use 'lead' and 'team-lead' in the same dialogue).

## Configuration

```
DIALOGUE_DIR="${DIALOGUE_DIR:-.dialogues}"
DIALOGUE_FILE="${DIALOGUE_DIR}/<topic>.md"
INSTRUCTIONS_FILE="<path-to-this-plugin>/dialogue-partner-instructions.md"
```

---

## Phase 1: Setup

### Step 1: Gather context

Ask the user: "What would you like to discuss or accomplish in this dialogue?"

Wait for their response. Before proceeding:
- If the user references prior conversation ("the idea we discussed", "that feature", etc.), expand it into a self-contained description using your conversation history
- Include relevant background: problem context, constraints, decisions already made
- The goal is for Subagent A to understand the task without access to your conversation history

This expanded context becomes the user context passed to Subagent A.

### Step 2: Check gitignore

Check if `.dialogues/` is in the project's `.gitignore`. If not, ask the user: "Would you like me to add `.dialogues/` to your .gitignore? Dialogue files are typically not committed."

If yes, append `.dialogues/` to `.gitignore` (create the file if it doesn't exist).

### Step 3: Determine roles

- If `--template` provided: Use roles from template table
- If `--role1`/`--role2` provided: Use those role names

Role A = first role (from template or `--role1`)
Role B = second role (from template or `--role2`)

### Step 4: Create the dialogue file

Use the **Write tool** to create `.dialogues/<topic>.md` (the Write tool will create the `.dialogues/` directory automatically):

```markdown
# Dialogue: <topic>

Started: <YYYY-MM-DD HH:MM>
Template: <template>          # Only if using a template
Max rounds: <max-rounds>

Participants:
  - <role-a> @ <current working directory>
  - <role-b> (subagent)            # when --partner claude (default)
  - <role-b> (codex)               # when --partner codex

---

```

Use the appropriate participant line based on `--partner`. Only include one `<role-b>` line (not both).

Note: Both participants are listed upfront. Do NOT write any dialogue turns — the subagents will do that.

### Step 5: Inform user and proceed

When `--partner claude` (default):
Tell the user: "Starting dialogue. Spawning <role-a> and <role-b>..."

When `--partner codex`:
Tell the user: "Starting dialogue. Spawning <role-a> (Claude) and <role-b> (Codex)..."

Now proceed to Phase 2: Dialogue Loop.

---

## Phase 2: Dialogue Loop

You coordinate two subagents. You do NOT write dialogue turns yourself — all writes happen in subagents where output is hidden from the user.

### Quiet Operation

During the dialogue loop, minimize terminal output. Only output to the user when:
- The dialogue concludes (Phase 3)
- The dialogue is STUCK (requires user input)
- Max rounds reached (requires user input)

Brief progress indicators are acceptable: "Round 2..." but no verbose status updates.

### Spawn Subagent A (Role A)

Spawn Subagent A using the **Task tool** with these parameters:
- `subagent_type`: `"general-purpose"`
- `description`: `"Dialogue: <role-a>"`
- `prompt`:
  ```
  You are the <role-a> in a dialogue.

  ## Context
  <expanded context from Phase 1 Step 1>

  Instructions: <path-to-plugin>/dialogue-partner-instructions.md
  Dialogue file: .dialogues/<topic>.md

  Read the instructions file first, then read the dialogue file. Write your opening turn (Round 1) proposing/presenting based on the context above.
  ```

Remember this agent ID for resumption in the loop.

### Spawn Subagent B (Role B)

After Subagent A returns, spawn Subagent B. The method depends on the `--partner` setting.

#### When `--partner claude` (default)

Spawn Subagent B using the **Task tool** with:
- `subagent_type`: `"general-purpose"`
- `description`: `"Dialogue: <role-b>"`
- `prompt`:
  ```
  You are the <role-b> in a dialogue.

  Instructions: <path-to-plugin>/dialogue-partner-instructions.md
  Dialogue file: .dialogues/<topic>.md

  Read the instructions file first, then read the dialogue file to understand the conversation. Write your response following the protocol.
  ```

**Important:** Do NOT pass the user's goal to Subagent B. Let them read and interpret the file themselves for an unbiased perspective.

Remember this agent ID for resumption in the loop.

#### When `--partner codex`

Instead of the Task tool, use the **Bash tool** to run Codex CLI. Use a **timeout of 300000ms** (5 minutes).

```bash
codex exec \
  --full-auto \
  -C "<project-root>" \
  --skip-git-repo-check \
  --ephemeral \
  "<prompt>"
```

Where `<project-root>` is the current working directory and `<prompt>` is the following. **Important:** Replace `<absolute-path-to-plugin>` with the full absolute path to the plugin's directory (e.g., `/Users/.../plugins/critique-loop`). Codex cannot resolve plugin-relative paths.

````
You are the <role-b> in a structured dialogue.

## Instructions

1. Read the file <absolute-path-to-plugin>/dialogue-partner-instructions.md for full protocol rules.
2. Read the file .dialogues/<topic>.md for the conversation so far.
3. Determine the correct round number by finding the last "## [role] Round N" header in the dialogue file and following the round tracking rules (increment when both parties have spoken in round N).
4. Write your response turn by appending to the dialogue file.

## How to Write Your Turn

Use a shell command to append your turn to the dialogue file:

```bash
cat >> '.dialogues/<topic>.md' << '__CRITIQUE_LOOP_TURN_BOUNDARY_EOF__'

---

## [<role-b>] Round N — YYYY-MM-DD HH:MM

<Your message content here>

Status: <STATUS>
__CRITIQUE_LOOP_TURN_BOUNDARY_EOF__
```

Replace N with the correct round number (N+1 when appropriate), fill in the current date/time, write your actual message, and use the correct status.

## Critical Format Rules

- Your turn MUST start with `---` (horizontal rule) followed by a blank line
- Header format: `## [<role-b>] Round N — YYYY-MM-DD HH:MM`
- The `Status:` line MUST be the very last line of the file after your append
- Valid statuses: `AWAITING <other-role>`, `PROPOSING_DONE`, `DONE`, `STUCK`
- If the other party's last status is `PROPOSING_DONE`: either confirm with a summary + `Status: DONE`, or dispute with `Status: AWAITING <other-role>`
- NEVER edit previous turns — only append

## Important

- Do NOT use the Write tool or apply_patch — use the `cat >>` shell command shown above
- Do NOT modify any files other than the dialogue file. Do not create files, run destructive commands, or make any changes beyond appending your turn.
- Read the instructions file for role-specific guidance on how to approach the dialogue
- Be a genuine, critical but helpful collaborator — not a yes-person
````

**Important:** Do NOT pass the user's goal to Codex. Let it read and interpret the file itself for an unbiased perspective.

Codex has no session resume — each call is stateless. No agent ID to store.

### Write Validation

Before each spawn/resume:
1. Store current status: `PREV_STATUS=$(tail -1 .dialogues/<topic>.md)`
2. Store header count: `PREV_HEADERS=$(grep -c '## \[' .dialogues/<topic>.md)`

After subagent returns:
3. Get new status: `NEW_STATUS=$(tail -1 .dialogues/<topic>.md)`
4. If `PREV_STATUS == NEW_STATUS`: subagent didn't write → go to Error Handling
5. Get new header count: `NEW_HEADERS=$(grep -c '## \[' .dialogues/<topic>.md)`
6. If `NEW_HEADERS - PREV_HEADERS != 1`: subagent wrote multiple turns or no header → go to Error Handling

### Loop Logic

After each subagent returns, check status:
```bash
tail -1 .dialogues/<topic>.md
```

If the status line doesn't match a known value (`AWAITING <role>`, `PROPOSING_DONE`, `DONE`, `STUCK`):
- Treat as `STUCK`
- Report to user: "Unrecognized status: '<value>'. Please review the dialogue file at <path>."

Act based on the status:

| Status | Action |
|--------|--------|
| `AWAITING <role-a>` | Resume Subagent A |
| `AWAITING <role-b>` | Resume Subagent B |
| `PROPOSING_DONE` | Resume the OTHER subagent to confirm/dispute |
| `DONE` | Go to Phase 3 |
| `STUCK` | Go to STUCK Handling below |

### Resuming Subagents

#### Subagent A (always Claude)

Resume Subagent A using the **Task tool** with:
- `resume`: The agent ID (`AGENT_A_ID`)
- `prompt`: `"Continue the dialogue. Read the file for the latest turn and respond."`

#### Subagent B

**When `--partner claude` (default):**
Resume Subagent B using the **Task tool** with:
- `resume`: The agent ID (`AGENT_B_ID`)
- `prompt`: `"Continue the dialogue. Read the file for the latest turn and respond."`

**When `--partner codex`:**
Execute a fresh `codex exec` call using the **Bash tool** — same command and prompt as in "Spawn Subagent B / When `--partner codex`" above. Codex has no session resume; the dialogue file carries all context, so nothing is lost.

### Max Rounds Check

Before resuming a subagent, check if the current round equals `Max rounds` from the file header.

If max rounds reached:
1. Tell user: "Reached <N> rounds without conclusion. See full context at <file path>."
2. Ask: "Continue for another 5 rounds?"
3. If yes: Update your internal max rounds tracking, continue loop
4. If no: End dialogue, remind user of file location

### STUCK Handling

If you detect `Status: STUCK`:

1. Read the turn to understand the blocker
2. Tell user: "The dialogue is stuck. <blocker explanation>. See full context at <file path>."
3. Ask: "How would you like to proceed?"
4. Wait for user guidance
5. Resume the appropriate subagent with guidance:
   - `resume`: The agent ID for the role that should respond next
   - `prompt`: `"User provided guidance: <guidance>. Continue the dialogue incorporating this."`

### Error Handling

**Task tool failure:** If the Task tool returns an error when spawning or resuming:
- Report to user: "Subagent failed: <error>. How would you like to proceed?"
- Options: retry, end dialogue, or provide guidance

**Status unchanged:** If subagent returns but status is unchanged:
- Report to user: "Subagent returned but didn't write a turn. How would you like to proceed?"
- Options: retry spawn, end dialogue, or provide guidance

#### Codex-Specific Error Handling (when `--partner codex`)

**Non-zero exit code:** If the `codex exec` Bash command returns a non-zero exit code:
- Read the stderr output from the Bash result
- Report to user: "Codex failed (exit code <N>): <stderr summary>. How would you like to proceed?"
- Options: retry, switch to `--partner claude` for the rest of the dialogue, or end dialogue

**File not modified:** If Codex returns successfully but the dialogue file status is unchanged:
- Same as the "Status unchanged" handling above

**Malformed turn:** After Codex returns, check if the last line of the dialogue file is a valid `Status:` line (matching one of: `AWAITING <role>`, `PROPOSING_DONE`, `DONE`, `STUCK`). If not:
- Report to user: "Codex wrote a turn but the format is malformed (last line: '<value>'). Please review the dialogue file at <path>."
- Options: manually fix and continue, retry, or end dialogue

**Timeout:** All Codex `exec` calls use a 300000ms (5 minute) Bash timeout. If the command times out:
- Report to user: "Codex timed out after 5 minutes. How would you like to proceed?"
- Options: retry, switch to `--partner claude`, or end dialogue

---

## Phase 3: Conclusion

When the dialogue reaches `Status: DONE`:

1. **Validate protocol:** Check if the second-to-last turn ended with `Status: PROPOSING_DONE`
   - If not, warn user: "Note: Dialogue ended without confirmation phase (PROPOSING_DONE was skipped)."
   - Proceed anyway — this is a warning, not a blocking error
2. Read the final turn from the dialogue file
3. Extract the summary (the content before `Status: DONE`)
4. Report to user: "Dialogue concluded after <N> rounds. Summary: <summary>"
5. Remind user: "Full transcript at <file path>"

---

## Role Guidance

See `dialogue-partner-instructions.md` for role-specific guidance.

---

## Protocol Rules Reference

These rules apply to all dialogue turns (written by subagents, not the orchestrator):

1. **Append only** — Never edit previous turns
2. **Status is always last** — Final line of each turn must be `Status: <value>`
3. **Use Write tool** — All dialogue file writes use Write (read, append, write back)
4. **Round tracking** — Increment round when both parties have spoken
5. **Orchestrator doesn't write turns** — Only creates header and coordinates subagents
