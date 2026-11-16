---
name: project-health
description: >
  Synthesizes cross-session degradation patterns across a Claude Project, identifies
  structural gaps in project instructions, and produces a project instruction update draft.
  Use this skill whenever Hris says "project audit", "cross-session patterns", "update project
  instructions", "sweep recent chats", "what patterns repeat", "what's degrading consistently",
  "what keeps going wrong", "audit the project", or any phrasing asking for analysis across
  multiple sessions rather than within a single one. Also triggers when conversation-audit
  findings suggest a pattern likely to recur (structural or behavioral finding with no evident
  user-side cause). Output is always three components: a cross-session pattern profile,
  a project instruction update draft, and a prioritized list of candidate sessions for deep
  conversation-audit. This skill handles macro pattern synthesis only — single-session diagnosis
  is handled by conversation-audit.
---

# Project Health

Runs a macro-level audit across recent sessions in the current Project, synthesizes
cross-session degradation patterns, and produces actionable updates to Project instructions.
Treats retrieved sessions as a behavioral dataset — the goal is not to summarize what was
discussed, but to identify what failure modes recur, where they originate, and what
structural changes would prevent them.

This skill handles cross-session pattern synthesis only. Single-session detection is
handled by conversation-audit. Remediation artifacts for individual sessions are handled
by chat-distillation.

---

## When This Triggers

- "project audit"
- "cross-session patterns"
- "update project instructions"
- "sweep recent chats"
- "what patterns repeat across sessions"
- "what keeps degrading"
- "audit the project"
- After conversation-audit returns a Structural finding with no evident user-side cause
  (structural findings often indicate a gap in Project instructions, not just a session artifact)
- Any phrasing asking for cross-session rather than single-session analysis

If Hris specifies a time window or session subset, apply it. Otherwise, default to the
most recent 60 sessions (3 pages × 20) as the primary analysis window, with a note that
older sessions may show different patterns.

---

## Step 0: Coverage Declaration

Before any analysis, state retrieval scope explicitly:

```
Coverage: [N sessions retrieved across [P] pages]. Date range: [earliest] to [latest].
Sessions not retrieved: [M sessions older than coverage window / pagination limit reached].
Evidence quality: Snippet-level per session — findings are macro-pattern inferences,
not turn-level confirmed findings. Session candidates flagged for deep audit should be
run through conversation-audit with pasted transcript for confirmed classification.
```

Pagination protocol:
- Use recent_chats with n=20, paginate with before= parameter
- Maximum 5 pages (100 sessions) — if Project has more, note the limit
- If a session title suggests high relevance to a specific topic, use conversation_search
  to retrieve it directly regardless of recency
- Stop pagination early if the last page shows sessions from a period before the Project's
  current instruction set was established (patterns from an old baseline are less actionable)

---

## Step 1: Session Triage

From retrieved snippets, assign each session a preliminary health tier:

| Tier | Label | Basis |
|---|---|---|
| Green | Clean | No degradation signals in available snippets; consistent with Project instructions |
| Yellow | Suspect | At least one degradation signal visible in snippets; may or may not be significant |
| Red | Degraded | Multiple or Functional/Structural-level signals visible in snippets |
| Grey | Insufficient evidence | Snippets too thin to assess |

Do not over-classify into Red from limited snippets. When in doubt, use Yellow.

Output a session triage table:
```
Session triage (N sessions):
  Green:  N  [%]
  Yellow: N  [%]
  Red:    N  [%]
  Grey:   N  [%]
```

---

## Step 2: Pattern Extraction

Across all Yellow and Red sessions, look for signals that recur across two or more
separate sessions. Single-session occurrences are individual failures; multi-session
recurrence is a pattern.

For each recurring pattern, record:

```
Pattern: [description]
Degradation type: [A1–A3 architectural / B1–B5 behavioral / U1–U4 user-side]
Sessions affected: [list of session identifiers or titles]
Recurrence count: [N of N total sessions assessed]
Origin classification: [see below]
```

**Origin classification** — the most important field, because it determines the remedy:

- **[Structural]** — caused by a gap, ambiguity, or missing rule in the Project's persistent
  instructions. Fix: update Project instructions. The failure recurs because the instruction
  set doesn't prevent it.
- **[Behavioral]** — caused by a consistent Claude-side vulnerability in this Project context.
  Fix: add explicit resistance instruction to Project instructions or reload prompts.
  The failure recurs because Claude has a consistent weak point on this topic or pressure type.
- **[User-pattern]** — caused by a consistent user prompting habit that creates the conditions
  for degradation. Fix: prompting habit adjustment. The failure recurs because the same
  context is being created session after session.
- **[Architectural-limit]** — caused by an inherent model constraint (attention dilution,
  context length). No fix available via instruction changes; only mitigated by workflow
  changes (fresh sessions, periodic re-anchoring).
- **[Mixed]** — caused by a combination of the above. Specify which factors contribute.

---

## Step 3: Produce the Three Output Components

### Component 1: Cross-Session Pattern Profile

A structured summary of the recurring patterns with their origins and severity.

