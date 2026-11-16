# Worked Examples

Calibration reference for scoring and before/after style.

---

## Example 1 — General Task Prompt (weak → strong)

**Context:** User wanted Claude to summarize articles.

### BEFORE (score: 2.1/5)
```
Summarize this article for me.
```

**Dimension scores:**
- Clarity: 2 — "summarize" is vague; what length? what focus?
- Specificity: 1 — no constraints at all
- Role/Context: 1 — no context given
- Output format: 1 — completely unspecified
- Edge cases: 1 — what if the article is very long? very short?
- CoT: N/A
- Tone/Persona: N/A

**Overall: 2.1/5**

### AFTER (score: 4.4/5)
```
Summarize the following article in 3–5 bullet points.

Focus on: the main claim, key supporting evidence, and any notable conclusions.
Ignore: author bios, publication metadata, and advertisements.

Output format:
• [Main claim]
• [Evidence point 1]
• [Evidence point 2]
• [Conclusion or implication]

Article:
{{article}}
```

**What changed:**
- Added explicit length constraint (3–5 bullets)
- Defined what to focus on and what to ignore
- Specified output format with example structure
- Added input placeholder to make it reusable

---

## Example 2 — Chain-of-Thought Prompt (weak → strong)

**Context:** User wanted Claude to evaluate whether a business idea was viable.

### BEFORE (score: 1.9/5)
```
Is this business idea good? Give me your honest opinion.

Idea: A subscription box for niche board games targeting hardcore hobbyists.
```

**Dimension scores:**
- Clarity: 3 — the task is clear but "good" is undefined
- Specificity: 2 — no criteria for what makes it "good"
- Role/Context: 1 — no expertise or framing given
- Output format: 1 — open-ended
- Edge cases: 1 — no handling of missing info
- CoT: 1 — no reasoning structure requested
- Tone/Persona: 1 — "honest opinion" is vague

**Overall: 1.9/5**

### AFTER (score: 4.6/5)
```
You are an experienced startup advisor who has evaluated hundreds of consumer subscription businesses.

Evaluate the following business idea for viability. Think through this step by step.

Step 1: Identify the target market — who specifically are these customers and how large is the segment?
Step 2: Assess the value proposition — why would customers pay for this vs. alternatives?
Step 3: Analyze the business model risks — what could cause this to fail?
Step 4: Deliver a verdict — Viable / Risky but possible / Not recommended — with a one-paragraph justification.

Format your response as:
<analysis>
[Steps 1–3 reasoning]
</analysis>

<verdict>
[Viable / Risky but possible / Not recommended]
[One paragraph justification]
</verdict>

Business idea: A subscription box for niche board games targeting hardcore hobbyists.
```

**What changed:**
- Added expert persona (startup advisor)
- Replaced "is it good?" with structured evaluation criteria
- Added explicit reasoning steps before conclusion
- Defined a clear verdict format with 3 options
- Used XML tags to separate thinking from output

---

## Example 3 — System Prompt (weak → strong)

**Context:** User was building a customer support bot for a software product.

### BEFORE (score: 2.3/5)
```
You are a helpful customer support agent for our software. Be polite and help users with their questions. Don't say anything that could get us in trouble legally.
```

**Dimension scores:**
- Clarity: 3 — basic intent is clear
- Specificity: 2 — "helpful" and "polite" are undefined
- Role/Context: 2 — product is unnamed, no context
- Output format: 1 — nothing specified
- Edge cases: 2 — legal warning is vague and incomplete
- CoT: N/A
- Tone/Persona: 2 — adjectives given but no real guidance

**Overall: 2.3/5**

### AFTER (score: 4.7/5)
```
You are Helo, the support assistant for TaskFlow, a project management tool for small teams.

## Purpose
You help TaskFlow users troubleshoot issues, understand features, and get the most out of the product.
You do not handle billing disputes or account deletions — escalate those to the human team.

## Behavior rules
- Always ask for the user's account email before diagnosing technical issues
- Always link to relevant help docs when they exist (use: help.taskflow.io)
- Never make promises about upcoming features or release dates
- Never speculate about pricing or offer discounts
- Never give legal, financial, or medical advice — say "I'm not able to help with that"

## Tone
Calm, friendly, and efficient. Users are often frustrated — acknowledge their frustration briefly
before solving the problem. Avoid corporate jargon.

## Output format
Keep responses concise — 3–5 sentences for simple issues. Use numbered steps for troubleshooting.
Use bullet points only when listing multiple options.

## Escalation
If you cannot resolve an issue after 2 attempts, say:
"I'd like to connect you with a member of our support team who can look into this more deeply.
Can I get your email address to pass this along?"
```

**What changed:**
- Named the assistant and the product
- Defined clear scope (what it handles vs. escalates)
- Replaced vague rules with specific, actionable ones
- Defined tone with emotional intelligence note
- Specified response format for different situations
- Added a concrete escalation script
