---
name: dialogue-partner
description: "Start or join a dialogue between two Claude Code instances. Enables planning, reviewing, and pair programming across separate workspaces."
---

# Dialogue Partner

You are participating in a structured dialogue with another Claude Code instance. Follow this protocol exactly.

## Parse Arguments

Extract from the invocation:
- `--topic <topic>` (REQUIRED): The dialogue identifier
- `--role <role>`: Your role name
- `--template <template>`: Predefined role pair (`planning`, `review`, or `pair`)
- `--max-rounds <N>`: Maximum rounds (default: 20)

**Validation:**
- `--topic` is required. If missing, ask the user for it.
- Either `--role` OR `--template` must be provided (not both, not neither)

## Configuration

```
DIALOGUE_DIR="${DIALOGUE_DIR:-$HOME/.claude/dialogues}"
DIALOGUE_FILE="${DIALOGUE_DIR}/<topic>.md"
```

## Determine Mode

Check the dialogue file to determine your mode:

| Condition | Mode |
|-----------|------|
| Dialogue file doesn't exist | **Start** — You create the dialogue |
| Dialogue file exists, your role is NOT in Participants | **Join** — You join an existing dialogue |
| Dialogue file exists, your role IS in Participants | **Resume** — You're rejoining |

**How to check:** Read the dialogue file's Participants section. If your role name appears there, you're resuming. If not (but file exists), you're joining.

## Templates

If `--template` was provided, use these role pairs:

| Template | First Role | Second Role | Max Rounds |
|----------|------------|-------------|------------|
| `planning` | proposer | critic | 20 |
| `review` | author | reviewer | 20 |
| `pair` | lead | partner | 30 |

- **If Starting:** Take the "First Role"
- **If Joining:** Take the "Second Role" (auto-assigned from file header)

### Role Guidance

**planning → proposer:** Drive discussion forward. Make concrete proposals. Incorporate valid critiques into revised proposals. Push back on critiques that are based on misunderstanding or that you disagree with — explain your reasoning.

**planning → critic:** Challenge assumptions. Identify gaps, risks, and edge cases. Acknowledge when concerns are addressed.

**review → author:** Present work clearly. Respond to feedback — accept, push back with reasoning, or ask for clarification.

**review → reviewer:** Be specific and actionable. Distinguish blocking issues from suggestions. Approve when standards are met.

**pair → lead:** Set direction, make decisions when stuck, keep progress moving.

**pair → partner:** Raise concerns, suggest alternatives, sanity-check decisions.

---

## Mode: Start (Create New Dialogue)

You are creating a new dialogue. Follow these steps:

### Step 1: Create directories

Use Bash to create the dialogues directory:

```bash
mkdir -p "${HOME}/.claude/dialogues"
```

This is the ONLY bash command needed during setup.

### Step 2: Determine your role

- If `--template` provided: Use the **first role** from the template table above
- If `--role` provided: Use that role name

### Step 3: Gather context

Ask the user: "What would you like to discuss or accomplish in this dialogue?"

Wait for their response. Include their answer as the focus of your opening turn.

### Step 4: Create the dialogue file

Use the **Write tool** to create `${HOME}/.claude/dialogues/<topic>.md`:

```markdown
# Dialogue: <topic>

Started: <YYYY-MM-DD HH:MM>
Template: <template>          # Only if using a template
Max rounds: <max-rounds>

Participants:
  - <your-role> @ <your current working directory>

---

```

### Step 5: Write your opening turn

Use the **Edit tool** to append your first message to the dialogue file:

```markdown
## [<your-role>] Round 1 — <YYYY-MM-DD HH:MM>

<Your opening message based on the user's context>

Status: AWAITING <other-role>
```

Where `<other-role>` is:
- If template: the second role from template table
- If custom role: ask the user what the other role name will be

### Step 6: Inform the user

Tell the user:
- Dialogue created at: `${HOME}/.claude/dialogues/<topic>.md`
- Waiting for counterpart to join with: `/dialogue-partner --topic <topic>`

### Step 7: Begin polling

Now enter the polling loop (see "Waiting for Response" below).

---

## Mode: Join (Join Existing Dialogue)

An existing dialogue was found. You are joining it.

### Step 1: Read the dialogue file

Use the **Read tool** to read `${HOME}/.claude/dialogues/<topic>.md` and understand:
- The topic and context
- Who the other participant is
- What template is being used (if any)
- What has been said so far

### Step 2: Determine your role

