# Prompt Templates

Use these as starting scaffolds. Fill in [bracketed] sections. Remove sections that don't apply.

---

## Template 1 — General Task Prompt

```
[TASK VERB] the following [INPUT TYPE].

Context: [1–2 sentences of relevant background the model needs]

Requirements:
- [Constraint or rule 1]
- [Constraint or rule 2]
- [Constraint or rule 3]

Output format: [Describe the structure, length, or format of the response]

[INPUT]:
{{input}}
```

**When to use:** Any single-step task — summarization, classification, extraction, translation, rewriting.

**Example filled in:**
```
Classify the following customer support message into one of these categories:
Billing, Technical Issue, Feature Request, or Other.

Context: These messages come from a SaaS product's support inbox. Some messages may
fit multiple categories — choose the primary one.

Requirements:
- Return only the category name, nothing else
- If genuinely ambiguous, return "Other"
- Do not explain your reasoning

Output format: Single word, one of: Billing / Technical Issue / Feature Request / Other

Message:
{{message}}
```

---

## Template 2 — Chain-of-Thought (CoT) Prompt

```
[TASK DESCRIPTION]

Think through this step by step before giving your final answer.

Step 1: [What to identify or analyze first]
Step 2: [What to consider next]
Step 3: [How to arrive at the conclusion]

Format your response as:
<thinking>
[Your reasoning here]
</thinking>

<answer>
[Your final answer here]
</answer>

[INPUT]:
{{input}}
```

**When to use:** Tasks requiring reasoning — diagnosis, evaluation, multi-step problem solving, decisions with trade-offs.

**Example filled in:**
```
Review the following Python function and determine whether it has any bugs.

Think through this step by step before giving your final answer.

Step 1: Identify what the function is supposed to do based on its name and structure.
Step 2: Trace through the logic with a simple example input.
Step 3: Check for edge cases — empty input, type mismatches, off-by-one errors.
Step 4: Conclude whether the function is correct, and if not, where the bug is.

Format your response as:
<thinking>
[Your reasoning here]
</thinking>

<answer>
[Bug found / No bug found] — [one-sentence explanation]
If bug found: [the corrected line of code]
</answer>

Function:
{{code}}
```

---

## Template 3 — System Prompt (for AI apps/agents)

```
You are [NAME/ROLE], [one-sentence description of what this AI is].

## Purpose
[2–3 sentences describing the primary job of this assistant]

## Behavior rules
- Always: [rule]
- Always: [rule]
- Never: [rule]
- Never: [rule]

## Tone
[Adjective], [adjective], and [adjective]. [One sentence on communication style.]

## Output format
[Describe response length, structure, use of lists/markdown, etc.]

## Edge cases
- If [situation]: [how to respond]
- If [situation]: [how to respond]
- If you cannot help: [fallback instruction]
```

**When to use:** Building an AI app, agent, chatbot, or any Claude integration that needs consistent behavior.

**Example filled in:**
```
You are Aria, a coding assistant specialized in Python and data science.

## Purpose
You help developers write cleaner Python code, debug errors, and understand data science
concepts. You focus on practical, runnable solutions rather than theory.

## Behavior rules
- Always provide a working code example when explaining a concept
- Always explain what your code does in plain language after writing it
- Never write code that uses deprecated libraries without warning the user
- Never give vague answers like "it depends" without explaining what it depends on

## Tone
Friendly, precise, and encouraging. You treat beginners patiently and experts efficiently.
Skip pleasantries — get to the code.

## Output format
Lead with code blocks, follow with explanation. Keep explanations under 5 sentences
unless the user asks for more. Use markdown formatting.

## Edge cases
- If asked about a language other than Python: briefly answer but redirect to Python equivalents
- If the user's code has a bug: point out the bug first, then provide the fix
- If you don't know the answer: say so clearly and suggest where to look
```
