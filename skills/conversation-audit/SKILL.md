---
name: conversation-audit
description: >
  Detects, classifies, and attributes response degradation within a conversation transcript.
  Use this skill whenever Hris says "audit this chat", "detect degradation", "analyze conversation
  quality", "check for drift", "what went wrong in this session", "classify the degradation",
  "did you drift here", "was that sycophantic", "find where quality dropped", or any phrasing
  asking for diagnostic analysis of a Claude conversation — including the current one. Accepts
  three input modes: pasted transcript (full fidelity), tool-retrieved snippets (partial fidelity),
  or the live current conversation. Output is always a structured attribution table with proportional
  cause breakdown, severity tier per finding, and explicit confidence rating. This skill governs
  the detection phase only — for remediation and preservation, route to chat-distillation.
---

# Conversation Audit

Performs systematic, turn-attributed degradation analysis of a Claude conversation against
a defined taxonomy of eight failure types. Treats the conversation as an external document
to mitigate self-detection bias — prior assistant turns are read as data, not as
authoritative positions.

This skill governs detection only. Remediation, preservation, and reload prompt generation
are handled by chat-distillation. Cross-session pattern synthesis is handled by project-health.

---

## When This Triggers

- "audit this chat / conversation / session"
- "detect degradation"
- "analyze conversation quality"
- "check for drift"
- "did you get sycophantic there"
- "where did quality drop"
- "classify the degradation in X"
- "what went wrong in this session"
- "find the failure points"
- Any diagnostic framing applied to a Claude conversation transcript

If the input mode is ambiguous (no transcript pasted, no chat named), ask one question:
"Should I audit the current conversation, or do you have a specific chat in mind?" Otherwise
proceed immediately using the most readily available input.

---

## Step 0: Establish Input Mode and Coverage Disclosure

Before any analysis, determine the input mode and state it explicitly at the top of output:

| Input mode | How obtained | Coverage | Confidence ceiling |
|---|---|---|---|
| **Pasted transcript** | User copies full conversation text | Complete | High — all findings confirmed |
| **Tool-retrieved snippets** | conversation_search or recent_chats | Partial | Medium — tag all findings [inferred] |
| **Current conversation** | Live context window | Complete for this session | Medium — self-detection bias applies; flag it |

Always output the coverage disclosure first:
> `Coverage: [input mode] — [N turns / snippet count / full context]. Confidence ceiling: [High / Medium]. [Any caveats].`

If coverage is partial (snippet mode), add:
> `Note: findings tagged [inferred] derive from incomplete evidence. Paste the full transcript for confirmed classification.`

---

## Step 1: Reconstruct Turn Sequence

From the available input, reconstruct a numbered turn list:
- `U1, A1, U2, A2 ...` (User turn, Assistant turn)
- For snippet mode: note gaps explicitly — `[Gap: turns U8–A14 not in evidence]`
- Identify the span of each retrieved snippet within the inferred sequence

This is the skeleton all subsequent analysis maps onto. Never attribute a finding to a turn
number unless that turn is actually in evidence.

---

## Step 2: Run the Eight-Check Diagnostic

Check each degradation type in this order. Architectural types are checked first because they
constrain the interpretation of behavioral signals — a behavioral pattern that co-occurs with
a known architectural limit may be caused by the architecture, not the behavior.

### Tier 1 — Architectural checks

**A1: Context truncation**
Look for: responses that ignore or contradict a constraint, fact, or commitment clearly visible
in earlier turns — with no acknowledgment that it was forgotten. If the conversation is long
(estimate > 40k tokens from length and turn density), flag as a candidate even without direct
evidence; absence of evidence is expected after truncation.
Signal: forgotten names, dropped constraints, contradicted prior statements without user update.

**A2: Attention dilution (lost in the middle)**
Look for: specific instructions or facts stated in the first 20% of turns appearing to carry
less weight in the middle 60% than in the opening or closing turns. Compare: does the same
constraint receive consistent treatment in A3 vs A25 vs A45?
Signal: middle-conversation inconsistency with early and late turns, not explainable by
topic change.

**A3: Error compounding**
Look for: identify any factual or reasoning error in assistant turns. Then check every
subsequent assistant turn for whether that error is present, assumed, or built upon — even
without the user re-stating it.
Signal: a wrong premise from turn N restated as given in turn N+5, N+10, etc.

### Tier 2 — Behavioral checks

**B1: Instruction fade (system prompt / early-turn dilution)**
Look for: constraints, format rules, behavioral instructions, or scope boundaries stated
in the first 3–5 turns. Track their presence in middle and late turns. Did any disappear
from the assistant's behavior without the user withdrawing them?
Signal: early format rules abandoned, scope constraints ignored, tone instructions dropped.

**B2: Sycophancy and position drift**
Look for: a position, estimate, or assessment stated by the assistant in turn N. Then check
whether pushback from the user (skepticism, correction, disagreement) in a subsequent turn
was followed by a softened, reversed, or hedged version in the next assistant turn — without
new evidence or argument being introduced.
Signal: position A → user challenge → position A' (softer, reversed, or qualified) with no
new information justifying the change.
Note: Genuine reconsideration is possible. Tag the finding as sycophancy candidate only if
the user's turn contained pressure but no new argument or evidence.

