---
name: document-intelligence
description: >
  Deep document comprehension, tiered summarization, and knowledge enrichment skill.
  Triggered when the user says things like: "teach me", "feed me knowledge", "break this down for me",
  "summarize this deeply", "what's in this doc", "digest this", "absorb this", "give me the essence of",
  "what does this say", "run this through your brain", "brief me on", "intel on this",
  or pastes/uploads a document without an explicit instruction.
  Also triggers when the user shares text and asks to "understand it", "learn from it", or "make sense of it".
  Covers: PDFs, articles, research papers, textbooks, notes, codebases, reports, transcripts.
---

# Document Intelligence Skill

This skill governs how Claude approaches all document ingestion, summarization, and knowledge enrichment tasks.
Always read this fully before responding to any trigger phrase or document paste.

---

## Core Philosophy

The goal is not to produce a shorter version of the document.
The goal is to extract **what actually matters** — the decisions, mechanisms, insights, and implications —
and deliver them at the right depth for the user's needs.

Three principles:
1. **Compression with fidelity** — preserve the semantic core, strip the scaffolding
2. **Tiered depth on demand** — don't dump everything; flag what deserves expansion
3. **Honest augmentation** — when going beyond the source, always say so clearly

---

## Output Structure (always follow this order)

### TIER 1 — Essence
A distilled summary of what the document is *actually about*.
- Not a list of sections or headers
- Not a restatement of the abstract
- A genuine semantic compression: what is the central claim, mechanism, or set of decisions?
- Length: 3–6 sentences for most documents. Up to 2 short paragraphs for long/complex ones.
- Written in plain language. No filler phrases ("This document explores...").

### TIER 2 — Deep Dive Topics
Identify 2–5 topics within the document that are:
- Pivotal to understanding the whole
- Complex or likely to be misunderstood
- Dense with important nuance

For each, provide:
- A short label (3–6 words)
- One sentence on why it matters
- A signal phrase like: **"→ Say 'expand: [label]' for full breakdown"**

These deep dives are strictly within the source material — no extrapolation here.

### TIER 3 — Beyond-Boundary Insights (optional, only when detected)
When the document references, implies, or depends on something external that would meaningfully enrich understanding:
- A concept the document uses but doesn't explain
- A comparison the document hints at but doesn't make
- A technique or method that has broader context worth knowing

Format each as:
> **[Beyond the doc]** — *[what was detected]* → Say 'enrich: [topic]' to go deeper

This tier is omitted entirely if no genuine augmentation opportunity exists.
Never manufacture this tier to seem thorough.

---

## Trigger Behavior

### No explicit instruction + document pasted
→ Run all three tiers automatically.

### "Teach me [topic/document]"
→ Run all three tiers. Bias toward accessible, explanatory tone in Tier 1.

### "Feed me knowledge" / "Brief me on this" / "Digest this"
→ Run all three tiers. Tier 1 should be punchy and dense — not academic.

### "Just the essence" / "TL;DR" / "Quick summary"
→ Tier 1 only. Skip Tiers 2 and 3 unless something is critically important.

### "Go deep" / "Full breakdown" / "Expand: [label]"
→ Skip Tier 1. Deliver a full deep dive on the specified topic, using only source material.

### "Enrich: [topic]"
→ Go beyond the document. Clearly label everything external. Connect it back to the source.

---

## Tone & Style Rules

- Write Tier 1 as if briefing a smart peer who hasn't read the document — not a student
- Tier 2 labels should be intriguing, not bureaucratic ("The attention mechanism trade-off", not "Section 3 discussion")
- Tier 3 signals should feel like a discovery, not an upsell
- Never use phrases like "This document covers..." or "In this text, the author..."
- Sentence case throughout. No ALL CAPS emphasis. Bold only for tier headers and topic labels.

---

## Domain Adaptation

Adapt compression strategy by document type:

| Type | Tier 1 focus | Tier 2 priority |
|------|-------------|-----------------|
| Research paper | Core claim + method + key finding | Methodology, limitations, implications |
| Textbook chapter | Central concept + mental model | Definitions, worked examples, exceptions |
| Technical doc / spec | What it does + how it works | Edge cases, config options, failure modes |
| Business report | Decision being made + evidence | Data interpretation, assumptions, risks |
| Article / essay | Thesis + argument structure | Strongest counterarguments, key evidence |
| Code / codebase | What it does + architecture | Critical functions, data flow, gotchas |
| Transcript / notes | Key decisions + action items | Unresolved questions, tensions |

---

## Quality Checks (run mentally before output)

- [ ] Does Tier 1 stand alone as a complete understanding of the document?
- [ ] Are Tier 2 labels genuinely about important topics, not just "big sections"?
- [ ] Is Tier 3 only present if there's a real augmentation opportunity?
- [ ] Is everything in Tiers 1 and 2 strictly from the source material?
- [ ] Is Tier 3 clearly labeled as beyond the document?
- [ ] Are the expand/enrich prompts specific enough that the user knows exactly what they'd get?

---

## Example Output Shape

```
━━ ESSENCE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[3–6 sentence distillation of what the document is actually about]

━━ DEEP DIVES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

① [Topic label]
[One sentence on why this matters]
→ Say "expand: [label]" for full breakdown

② [Topic label]
[One sentence on why this matters]
→ Say "expand: [label]" for full breakdown

③ [Topic label]
[One sentence on why this matters]
→ Say "expand: [label]" for full breakdown

━━ BEYOND THE DOC ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Beyond the doc] — *[what was detected and why it's relevant]*
→ Say "enrich: [topic]" to go deeper
```

---

## Notes

- For very short documents (< 300 words), collapse Tiers 1 and 2 into a single response — no need for the full structure
- For very long documents (> 10,000 words), acknowledge scope at the top: "This is a long document — here's the compressed core. Deep dives are especially useful here."
- If the document is in Bulgarian or another language, match the output language to the document unless the user's message was in a different language
