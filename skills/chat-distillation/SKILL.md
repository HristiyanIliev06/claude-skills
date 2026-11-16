---
name: chat-distillation
description: >
  Preserves the intellectual content of a conversation, remediates detected degradation, and
  produces paste-ready artifacts for continuation in a fresh session. Use this skill whenever
  Hris says "distill this conversation", "preserve this chat", "prepare a reload prompt",
  "extract the decisions", "make a conversation skeleton", "save what matters before closing",
  "summarize for a new chat", or any phrasing asking to carry a conversation's value forward
  into a new session — especially after degradation has been detected. Also triggers after any
  conversation-audit run that found Functional or Structural severity findings. Output is always
  three artifacts: a distillation document, a reload prompt, and a conversation skeleton.
  Scope boundary: this skill handles full-conversation preservation with continuation intent.
  For a single isolated technical error, use error-isolation-prompt-generator instead.
---

# Chat Distillation

Extracts, preserves, and restructures the intellectual value of a conversation for clean
continuation in a fresh session. Operates in two modes: standard distillation (no prior
audit) and profile-aware distillation (conversation-audit output is available and feeds
directly into the reload prompt's re-anchoring block).

This skill handles remediation and preservation. Detection is handled by conversation-audit.
Cross-session pattern synthesis is handled by project-health.

---

## When This Triggers

- "distill this conversation"
- "preserve this chat"
- "prepare a reload prompt"
- "extract the decisions"
- "conversation skeleton"
- "save what matters before closing"
- "summarize for a new chat"
- "make a handoff prompt"
- After any conversation-audit finding of Functional or Structural severity
- Any phrasing asking to carry a conversation forward into a new session cleanly

If the conversation to distill is not the current one, ask one question: "Should I distill
the current conversation, or a specific other chat?" Then proceed. Do not ask for more
than one clarification.

---

## Step 0: Check for Degradation Profile

Before any extraction, check whether conversation-audit output is present in the current
session context.

- **Profile present**: activate profile-aware mode. The reload prompt (Artifact 2) will
  contain an explicit re-anchoring block derived from the detected failure modes. Note
  which degradation types were flagged and at what severity.
- **Profile absent**: activate standard mode. The reload prompt will include general
  re-anchoring best practices but no failure-specific targeting.

State the mode at the top of output:
> `Mode: [Standard / Profile-aware]. [If profile-aware: primary degradation drivers from audit: X, Y, Z]`

---

## Step 1: Turn Quality Classification

Read all available turns and assign each assistant turn a quality tier:

| Tier | Label | Criteria |
|---|---|---|
| 1 | High-quality | Precise, independent, consistent with constraints, no detected degradation signal |
| 2 | Neutral | Adequate but generic; no active degradation signal; low information density |
| 3 | Degraded | Sycophantic reversal, compounded error, faded instruction, mirrored framing, or confirmed drift |

User turns are not classified — they are always preserved as-is (they are the ground truth
of what was asked). Only assistant turns receive quality tiers.

Mark each turn: `A[N] [H]` / `A[N] [N]` / `A[N] [D]`

If operating from snippets rather than full transcript, note: turns not in evidence are
treated as neutral and included in the distillation document but excluded from the skeleton.

---

## Step 2: Extract the Five Components

From the full turn set (all tiers), extract:

**1. Purpose**
The original stated or inferred goal of the conversation. One sentence. This is the anchor
that every subsequent component must connect to. If the conversation's purpose drifted
(scope creep), state the original purpose and note the drift separately.

**2. Decisions and reasoning chains**
Every substantive decision made during the conversation, with its reasoning. Format:
- Decision: [what was decided]
- Reasoning: [why, in the assistant's own words from the highest-quality turn where
  this was discussed — not from a degraded turn]
- Status: [confirmed / revised / open]

If a decision was made in a degraded turn, flag it: `[Decision made in degraded turn A[N] —
verify before carrying forward]`

**3. Active constraints**
Every constraint, format rule, behavioral instruction, or scope boundary that should govern
the continuation session. Source from: system prompt (if visible), early user turns, and
any explicitly confirmed rules. Do not include constraints that were withdrawn.
List as explicit rules, not prose.

**4. Current work state**
Where the conversation was in its task at the point of distillation. Specifically:
- What has been completed
- What is in progress (and at what stage)
- What has not yet been started
This maps directly to the reload prompt's opening instruction.

**5. Open threads**
Questions raised but not answered, ideas flagged for later, tangents deliberately deferred.
These are distinct from the current work state — they are potential future directions,
not immediate next steps.

---

## Artifact 1: Distillation Document

A standalone, structured record of the conversation's intellectual content. This is the
canonical preservation artifact — readable independent of the original conversation.

```
── DISTILLATION ─────────────────────────────────────────────────────
Source: [conversation name / current session / date if known]
Coverage: [full transcript / snippet-based — gaps noted]
Quality note: [N high-quality turns / M degraded turns detected and excluded from skeleton]

PURPOSE
[One sentence]

DECISIONS
1. [Decision] — [Reasoning] — Status: [confirmed / revised / open]
   ⚠ [Flag if sourced from degraded turn]
2. ...

ACTIVE CONSTRAINTS
- [Constraint]
- [Constraint]
- ...

CURRENT WORK STATE
Completed: [list]
In progress: [specific stage]
Not started: [list]

OPEN THREADS
- [Thread]
- [Thread]
- ...
```

---

## Artifact 2: Reload Prompt

A single paste-ready block that reconstructs a productive working state in a fresh session.
Must require zero editing. Two sub-modes:

### Standard reload prompt structure

```
[Purpose and role statement — one paragraph establishing what this session is for and
 what kind of assistant behavior it requires]

**Context**
[Compressed summary of what has been established: decisions, current state, constraints.
 Written as the user briefing a fresh assistant — past tense for completed work,
 present tense for active constraints.]

**Active constraints**
[Bullet list — identical to Artifact 1's active constraints section]

**Current state**
[Where the work stands. One short paragraph.]

**Continue from**
[Exact instruction for what to do first in the new session]
```

### Profile-aware reload prompt additions

When degradation profile is present, insert a re-anchoring block immediately after
the context section:

```
**Re-anchoring — read before responding**
The conversation being continued had the following detected failure patterns. Treat
these as explicit behavioral instructions for this session:

[For each Functional or Structural finding from conversation-audit:]
- [B2 Sycophancy detected on topic X]: Hold positions under pushback. If the user
  challenges an assessment, acknowledge the disagreement explicitly, state whether
  new evidence was provided, and maintain the original position if it was not.
  Do not soften under pressure alone.
- [B1 Instruction fade — constraint Y dropped at turn N]: The following constraint
  must be treated as permanently active regardless of session length: [constraint].
- [A3 Error compounding — root error at A[N]]: The following claim from the source
  conversation was identified as an error and must not be carried forward: [claim].
  Correct position: [correction].
- [B3 Persona drift]: Maintain specificity and directness equivalent to the early
  turns of the source conversation. Resist increasing hedging as the session lengthens.
- [B4 Path dependency]: The source conversation was locked into [frame]. This session
  should treat that frame as one option among several, not as given.
[Continue for each relevant finding]
```

The re-anchoring block is written as direct behavioral instructions to Claude, not
as a description of what went wrong. It is positioned before "Continue from" so it
governs the first response.

---

## Artifact 3: Conversation Skeleton

The source conversation stripped to its highest-quality turns only. This is the paste
context when you want a fresh session to have the actual exchange — not just a summary —
but without carrying degraded turns into the new window.

Rules:
- Include all User turns (unfiltered — they are the request record)
- Include only Assistant turns classified High-quality [H]
- Replace each excluded Degraded [D] turn with: `[A[N] excluded — [type] degradation detected]`
- Replace each excluded Neutral [N] turn with: `[A[N] excluded — low information density]`
  (only if the turn adds nothing; neutral turns with meaningful content may be included)
- The skeleton must read coherently — if excluding a turn breaks the thread,
  add a one-sentence bridge: `[Bridge: A[N] excluded; conversation continued on [topic]]`

Format:
```
── CONVERSATION SKELETON ─────────────────────────────────────────────
[High-quality turns only. Degraded turns marked and excluded.]

U1: [verbatim]
A1: [verbatim, if High-quality] / [A1 excluded — B2 Sycophancy]
U2: [verbatim]
A2: [verbatim, if High-quality]
...
```

---

## Output Structure

Deliver all three artifacts in sequence with a one-line header per artifact.
No preamble before Artifact 1. One blank line between artifacts.

```
Mode: [Standard / Profile-aware]. [Degradation drivers if profile-aware.]

── ARTIFACT 1: DISTILLATION DOCUMENT ───────────────────────────────
[...]

── ARTIFACT 2: RELOAD PROMPT ───────────────────────────────────────
[...]

── ARTIFACT 3: CONVERSATION SKELETON ───────────────────────────────
[...]
```

---

## Quality Checks Before Outputting

- [ ] Mode declared (standard vs profile-aware) at top of output
- [ ] All five components extracted in Artifact 1 — none omitted even if empty (mark as "None identified")
- [ ] Every decision sourced from a degraded turn is flagged
- [ ] Reload prompt requires zero editing — paste-ready
- [ ] Profile-aware reload prompt contains re-anchoring block for every Functional and Structural finding
- [ ] Re-anchoring instructions are written as behavioral directives, not as descriptions of past failures
- [ ] Conversation skeleton preserves all User turns without exception
- [ ] Skeleton exclusions are marked with reason, not silently dropped
- [ ] Skeleton reads coherently with bridge notes where needed
- [ ] Scope boundary honored: if request is for a single isolated error, route to error-isolation-prompt-generator
