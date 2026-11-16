---
name: error-isolation-prompt-generator
description: >
  Generates a paste-ready prompt that isolates an in-progress error, failure, or emergency
  (e.g. a Git authentication failure, a broken CI pipeline, a credential helper loop, any
  multi-turn troubleshooting thread that's gone in circles) into a clean, standalone diagnostic
  prompt for a fresh chat. Use this skill whenever Hris says "isolate error handling", "isolate
  this error", "make an error isolation prompt", "pull this into its own prompt", "give me a
  prompt to debug this separately", or any similar phrasing asking to extract troubleshooting
  context out of the current conversation. Also trigger proactively if Hris is several back-and-forth
  troubleshooting turns deep on an error and says something like "let's start fresh on this" or
  "I want to take this somewhere else." The output is always a single self-contained prompt block,
  ready to paste into a new conversation with no editing required.
---

# Error Isolation Prompt Generator

Generates a complete, paste-ready prompt that extracts a specific error or emergency-handling
thread out of a larger, possibly meandering conversation and reframes it as a clean, standalone
diagnostic brief for a fresh Claude chat. The output must require zero editing — Hris should be
able to copy and paste it directly into a new conversation.

This exists because troubleshooting threads tend to sprawl: the original task gets buried under
five rounds of "try this... no, that didn't work... what about this instead." A fresh chat with
a tight, well-structured brief gets a faster and cleaner diagnosis than continuing in a thread
that's accumulated noise.

---

## When This Triggers

- "isolate error handling"
- "isolate this error"
- "isolate the [X] issue"
- "make an error isolation prompt"
- "pull this into its own prompt"
- "give me a prompt to debug this separately"
- "start fresh on this error"
- "I want to take this to a new chat"
- Any clear variant of extracting an in-progress troubleshooting/emergency thread into a
  standalone prompt

If it's unclear which error or thread to isolate (e.g. multiple unresolved issues exist in the
conversation), ask one clarifying question before generating. Otherwise, generate immediately
using the most recent unresolved error/emergency thread in context.

---

## Step 1: Reconstruct the Incident from Conversation History

Before writing the prompt, scan back through the conversation and extract:

1. **The original goal** — what was Hris trying to accomplish when the error first appeared?
   (e.g. "push a squashed commit to a forked repo as part of a Git exam workflow")
2. **The triggering action** — the exact command, click, or step that produced the error.
3. **The exact error text(s)** — verbatim, in the order they appeared. If the error evolved
   across turns (error A led to trying fix X, which produced error B), capture that chain.
4. **Fixes already attempted** — every remedy already tried, in order, with outcome
   (worked / didn't work / unknown — never verified). Do not omit attempts that failed; they
   are exclusionary signal for the next diagnosis.
5. **Environment facts already known** — OS, tools/CLI versions, credential managers, repo
   ownership, network constraints, anything stated as fact during the thread. Do not
   invent or assume anything not actually stated.
6. **Open diagnostic questions** — anything the conversation was about to ask or check but
   didn't get an answer to yet (e.g. "what does `git config --global --list | grep credential`
   return?").

If any of these six are genuinely absent from the conversation (e.g. no fixes have been
attempted yet), state that explicitly in the relevant section rather than fabricating content —
write "No fixes attempted yet" rather than inventing one.

---

## Step 2: Write the Isolated Prompt

The generated prompt must contain these sections, in this order:

### 1. Context Framing (one paragraph)
Tells the new chat what kind of session this is and what NOT to do:
- States plainly this is a single isolated technical problem, not a request to redo prior
  unrelated work
- Instructs Claude to focus exclusively on resolving the stated error — no tangential
  suggestions, no re-explaining concepts already demonstrated as understood
- Notes the broader task the error interrupted, in one sentence, for situational context only

### 2. Goal
One or two sentences: what the person is actually trying to accomplish (the task the error is
blocking), independent of the error itself.

### 3. Environment
A short bullet list of every confirmed environment fact pulled in Step 1. Mark anything inferred
rather than confirmed with "(unconfirmed)".

### 4. Error Timeline
A numbered sequence:
1. Action taken →
2. Exact error received (verbatim, in a code block)
3. Repeat for each subsequent attempt/error in the chain

This must preserve the actual chronology — don't compress or summarize away the sequence, since
the order of failed attempts is diagnostic information.

### 5. Already Ruled Out
A bullet list of fixes already attempted and their outcomes. This prevents the new chat from
re-suggesting the same dead ends. Format: `Tried: [fix] → Result: [outcome]`.

### 6. Immediate Ask
A direct, specific final instruction telling Claude exactly what's needed next — e.g. "Diagnose
why push authentication is failing silently with no credential prompt, and give the next
diagnostic command to run" — not a vague "please help."

---

## Output Format

The isolated prompt is delivered as a single fenced code block. No preamble, no explanation —
just the block, optionally preceded by one short sentence noting it's ready to paste.

```
This is an isolated technical troubleshooting session. [one-sentence framing — what broader
task this interrupted, and an instruction to stay focused only on resolving this error].

**Goal**
[what the person is trying to accomplish]

**Environment**
- [fact]
- [fact]
- [fact, marked (unconfirmed) if inferred]

**Error timeline**
1. Ran: `[command/action]`
   Got:
   ```
   [verbatim error]
   ```
2. Tried: `[next action]`
   Got:
   ```
   [verbatim error]
   ```
[continue for full chain]

**Already ruled out**
- Tried: [fix] → Result: [outcome]
- Tried: [fix] → Result: [outcome]

**What I need**
[specific, direct ask]
```

---

## Quality Checks Before Outputting

- [ ] Original task is mentioned exactly once, for context, and never re-litigated
- [ ] Every error message is verbatim, not paraphrased
- [ ] The chronological order of attempts is preserved, not reordered for tidiness
- [ ] Nothing is fabricated — absent information is marked as absent or unconfirmed, never invented
- [ ] Already-ruled-out fixes are listed so the new chat doesn't repeat them
- [ ] The final ask is specific and actionable, not generic ("please help debug this")
- [ ] The entire prompt requires zero editing to paste and use
