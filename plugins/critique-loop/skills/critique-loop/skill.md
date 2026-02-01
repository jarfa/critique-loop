---
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
- `--max-rounds <N>`: Maximum rounds (default: 20, or template default)

**Validation:**
- `--topic` is required. If missing, ask the user for it.
- Must provide EITHER `--template` OR BOTH `--role1` and `--role2`
- If validation fails, explain the requirement and ask for correct arguments

## Templates

If `--template` was provided, use these role pairs:

| Template | Your Role | Partner Role | Max Rounds |
|----------|-----------|--------------|------------|
| `planning` | proposer | critic | 20 |
| `review` | author | reviewer | 20 |
| `pair` | lead | partner | 30 |

If using custom roles: You take `--role1`, partner takes `--role2`.

## Configuration

```
DIALOGUE_DIR="${DIALOGUE_DIR:-$HOME/.claude/dialogues}"
DIALOGUE_FILE="${DIALOGUE_DIR}/<topic>.md"
INSTRUCTIONS_FILE="<path-to-this-plugin>/dialogue-partner-instructions.md"
```

---

## Phase 1: Setup

### Step 1: Gather context

Ask the user: "What would you like to discuss or accomplish in this dialogue?"

Wait for their response. This becomes the focus of your opening turn.

### Step 2: Offer to compact

Tell the user: "This dialogue may consume significant context. Would you like to /compact before we begin?"

Wait for their response. If they want to compact, wait for them to do so before continuing.

### Step 3: Create directories

Use Bash to create the dialogues directory:

```bash
mkdir -p "${HOME}/.claude/dialogues"
```

This is the ONLY bash command needed during setup.

### Step 4: Determine roles

- If `--template` provided: Use roles from template table
- If `--role1`/`--role2` provided: Use those role names

Your role = first role (from template or `--role1`)
Partner role = second role (from template or `--role2`)

### Step 5: Create the dialogue file

Use the **Write tool** to create `${HOME}/.claude/dialogues/<topic>.md`:

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

Note: Both participants are listed upfront. The partner is marked with "(partner)".

### Step 6: Write your opening turn

Use the **Edit tool** to append your first message to the dialogue file:

```markdown
## [<your-role>] Round 1 — <YYYY-MM-DD HH:MM>

<Your opening message based on the user's context>

Status: AWAITING <partner-role>
```

### Step 7: Inform user and proceed

Tell the user: "Starting dialogue as <your-role>. Spawning <partner-role> partner..."

Now proceed to Phase 2: Dialogue Loop.

---

## Phase 2: Dialogue Loop

This is the main orchestration loop. You spawn the partner, they respond, you respond, repeat.

### Initial Spawn

Spawn the partner using the **Task tool** with these parameters:
- `subagent_type`: `"general-purpose"`
- `description`: `"Dialogue partner: <partner-role>"`
- `prompt`: A neutral prompt (see below)

**Neutral prompt template:**
```
You are participating in a dialogue as the <partner-role> role.

Instructions file: <path-to-plugin>/dialogue-partner-instructions.md
Dialogue file: ${HOME}/.claude/dialogues/<topic>.md

Read the instructions file first, then read the dialogue file to understand the conversation. Write your response following the protocol in the instructions.
```

**Important:** Do NOT summarize the dialogue content or frame the discussion. Let the partner read and interpret the file themselves.

Save the agent ID returned by the Task tool — you will use it to resume the partner.

### Loop Logic

After the partner returns, read the dialogue file and check the latest `Status:` line:

| Status | Action |
|--------|--------|
| `AWAITING <your-role>` | Write your turn, then resume partner |
| `AWAITING <partner-role>` | Something went wrong — partner should have written AWAITING you. Read file and debug. |
| `PROPOSING_DONE` | Other party wants to conclude. Evaluate and either confirm (write summary + DONE) or dispute (explain + AWAITING). Then resume partner if not DONE. |
| `DONE` | Dialogue complete. Go to Phase 3. |
| `STUCK` | Partner is stuck. Go to STUCK Handling below. |

### Writing Your Turn

When it's your turn, use the **Edit tool** to append:

```markdown
---

## [<your-role>] Round N — <YYYY-MM-DD HH:MM>

<Your message>

Status: <STATUS>
```

**Round tracking:** Increment round when both parties have spoken.

### Resuming the Partner

After writing your turn, resume the partner using the **Task tool** with:
- `resume`: The agent ID from initial spawn
- `prompt`: `"Continue the dialogue. Read the file for the latest turn and respond."`

### Max Rounds Check

Before resuming the partner, check if the current round equals `Max rounds` from the file header.

If max rounds reached:
1. Tell user: "Reached <N> rounds without conclusion. See full context at <file path>."
2. Ask: "Continue for another 10 rounds?"
3. If yes: Update your internal max rounds tracking, continue loop
4. If no: End dialogue, remind user of file location

### STUCK Handling

If you detect `Status: STUCK` (from either party):

1. Read the turn to understand the blocker
2. Tell user: "The dialogue is stuck. <blocker explanation>. See full context at <file path>."
3. Ask: "How would you like to proceed?"
4. Wait for user guidance
5. Write your next turn incorporating the user's guidance (you are acting as the user's representative)
6. Resume partner with the agent ID

### PROPOSING_DONE Handling

If partner wrote `PROPOSING_DONE`:
- Evaluate whether the dialogue is truly complete
- If complete: Write summary + `Status: DONE`, proceed to Phase 3
- If not complete: Explain what's missing + `Status: AWAITING <partner-role>`, resume partner

If YOU want to propose completion:
- Write `Status: PROPOSING_DONE`
- Resume partner
- Partner will either confirm (DONE) or dispute (AWAITING you)

---

## Phase 3: Conclusion

When the dialogue reaches `Status: DONE`:

1. Read the final turn from the dialogue file
2. Extract the summary (the content before `Status: DONE`)
3. Report to user: "Dialogue concluded after <N> rounds. Summary: <summary>"
4. Remind user: "Full transcript at <file path>"

---

## Role Guidance

Follow this guidance based on your role:

**proposer (planning):** Drive discussion forward. Make concrete proposals. Incorporate valid critiques into revised proposals. Push back on critiques that are based on misunderstanding or that you disagree with — explain your reasoning.

**critic (planning):** Challenge assumptions. Identify gaps, risks, and edge cases. Acknowledge when concerns are addressed. Don't be contrarian for its own sake.

**author (review):** Present work clearly. Respond to feedback — accept, push back with reasoning, or ask for clarification.

**reviewer (review):** Be specific and actionable. Distinguish blocking issues from suggestions. Approve when standards are met.

**lead (pair):** Set direction, make decisions when stuck, keep progress moving.

**partner (pair):** Raise concerns, suggest alternatives, sanity-check decisions.

For custom roles: Use the role name as guidance for your perspective. Be a genuine collaborator.

---

## Protocol Rules Reference

These rules apply to all turns you write:

1. **Append only** — Never edit previous turns
2. **Status is always last** — Final line of each turn must be `Status: <value>`
3. **Use Edit tool** — All dialogue file writes use Edit, not Bash commands
4. **Round tracking** — Increment round when both parties have spoken
