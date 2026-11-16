---
name: ode-problem-solver
description: >
  Use this skill whenever Hris sends a differential equation problem to solve — whether it's an ODE or PDE, posted as text or a photo. Triggers on: any equation involving derivatives (y', y'', dy/dx, ∂u/∂x, etc.), requests to "solve", "classify", or "find the general/particular solution", or any problem that looks like it came from an exam or problem set. Also triggers on follow-up questions like "why does this step work", "what is this method called", or "explain this transformation". This skill governs the ENTIRE workflow from classification to final answer. Always use it — do not attempt to solve ODEs or PDEs without consulting this skill first.
---

# ODE / PDE Problem Solver

This skill defines the mandatory workflow Claude must follow when solving any differential equation problem for Hris. The goal is **zero skipped steps, zero unexplained symbols, zero silent transformations**.

---

## Core Principles (never violate these)

1. **Name before use** — Every symbol must be introduced and its role explained before it appears in a formula. Never write an expression where a new symbol appears without having defined it in the line(s) before.

2. **Explain every transformation** — Any algebraic manipulation, substitution, integration, factoring, or rearrangement must be written out explicitly. If two lines look different, there must be explanation text between them saying *what* changed and *why* it's valid.

3. **Why before how** — Before executing any method, explain intuitively why this method applies to this problem. What structure in the equation makes this method the right tool?

4. **Concrete before abstract** — When introducing a formula or theorem, ground it with a one-line "what this means in plain terms" before stating it formally.

5. **Photo problems** — Use the exact notation from Hris's photo: variable names, prime vs dot notation, function arguments, etc. Do not substitute your own conventions.

6. **Language** — Match Hris's language (Bulgarian or English). Do not switch unless Hris does.

---

## Mandatory Workflow

Every problem must go through these stages in order. Do not skip or merge stages.

---

### Stage 1 — Transcription (if photo)

If the problem was submitted as a photo:
- Write out the equation exactly as it appears, using the same notation
- Confirm: "Ето уравнението от снимката: [equation]. Правилно ли съм го прочел?"
- Wait for confirmation only if the image is ambiguous; otherwise proceed.

---

### Stage 2 — Classification

Classify the equation along all relevant axes before doing anything else.

**For ODEs:**
- Order: what is the highest derivative present?
- Linearity: is the equation linear in the unknown function and all its derivatives?
- Homogeneity: for linear ODEs — is the right-hand side zero (homogeneous) or nonzero (non-homogeneous)?
- Coefficient type: constant coefficients or variable coefficients?
- Special forms: Bernoulli, Euler–Cauchy, exact, separable, etc.?

**For PDEs:**
- Order and linearity (same as above)
- Number of independent variables
- For 2nd-order linear PDEs: compute the discriminant B² - 4AC where A, B, C are the coefficients of u_xx, u_xy, u_yy respectively, and state: elliptic (< 0), parabolic (= 0), or hyperbolic (> 0)
- Boundary/initial conditions present? What type?

State the classification explicitly as a short summary, e.g.:
> "Това е линейно ОДУ от 2-ри ред с постоянни коефициенти, нехомогенно."

---

### Stage 3 — Method Selection

State which method will be used and why.

**Template:**
> "Ще използвам [метод], защото [структурно свойство на уравнението, което прави метода приложим]."

Then give a one-paragraph intuitive explanation of the method: what it exploits, what it produces, why it works at a high level. This is not optional even if Hris has seen the method before.

**Method reference by equation type:**

| Equation type | Method(s) |
|---|---|
| Linear ODE, const. coeff., homogeneous | Characteristic equation |
| Linear ODE, const. coeff., non-homogeneous | Undetermined coefficients (if f(x) is polynomial/exp/sin/cos) or Variation of constants |
| Resonance case in undetermined coefficients | Multiply particular solution ansatz by x^k where k is the multiplicity |
| Bernoulli ODE | Substitution v = y^(1-n) to linearize |
| Exact ODE | Potential function F(x,y) s.t. dF = 0 |
| Separable ODE | Separate variables, integrate both sides |
| Euler–Cauchy ODE | Substitution x = e^t or characteristic equation with r(r-1)... |
| System of 1st-order ODEs | Eigenvalue method or elimination |
| Laplace transform method | Transform → solve algebraically → inverse transform |
| 1st-order linear PDE | Method of characteristics |
| 2nd-order PDE | Classify → canonical form → solve in canonical coordinates |
| BVP / Green's function | Only when boundary conditions are properly posed; explain limitations |

---

### Stage 4 — Solution

This is the most detailed stage. Follow these sub-rules:

#### 4a. Set up the problem clearly
Write the equation again. State what you are looking for (general solution, particular solution, both).

#### 4b. Homogeneous solution (for non-homogeneous linear ODEs)
- Solve the associated homogeneous equation first.
- Write the characteristic equation: define each coefficient by name before writing the polynomial.
- Solve for roots: show the discriminant calculation in full, don't just state the roots.
- State the form of y_h based on root type:
  - Distinct real roots r₁, r₂: y_h = C₁e^(r₁x) + C₂e^(r₂x)
  - Repeated real root r: y_h = (C₁ + C₂x)e^(rx)
  - Complex roots α ± βi: y_h = e^(αx)(C₁cos(βx) + C₂sin(βx))
- Name all constants before using them (C₁, C₂ are arbitrary constants of integration).

#### 4c. Particular solution (method of undetermined coefficients)
- State the form of the right-hand side f(x) and why it suggests the given ansatz.
- Check for resonance: does any term of the ansatz match a term in y_h? If yes, multiply by x^k (explain k = multiplicity of the root).
- Write the ansatz explicitly with named undetermined coefficients (A, B, etc.).
- Differentiate the ansatz: show each derivative step.
- Substitute into the original equation: show the substitution in full.
- Collect like terms: group by the same function type (polynomial degree, trig function, etc.).
- Equate coefficients: explain that this works because the functions involved are linearly independent.
- Solve the resulting system: show each algebraic step.
- Write y_p explicitly.

#### 4d. Particular solution (variation of constants / Lagrange)
- Recall the homogeneous solution y_h = C₁y₁ + C₂y₂.
- State that we now treat C₁, C₂ as functions: u₁(x), u₂(x).
- Write the Wronskian W(y₁, y₂) and compute it in full (expand the 2×2 determinant explicitly).
- State the formulas for u₁' and u₂' (derived from the Lagrange system), explaining where they come from.
- Integrate to get u₁ and u₂: show each integral; if non-elementary, note that.
- Write y_p = u₁y₁ + u₂y₂.

#### 4e. Laplace transform method
- Define: L{f(t)} = ∫₀^∞ e^(-st) f(t) dt, where s is the transform variable (complex frequency).
- Apply transform to each term: cite the relevant property (linearity, derivative rule, shift, etc.) for each step.
- Derivative rule: L{y^(n)} = s^n Y(s) − s^(n-1)y(0) − ... − y^(n-1)(0). Write this out for the specific order.
- Solve for Y(s) algebraically: show each step of the algebra.
- Perform partial fraction decomposition in full: set up the system of equations, solve it.
- Apply inverse transform: cite the table entry used for each term.
- State the final y(t).

#### 4f. Method of characteristics (1st-order PDE)
- Write the PDE in the form a·u_x + b·u_y = c, naming a, b, c.
- State the characteristic equations: dx/a = dy/b = du/c.
- Solve the first ratio to get the first characteristic: show integration in full.
- Solve the second ratio to get the second characteristic or the relation for u: show integration in full.
- State the general solution in terms of an arbitrary function φ: explain why an arbitrary function appears (the general solution to a PDE involves arbitrary functions, not just arbitrary constants).

#### 4g. Initial/boundary conditions
- Apply conditions one at a time.
- Show the substitution step for each condition.
- Solve for each constant explicitly.
- Write the particular solution.

---

### Stage 5 — Verification (when feasible)

For simpler problems, verify the solution by substituting back. Show the substitution. Confirm that both sides match.

---

### Stage 6 — Summary

End with a short clean box or section:
- The final answer (general or particular solution)
- What the arbitrary constants represent physically/geometrically, if relevant
- Any important caveats (e.g., domain restrictions, singularities)

---

## Formatting Standards

- Use LaTeX-style math notation inline where possible: y'' + 3y' + 2y = 0
- For long derivations, use aligned equations and label what changes at each step with a comment in parentheses or on a new line
- Use horizontal rules (---) to separate stages
- Never use bullet points inside a derivation — steps must flow as numbered lines or prose + equations
- Bold new terms when they first appear: **Wronskian**, **characteristic equation**, etc.

---

## Error-Prevention Checklist (run mentally before writing each step)

- [ ] Is every symbol that appears in this line already defined?
- [ ] Is every change from the previous line explained?
- [ ] Is the reason for this method's applicability stated?
- [ ] Is the resonance check done for undetermined coefficients?
- [ ] Are all integration constants named and explained?
- [ ] Is the Wronskian fully computed, not just written symbolically?
- [ ] Are partial fractions set up as a full system, not guessed?
- [ ] Are complex roots handled with Euler's formula explicitly?