- If the file has a `Template:` line, take the **second role** from that template
- If `--role` was provided, use that role (verify it's not already taken)
- The role must NOT already be in the Participants list

### Step 3: Update the dialogue file header

Use the **Edit tool** to add yourself to the Participants section:

```markdown
Participants:
  - <existing-role> @ <their-directory>
  - <your-role> @ <your current working directory>
```

### Step 4: Write your response

Use the **Edit tool** to append your turn to the dialogue file:

```markdown
---

## [<your-role>] Round N — <YYYY-MM-DD HH:MM>

<Your response to the dialogue>

Status: AWAITING <other-role>
```

The round number should match the current round from the dialogue.

### Step 5: Begin polling

Now enter the polling loop (see "Waiting for Response" below).

---

## Mode: Resume (Rejoin Existing Dialogue)

You were previously part of this dialogue and are rejoining (perhaps after a restart).

### Step 1: Read recent dialogue

Use the **Read tool** to read the dialogue file. Focus on the last few turns to understand current state.

### Step 2: Check current status

Parse the latest `Status:` line:
- If `AWAITING <your-role>`: It's your turn. Write your response.
- If `AWAITING <other-role>`: Enter polling loop.
- If `DONE` or `STUCK`: Inform user the dialogue has ended.

Also check if max rounds was exceeded (compare last turn's round number against `Max rounds:` in header). If so, inform user the dialogue was paused at max rounds.

### Step 3: Continue or wait

If it's your turn, write your response following the Protocol Rules below, then enter the polling loop.
If not, enter the polling loop immediately.

---

## Waiting for Response (Polling Loop)

After writing your turn with `Status: AWAITING <other-role>`, poll for the counterpart's response:

1. **Wait**: Pause for 10 seconds
2. **Read**: Use the **Read tool** to read the dialogue file
3. **Check status**: Parse the latest `Status:` line:
   - If `AWAITING <your-role>`: It's your turn — respond following Protocol Rules
   - If `AWAITING <other-role>`: Go back to step 1
   - If `DONE`: Extract the summary from the final turn, display it to the user, stop polling
   - If `STUCK`: Inform user the dialogue needs human intervention, stop polling
   - If max rounds exceeded: Inform user, stop polling
4. **Keep user informed**: Display "Waiting for <other-role>... (checking every 10s)"
5. **Check timeout**: Parse the timestamp from the last turn header. If >5 minutes old and still awaiting the other role, warn: "Counterpart may have disconnected. They can rejoin with `/dialogue-partner --topic <topic>`"

Continue this loop until it's your turn or a terminal state is reached.

### On Dialogue Completion

When you detect `Status: DONE` (whether you wrote it or the counterpart did):

1. Locate the final turn in the dialogue file
2. Extract the summary (the content before `Status: DONE`)
3. Report to your user: "Dialogue concluded. Here's the summary: <summary>"

Both participants must report the same summary from the shared file.

---

## Protocol Rules

**ALWAYS follow these rules when participating in a dialogue:**

### Rule 1: Append Only

NEVER edit previous turns. Only append new content to the dialogue file.

### Rule 2: Turn Format

Every turn MUST follow this format:

```markdown
---

## [<role>] Round N — YYYY-MM-DD HH:MM

<Your message content>

Status: <STATUS>
```

### Rule 3: Status is ALWAYS Last

The `Status:` line MUST be the final line of your turn. No exceptions.

### Rule 4: Valid Status Values

| Status | When to Use |
|--------|-------------|
| `AWAITING <other-role>` | Normal turn - waiting for counterpart |
| `PROPOSING_DONE` | You believe the dialogue can conclude |
| `DONE` | Confirming conclusion (include summary first) |
| `STUCK` | Escalating to human (explain blocker first) |

### Rule 5: Concluding a Dialogue

To conclude:
1. Set `Status: PROPOSING_DONE`
2. Counterpart either:
   - **Confirms:** Writes summary, then `Status: DONE`
   - **Rejects:** Explains why not done, continues with `Status: AWAITING <role>`

Example confirmation:
```markdown
---

## [critic] Round 5 — 2026-01-25 15:00

Agreed. We've addressed all concerns.

Summary: We decided to use REST with cursor-based pagination. Error responses follow RFC 7807. JWT auth with 1-hour expiry.

Status: DONE
```

Example rejection:
```markdown
---

## [critic] Round 5 — 2026-01-25 15:00

I don't think we're done yet — we haven't addressed the rate limiting strategy.

Let me propose: 100 requests per minute per API key, with exponential backoff guidance in 429 responses.

Status: AWAITING proposer
```

### Rule 6: Escalating to Human

When stuck:
```markdown
---

## [<role>] Round N — YYYY-MM-DD HH:MM

We are stuck because <explanation of the blocker>.

Status: STUCK
```

### Rule 7: Round Tracking

Increment the round number when BOTH parties have spoken. If you're the second to speak in a round, the next turn starts a new round.

Determine the current round by counting turn pairs in the dialogue, or by parsing the last `## [role] Round N` header.

### Rule 8: Use Edit Tool for File Writes

ALWAYS use the **Edit tool** to append turns to the dialogue file. Do NOT use bash commands (echo, cat, etc.) for writing to the dialogue file.

The only Bash command in this entire skill should be `mkdir -p` at setup. Everything else uses Read, Edit, or Write tools.
