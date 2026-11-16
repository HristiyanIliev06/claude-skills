---
name: prompt-engineering
description: >
  Expert prompt engineering skill for writing, critiquing, structuring, and improving prompts.
  Use this skill whenever the user: asks to write a prompt, improve a prompt, critique a prompt,
  evaluate prompt quality, build a system prompt, create chain-of-thought instructions, design
  few-shot examples, or says anything like "help me prompt", "is this prompt good", "rewrite my
  prompt", "make this clearer for the AI", "what's wrong with my prompt", or "turn this into a
  system prompt". Always apply this skill even for casual phrasing like "can you make a prompt for X".
  Covers: general task prompts, chain-of-thought / reasoning prompts, and system prompts for AI apps/agents.
---

# Prompt Engineering Skill

This skill governs how Claude approaches all prompt writing and evaluation work.
Always read this fully before responding to any prompt-related request.

---

## Core Philosophy

A good prompt is **unambiguous, complete, and kind to the model**.
It tells the model: who it is, what to do, how to respond, and what success looks like.
Every critique and every newly written prompt should be measured against this standard.

---

## Two Modes

### Mode A — Critique & Score (user shares an existing prompt)
Triggered when the user pastes or describes a prompt they've written.

### Mode B — Build from Scratch (user wants a new prompt)
Triggered when the user describes a task and wants Claude to write the prompt for it.

Both modes **always end with a side-by-side before/after** (or original/new for Mode B).

---

## Mode A — Critique & Score

### Step 1: Analyze against the 7 dimensions

Score each dimension 1–5. Be direct — no dimension gets a 5 unless it genuinely earns it.

| # | Dimension | What it checks |
|---|-----------|---------------|
| 1 | **Clarity** | Is the instruction unambiguous? Could the model misinterpret it? |
| 2 | **Specificity** | Are constraints, scope, and requirements defined? |
| 3 | **Role / Context** | Does the prompt establish who the model is and what situation it's in? |
| 4 | **Output format** | Is the desired format, length, or structure specified? |
| 5 | **Edge case handling** | Are failure modes, exceptions, or tricky cases addressed? |
| 6 | **Chain-of-thought** (if applicable) | Does it ask the model to reason before concluding? |
| 7 | **Tone / Persona** (for system prompts) | Is the voice and behavior of the model defined? |

### Step 2: Identify the top 2–3 issues

Be specific. "Vague" is not a critique — "the instruction 'be helpful' gives the model no actionable guidance on what help means in this context" is.

### Step 3: Produce the side-by-side

```
── BEFORE ─────────────────────────────────
[original prompt, unchanged]

── AFTER ──────────────────────────────────
[improved prompt]

── WHAT CHANGED ───────────────────────────
• [change 1 + why]
• [change 2 + why]
• [change 3 + why]
```

---

## Mode B — Build from Scratch

### Step 1: Clarify intent (if needed)
If the user's description is thin, ask one targeted question before writing. Example:
> "Is this prompt going to a system prompt for an app, or are you running it directly?"
Never ask more than one question. If you can infer, infer and proceed.

### Step 2: Draft using the appropriate template

Read `/references/templates.md` for the relevant template (general task, CoT, or system prompt).
Select the one that best fits the use case — or combine elements if the task is hybrid.

### Step 3: Produce the side-by-side

For Mode B, "before" is the user's raw description (their words), "after" is the structured prompt.

```
── USER'S DESCRIPTION (before) ────────────
[paste their description verbatim]

── STRUCTURED PROMPT (after) ──────────────
[the fully written prompt]

── DESIGN DECISIONS ───────────────────────
• [why you chose this structure]
• [why you added X]
• [what to adjust if their use case differs]
```

---

## Prompt Type Guidance

### General Task Prompts
- Lead with the task verb: "Summarize / Classify / Extract / Generate / Translate..."
- State the input format and output format explicitly
- Add a constraint clause: "Do not include X", "Limit to N words"
- If multiple steps exist, number them

### Chain-of-Thought (CoT) Prompts
- Add an explicit reasoning instruction before the answer instruction:
  `"Think through this step by step before giving your final answer."`
  or: `"First identify [X], then [Y], then conclude with [Z]."`
- Never ask for the conclusion first — reasoning must precede output
- For complex reasoning: use XML tags to separate thinking from answer:
  `<thinking>...</thinking>` then `<answer>...</answer>`
- Specify what "good reasoning" looks like for this specific task

### System Prompts (for AI apps/agents)
Every strong system prompt has these sections (in order):

1. **Identity** — Who is this model? One sentence.
   > "You are a customer support assistant for [Product], a [brief description]."

2. **Purpose** — What is its job?
   > "Your job is to help users troubleshoot issues, understand features, and escalate when needed."

3. **Behavior rules** — What does it always/never do?
   > "Always ask for the user's account ID before diagnosing. Never speculate about pricing."

4. **Tone** — How does it communicate?
   > "Friendly, concise, and patient. No jargon. Use plain language."

5. **Output format** (if applicable)
   > "Respond in 3 sentences max unless the user explicitly asks for more detail."

6. **Edge cases / escalation**
   > "If you cannot resolve the issue, say: 'Let me connect you with a human agent.'"

---

## Scoring Reference

| Score | Meaning |
|-------|---------|
| 5 | Excellent — nothing to improve here |
| 4 | Good — minor polish possible |
| 3 | Adequate — works but leaves value on the table |
| 2 | Weak — will produce inconsistent results |
| 1 | Missing — this dimension is absent entirely |

**Overall score** = weighted average (clarity and specificity count double).
Show the overall as: `Overall: X/5`

---

## Tone of Feedback

- Direct but not harsh. The user is learning.
- Always explain *why* a change improves the prompt, not just *what* changed.
- If a prompt is already strong, say so clearly — don't invent criticism.
- Frame improvements as options where there's genuine ambiguity: "If your audience is non-technical, consider..."

---

## References

- `references/templates.md` — Copy-paste templates for all three prompt types. Read before writing any new prompt in Mode B.
- `references/examples.md` — Worked before/after examples for each type. Read if you need calibration on scoring or style.