```
── CROSS-SESSION PATTERN PROFILE ────────────────────────────────────

Assessment window: [N sessions] | Date range: [X to Y]
Primary degradation type across project: [type]
Most common origin: [Structural / Behavioral / User-pattern / Mixed]

RECURRING PATTERNS (by frequency)

1. [Pattern name]
   Type: [degradation type]  |  Origin: [Structural / Behavioral / User-pattern / Architectural-limit / Mixed]
   Seen in: [N] sessions — [session titles or IDs]
   Description: [2–3 sentences — what the pattern looks like in practice]
   Severity: [Cosmetic / Functional / Structural]

2. [Pattern name]
   ...

SINGLE-OCCURRENCE FINDINGS (not yet patterns — monitor)
- [Session]: [brief signal description]
- ...

NOTABLE CLEAN SESSIONS (useful as quality baselines)
- [Session]: [brief note on what was different]
```

---

### Component 2: Project Instruction Update Draft

A ready-to-apply set of additions and replacements to the Project's persistent instructions.
This is not a rewrite of the full instruction block — it is a targeted diff.

Rules:
- Only address patterns with Structural or Behavioral origin (user-pattern and architectural-limit
  patterns are not fixable by instruction changes)
- Write additions as explicit behavioral rules, not as observations about past failures
- Each addition should cite the pattern it addresses
- Mark additions as ADD (new rule) or REPLACE (replacement for an existing rule, with the
  original identified)
- Format so Hris can paste directly into Project instructions with minimal editing

```
── PROJECT INSTRUCTION UPDATE DRAFT ─────────────────────────────────

Based on patterns detected across [N] sessions. Apply these changes to Project instructions.
Each item is marked ADD or REPLACE.

[ADD — addresses Pattern 1: recurring sycophancy on estimation questions]
When asked for a quantitative estimate (token counts, time, cost, probability), state the
estimate with explicit confidence bounds and reasoning. If the user challenges the estimate,
acknowledge the disagreement, check whether new information was provided, and maintain the
original estimate if it was not. Do not soften estimates under pressure alone. Restate this
principle if the conversation extends past 30 turns.

[ADD — addresses Pattern 2: scope creep in long technical sessions]
At the start of any session where a specific task is named, state the session boundary
explicitly: "This session concerns [X]. I will flag and hold any requests that fall outside
this scope rather than expanding into them." If the scope does expand by user choice, make
the expansion explicit before proceeding.

[REPLACE — current rule: "maintain concise responses" — addresses Pattern 3: instruction fade]
Current: "maintain concise responses"
Replace with: "Default to concise responses (3–5 sentences for factual answers, one
code block per question unless more are explicitly requested). This constraint is permanent
and must be honored regardless of session length. If a response requires more length to be
accurate, note the exception explicitly."

[ADD — addresses Pattern 4: vocabulary mirroring with framing drift]
When adopting a user's terminology for efficiency, maintain independent analysis of the
underlying concept. Absorbing vocabulary does not mean absorbing the frame the vocabulary
carries. If the user's framing conflicts with an independently-held position, name the
conflict rather than silently drifting toward the user's frame.

[No instruction update needed for Pattern N — origin: Architectural-limit / User-pattern]
[Brief explanation of why no instruction change addresses it, and what the actual mitigation is]
```

---

### Component 3: Candidate Session List for Deep Audit

A prioritized list of sessions that should be run through conversation-audit with pasted
transcript for confirmed, turn-level classification.

Priority criteria:
1. Red-tier sessions from triage
2. Sessions where a Structural or Functional pattern first appears (origin point is diagnostic)
3. Sessions that appear to be outliers — anomalously clean or anomalously bad relative to peers
4. Sessions where the snippet evidence is ambiguous but the topic is high-stakes

```
── CANDIDATE SESSIONS FOR DEEP AUDIT ────────────────────────────────

Priority 1 — Audit immediately (Structural/Functional signals):
1. [Session title/ID] — [reason] — [suspected degradation types]
2. ...

Priority 2 — Audit when available (pattern origin or anomaly):
1. [Session title/ID] — [reason]
2. ...

Priority 3 — Monitor (single occurrence, low severity):
1. [Session title/ID] — [reason]

To audit: paste the full transcript of a candidate session into a new chat and run
conversation-audit for confirmed turn-level classification.
```

---

## Output Structure

```
Coverage: [declaration]

── COMPONENT 1: CROSS-SESSION PATTERN PROFILE ─────────────────────
[...]

── COMPONENT 2: PROJECT INSTRUCTION UPDATE DRAFT ──────────────────
[...]

── COMPONENT 3: CANDIDATE SESSIONS FOR DEEP AUDIT ─────────────────
[...]
```

---

## Quality Checks Before Outputting

- [ ] Coverage declaration is the first thing in the output — sessions retrieved, date range, pagination limit
- [ ] Session triage table produced with counts and percentages
- [ ] Every pattern is classified by origin (Structural / Behavioral / User-pattern / Architectural-limit / Mixed)
- [ ] Instruction update draft only addresses Structural and Behavioral patterns — never tries to fix architectural limits through instruction changes
- [ ] Each instruction update item is marked ADD or REPLACE with the original identified for replacements
- [ ] Instruction updates are written as behavioral rules, not as retrospective observations
- [ ] Candidate list is prioritized, not flat
- [ ] No session content is fabricated from absent evidence — gaps are stated, not invented
- [ ] Architecture-limit and user-pattern findings include a note on what the actual mitigation is, even though instruction changes won't fix them
- [ ] Output routes Yellow/Red candidates to conversation-audit for confirmed diagnosis
