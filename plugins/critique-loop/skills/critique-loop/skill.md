t---
name: critique-loop
description: "Start a dialogue with a spawned partner. Enables planning, reviewing, and pair programming with automated turn-taking."
---

# Critique Loop

You orchestrate a structured dialogue with a spawned partner. Follow this protocol exactly.

## Parse Arguments

Extract from the invocation:
- `--topic <topic>` (REQUIRED): The dialogue identifier
- `--template <template>`: Predefined role pair (`planning`, `review`, or `pair`)
- `--role1 <role>` + `--role2 <role>`: Custom role names (both required if not using template)
- `--max-rounds <N>`: Maximum rounds (default: 5, or template default)

**Validation:**
- `--topic` is required. If missing, ask the user for it.
- Must provide EITHER `--template` OR BOTH `--role1` and `--role2`
- If validation fails, explain the requirement and ask for correct arguments

## Templates

If `--template` was provided, use these role pairs:

| Template | Your Role | Partner Role | Max Rounds |
|----------|-----------|--------------|------------|
| `planning` | proposer | critic | 5 |
| `review` | author | reviewer | 5 |
| `pair` | lead | partner | 7 |

If using custom roles: You take `--role1`, partner takes `--role2`.

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

Your role = first role (from template or `--role1`)
Partner role = second role (from template or `--role2`)

### Step 4: Create the dialogue file

Use the **Write tool** to create `.dialogues/<topic>.md` (the Write tool will create the `.dialogues/` directory automatically):

```markdown
# Dialogue: <topic>

Started: <YYYY-MM-DD HH:MM>
Template: <template>          # Only if using a template
Max rounds: <max-rounds>

Participants:
  - <your-role> @ <your current working directory>
  - <partner-role> (partner)

---

```

Note: Both participants are listed upfront. Do NOT write any dialogue turns — the subagents will do that.

### Step 5: Inform user and proceed

Tell the user: "Starting dialogue. Spawning <your-role> and <partner-role>..."

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

### Spawn Subagent A (your role)

Spawn the first subagent (your role) using the **Task tool** with these parameters:
- `subagent_type`: `"general-purpose"`
- `description`: `"Dialogue: <your-role>"`
- `prompt`:
  ```
  You are the <your-role> in a dialogue.

  ## Context
  <expanded context from Phase 1 Step 1>

  Instructions: <path-to-plugin>/dialogue-partner-instructions.md
  Dialogue file: .dialogues/<topic>.md

  Read the instructions file first, then read the dialogue file. Write your opening turn (Round 1) proposing/presenting based on the context above.
  ```

Remember this agent ID for resumption in the loop.

### Spawn Subagent B (partner role)

After Subagent A returns, spawn the second subagent (partner role) using the **Task tool** with:
- `subagent_type`: `"general-purpose"`
- `description`: `"Dialogue: <partner-role>"`
- `prompt`:
  ```
  You are the <partner-role> in a dialogue.

  Instructions: <path-to-plugin>/dialogue-partner-instructions.md
  Dialogue file: .dialogues/<topic>.md

  Read the instructions file first, then read the dialogue file to understand the conversation. Write your response following the protocol.
  ```

**Important:** Do NOT pass the user's goal to Subagent B. Let them read and interpret the file themselves for an unbiased perspective.

Remember this agent ID for resumption in the loop.

### Write Validation

Before each spawn/resume:
1. Store current status: `PREV_STATUS=$(tail -1 .dialogues/<topic>.md)`

After subagent returns:
2. Get new status: `NEW_STATUS=$(tail -1 .dialogues/<topic>.md)`
3. If `PREV_STATUS == NEW_STATUS`: subagent didn't write → go to Error Handling

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
| `AWAITING <your-role>` | Resume Subagent A |
| `AWAITING <partner-role>` | Resume Subagent B |
| `PROPOSING_DONE` | Resume the OTHER subagent to confirm/dispute |
| `DONE` | Go to Phase 3 |
| `STUCK` | Go to STUCK Handling below |

### Resuming Subagents

Resume a subagent using the **Task tool** with:
- `resume`: The agent ID (`AGENT_A_ID` or `AGENT_B_ID`)
- `prompt`: `"Continue the dialogue. Read the file for the latest turn and respond."`

### Max Rounds Check

Before resuming a subagent, check if the current round equals `Max rounds` from the file header.

If max rounds reached:
1. Tell user: "Reached <N> rounds without conclusion. See full context at <file path>."
2. Ask: "Continue for another 10 rounds?"
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

---

## Phase 3: Conclusion

When the dialogue reaches `Status: DONE`:

1. Read the final turn from the dialogue file
2. Extract the summary (the content before `Status: DONE`)
3. Report to user: "Dialogue concluded after <N> rounds. Summary: <summary>"
4. Remind user: "Full transcript at <file path>"

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