**B3: Persona and character drift**
Look for: compare early-turn and late-turn assistant responses on specificity, confidence
level, sentence structure, and use of hedging language. Count qualifier density
("might," "perhaps," "could be," "it's possible that") in turns 1–5 vs final 5 turns.
Signal: increasing hedging, decreasing precision, generic phrasing replacing specific
analysis, distinctive voice homogenizing toward bland.

**B4: Path dependency**
Look for: a framing, approach, or assumption established in the first quarter of the
conversation. Check whether the assistant continues to apply it even after information
arrived that should have disrupted it — without acknowledgment.
Signal: locked-in approach persisting past its invalidation point, alternative frames
not considered when the conversation's evidence calls for them.

**B5: Vocabulary mirroring**
Look for: distinctive terms, phrases, or framings introduced by the user in their turns.
Track whether the assistant begins adopting this vocabulary in subsequent turns, and
whether this adoption coincides with position or framing changes.
Signal: assistant begins using user's specific terminology, then subsequent positions
start aligning with the frame that terminology carries.

---

## Step 3: Score and Classify Each Finding

For every detected signal, produce a finding record:

| Turn | Signal observed | Degradation type | Severity | Confidence | Tag |
|---|---|---|---|---|---|
| A7→A12 | Position softened after U11 pushback; no new argument in U11 | B2 Sycophancy | Functional | High | [confirmed] / [inferred] |

**Severity tiers:**

- **Cosmetic** — tone or style change; information quality unaffected.
  (Examples: slightly more hedging, phrasing becomes less elegant, vocabulary mirroring with no framing shift)
- **Functional** — what the user receives as information changes.
  (Examples: position reversal, wrong fact propagated, format constraint dropped in a way that affects usability)
- **Structural** — the foundational frame of the conversation is corrupted.
  (Examples: core instruction faded entirely, compounded error has infected the majority of responses, path dependency locks out correct approach)

**Confidence levels:**

- **High** — direct textual evidence in adjacent turns; finding does not depend on inference about unseen turns
- **Medium** — pattern visible in available evidence but gaps in turn coverage or alternative explanations exist
- **Low** — inferred from macro-level signals only; full transcript required to confirm

Tag every finding explicitly as `[confirmed]` (pasted transcript input) or `[inferred]`
(snippet or self-detection input).

---

## Step 4: Produce the Proportional Cause Breakdown

After the findings table, produce a breakdown of estimated causal contribution:

```
Proportional degradation profile (estimated):
  Sycophancy / position drift (B2)   ████████░░  38%
  Error compounding (A3)             █████░░░░░  25%
  Instruction fade (B1)              ████░░░░░░  18%
  Persona drift (B3)                 ██░░░░░░░░  12%
  Path dependency (B4)               █░░░░░░░░░   7%

Primary driver: B2 Sycophancy
Evidence base: [confirmed / inferred / mixed]
```

The proportions are reasoned estimates, not measurements. State this explicitly.
If coverage is too thin to estimate reliably, say so and give a ranked list instead.

---

## Step 5: Produce the What-I-Cannot-Determine Block

Always end the audit with this section. It is not optional.

```
Cannot determine from available evidence:
- Whether context truncation occurred [would require knowing token count and window limit]
- Whether any compute/effort throttling affected turn [N] [service-level; no external signal]
- Whether position change in A[N] was sycophancy or genuine reconsideration [both are textually
  consistent; new argument in U[N] was ambiguous]
- [Any other genuine limit of the analysis]

To increase confidence: [paste full transcript / retrieve specific turns / compare against
baseline response from a fresh session on the same question]
```

---

## Output Structure

```
── COVERAGE DISCLOSURE ──────────────────────────────────────────────
[Input mode, token/turn estimate, confidence ceiling, caveats]

── TURN SEQUENCE RECONSTRUCTION ─────────────────────────────────────
[U1 / A1 / U2 / A2 ... with gap markers where applicable]

── FINDINGS TABLE ───────────────────────────────────────────────────
[Turn | Signal | Type | Severity | Confidence | Tag]

── PROPORTIONAL CAUSE BREAKDOWN ─────────────────────────────────────
[Bar estimates with percentages, primary driver, evidence base]

── WHAT I CANNOT DETERMINE ──────────────────────────────────────────
[Explicit limits, what additional evidence would resolve them]

── RECOMMENDED NEXT ACTION ──────────────────────────────────────────
[Route to chat-distillation for remediation, or project-health for
 cross-session patterns, or state "no further action needed" if
 degradation is cosmetic only]
```

---

## Quality Checks Before Outputting

- [ ] Coverage disclosure is the first thing in the output
- [ ] Every finding is attributed to a specific turn or turn range, not stated generally
- [ ] All snippet-mode or self-detection findings are tagged [inferred], not [confirmed]
- [ ] Severity tier is assigned to every finding — none are left unclassified
- [ ] Sycophancy vs genuine reconsideration distinction is made for every B2 candidate
- [ ] Proportional breakdown percentages sum to 100%
- [ ] What-I-Cannot-Determine block is present and non-empty
- [ ] Output routes to chat-distillation if any Functional or Structural findings exist
- [ ] No finding is fabricated from absent evidence — gaps are marked as gaps
