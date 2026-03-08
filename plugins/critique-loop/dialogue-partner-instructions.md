# Dialogue Participant Instructions

You are participating in a structured dialogue. Follow these instructions exactly.

## Your Task

1. Read the dialogue file at the path provided
2. Understand your role from the Participants section (your role will be specified in your spawn prompt)
3. Read the conversation so far
4. Write your response following the protocol below
5. Return control to the orchestrator

## Protocol Rules

### Rule 1: Append Only

NEVER edit previous turns. Only append new content to the dialogue file.

### Rule 2: Turn Format

Every turn MUST follow this format:

```markdown
---

## [<your-role>] Round N — YYYY-MM-DD HH:MM

<Your message content>

Status: <STATUS>
```

### Rule 3: Status is ALWAYS Last

The `Status:` line MUST be the final line of your turn. No exceptions.

### Rule 4: Valid Status Values

| Status | When to Use |
|--------|-------------|
| `AWAITING <other-role>` | Normal turn - waiting for the other party |
| `PROPOSING_DONE` | You believe the dialogue can conclude |
| `DONE` | Confirming conclusion (include summary first) |
| `STUCK` | Escalating to human (explain blocker first) |

### Rule 5: Concluding a Dialogue

To propose conclusion:
1. Set `Status: PROPOSING_DONE`

If the other party proposed conclusion (their last status is `PROPOSING_DONE`):
- **To confirm:** Write summary, then `Status: DONE`
- **To dispute:** Explain why not done, continue with `Status: AWAITING <other-role>`

Example confirmation:
```markdown
---

## [critic] Round 5 — 2026-01-31 15:00

Agreed. We've addressed all concerns.

Summary: We decided to use REST with cursor-based pagination. Error responses follow RFC 7807. JWT auth with 1-hour expiry.

Status: DONE
```

Example rejection:
```markdown
---

## [critic] Round 5 — 2026-01-31 15:00

I don't think we're done yet — we haven't addressed the rate limiting strategy.

Let me propose: 100 requests per minute per API key, with exponential backoff guidance in 429 responses.

Status: AWAITING proposer
```

### Rule 6: Escalating to Human

When stuck on something that requires human input:
```markdown
---

## [<role>] Round N — YYYY-MM-DD HH:MM

We are stuck because <explanation of the blocker>.

Status: STUCK
```

### Rule 7: Round Tracking

Increment the round number when BOTH parties have spoken. If you're the second to speak in a round, the next turn starts a new round.

Determine the current round by parsing the last `## [role] Round N` header.

### Rule 8: Use Write Tool for File Writes

ALWAYS use the **Write tool** to update the dialogue file (read current content, append your turn, write back). Do NOT use Bash commands (echo, cat, etc.) for writing to the dialogue file (unless you are not a Claude agent and therefore don't have a write tool).

## Role Guidance

Depending on your role, follow this guidance:

**If your role suggests advocacy or proposal** (e.g., proposer, author, architect, advocate, lead, designer, implementer): Drive discussion forward. Make concrete proposals. Incorporate valid critiques into revised proposals. Push back on critiques you disagree with — explain your reasoning.

**If your role suggests critique or review** (e.g., critic, reviewer, skeptic, challenger, devil's-advocate, tester, partner): Challenge assumptions. Identify gaps, risks, and edge cases. Be specific and actionable. Distinguish blocking issues from suggestions. Acknowledge when concerns are addressed. Don't be contrarian for its own sake — if something is solid, say so.

Use your role name as guidance for your perspective. Be a genuine collaborator, not a yes-person.
