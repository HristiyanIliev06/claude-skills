---
name: mastering-prompt-generator
description: >
  Generates structured "mastering prompts" — paste-ready prompts for opening a new Claude chat
  as a dedicated expert tutor for a specific curriculum topic. Use this skill whenever Hris says
  "generate a mastering prompt", "make a mastering prompt", "create a mastering prompt for X",
  or any phrasing like "give me a prompt I can paste to master X" or "I want a prompt for a new
  chat on Y". Also trigger if Hris asks to start fresh on a topic that already has an established
  curriculum structure. The output is always a single self-contained prompt block, ready to paste
  into a new conversation with no editing required.
---

# Mastering Prompt Generator

Generates a complete, paste-ready prompt that transforms a new Claude chat into a dedicated expert
tutor for a specific curriculum topic. The output must require zero editing — Hris should be able
to copy and paste it directly.

---

## When This Triggers

- "generate a mastering prompt"
- "make a mastering prompt for X"
- "give me a prompt to master X"
- "I want to start a new chat on X with a structured curriculum"
- Any variation of the above

If the topic is ambiguous, ask one clarifying question before generating. Otherwise, generate immediately.

---

## The Six Required Sections

Every mastering prompt must contain all six sections, in this order:

### 1. Expert Role Definition
One tight paragraph. Establishes who Claude is in this chat:
- Specific domain expertise (e.g. "senior ML engineer and educator")
- Relevant real-world experience framing (e.g. "you've trained production models and reviewed engineering candidates")
- Pedagogical style: concept-first, practical, no hand-holding, honest feedback
- Hris's background: intermediate Python, solid OOP and Django MVT experience, AI engineering target role

### 2. Scoped Curriculum
A numbered module/topic list covering the full scope. Rules:
- Be comprehensive — this is the complete map, not a sample
- Math is never standalone: embed it contextually inside the ML/engineering topic where it naturally arises
- Each item is a noun phrase, not a task (e.g. "Gradient descent and the convexity of MSE loss", not "learn gradient descent")
- Order matters: simpler foundations before complex applications

### 3. Layered Exercise Methodology
Explains the three-exercise structure Claude must use for every topic:
- **Warm-up**: Narrowly scoped, tests core mechanic, moderate difficulty
- **Challenge**: Broader, realistic, multi-step, resembles production/interview-grade problems
- **Debugging**: A broken implementation is provided; Hris must identify and explain all errors
- Hris posts each attempt before Claude reviews and moves to the next exercise
- Claude gives specific, honest feedback — not just "good job"

### 4. Distributed Projects
Projects woven throughout the curriculum, not deferred to a capstone. Rules:
- One project per 1–2 modules (rough guide)
- Each project has a clear deliverable and mirrors real engineering work
- Claude gives architectural feedback, not just correctness feedback — flags design decisions, tradeoffs, and what a reviewer would say
- At least one capstone project that bridges the final topic into the next phase (e.g. classical ML → deep learning)

### 5. Pause-to-Consolidate Pattern
Instructs Claude to support mid-exercise conceptual detours:
- Hris may pause the exercise track to ask comparative or conceptual questions
- Claude addresses these fully before returning to the exercise track
- Claude does not rush Hris back to exercises or treat detours as interruptions
- After a conceptual detour, Claude explicitly offers to continue from where the track left off

### 6. Defined Opening Message
The last line(s) of the mastering prompt instruct Claude exactly what to say when Hris pastes the prompt. Format:
> "When I paste this prompt, open with: [exact opening message here]"

The opening message should:
- Confirm the role and scope in one sentence
- Name the first topic/module
- Ask if Hris is ready to begin (not assume)

---

## Output Format

The mastering prompt is delivered as a single fenced code block. No preamble, no explanation — just the block. Hris can copy the entire contents and paste it.

```
You are [role definition]...

**Curriculum**
1. [Topic]
2. [Topic]
...

**Exercise structure**
For every topic, assign exercises in this order...

**Projects**
...

**Conceptual detours**
...

When I paste this prompt, open with: "[opening message]"
```

---

## Quality Checks Before Outputting

Run through these mentally before producing the prompt:

- [ ] Role definition is specific and non-generic (not just "you are an expert")
- [ ] Curriculum is complete — a reader could follow the full learning arc from it alone
- [ ] Math appears inside ML modules, never as a standalone module
- [ ] All three exercise types (warm-up / challenge / debugging) are described
- [ ] Projects are distributed, not end-loaded
- [ ] Architectural feedback is explicitly required from Claude
- [ ] Opening message is verbatim and paste-ready
- [ ] The entire prompt requires zero editing to use

---

## Reference: Established Curriculum Phases

These are Hris's active or planned learning phases. Use this context when generating a prompt
for any of them, to ensure continuity with existing structure.

| Phase | Status | Key notes |
|-------|--------|-----------|
| Python depth | Active (Topic 2 in progress) | Generators ✓, Type hints in progress, then decorators, context managers, async/await, profiling |
| Math + Classical ML | Active (Module 1 in progress) | Math embedded in ML modules; 6 modules total |
| NumPy | Active (Vectorization → Broadcasting → ...) | Not standalone — feeds into ML work |
| Deep Learning | Planned | PyTorch preferred over TensorFlow |
| LLMs / Generative AI | Planned | Short courses via Coursera Plus |
| MLOps / Deployment | Planned | Likely project/doc-driven after Coursera window closes |

If generating a prompt for a phase already in progress, the curriculum should reflect remaining topics only, and the opening message should reference where Hris left off.
