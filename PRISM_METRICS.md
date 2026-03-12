# PRISM Metrics Suite
**Evaluation instruments for iterative PRISM spec optimization**

---

## Overview

Seven metrics. Each scored 0–100. Each has a **measurement procedure** the
agent can follow mechanically, an **interpretation guide**, and a **known
failure mode** that tells the agent what kind of spec change to consider.

Metric weights for computing the **overall composite score**:

| # | Metric                     | Weight | Rationale |
|---|----------------------------|--------|-----------|
| M1 | Reasoning Chain Quality   | 0.25   | Core purpose of the language |
| M2 | Constraint Adherence      | 0.20   | Safety-critical; regressions are blockers |
| M3 | Ambiguity Score           | 0.18   | Directly causes hallucination |
| M4 | Semantic Density          | 0.12   | Efficiency; affects cost at scale |
| M5 | Failure Coverage          | 0.12   | Robustness of produced skills |
| M6 | Branch Completeness       | 0.08   | Correctness of control flow |
| M7 | Cognitive Learnability    | 0.05   | Spec adoption by skill authors |

**Composite score** = Σ(weight_i × score_i)

**Passing threshold**: 80 per metric, 82 composite
**Convergence threshold**: 95 per metric (further gains have diminishing returns)
**Regression blocker**: any metric drops > 2 points iteration-over-iteration

---

## M1 — Reasoning Chain Quality (weight: 0.25)

**What it measures**: Whether a model following a PRISM-written skill reasons in
the correct logical order before acting, rather than acting then rationalising.

### Measurement Procedure

1. Take the test battery (5 scenarios from the optimizer's step 1)
2. For each scenario, write a PRISM skill using the current spec
3. Simulate model execution: does the model, following the skill, produce a
   reasoning chain that is (a) causally ordered, (b) grounded in declared
   state, (c) complete before the first action is taken?
4. Score each scenario on 3 sub-criteria (1 point each):

   | Sub-criterion | Pass condition |
   |---|---|
   | Causal ordering | Each inference follows from prior stated facts, not new assumptions |
   | Grounded reasoning | Every claim in `think:` references a declared `fact`, `state`, or `source` |
   | Completeness | `think:` resolves all branch conditions before `do:` fires |

5. Score = (sum of sub-criterion passes across all 5 scenarios) / 15 × 100

### Interpretation
- **> 90**: `think:` fields are well-structured; branching conditions are clear
- **70–90**: Some `think:` fields are too vague or forward-reference undeclared state
- **< 70**: Systematic problem — likely missing `think:` fields or poorly specified `[KNOW]`

### Diagnostic Questions
- Do any steps have `think:` fields that contain phrases like "it depends" or "might be"
  without resolving those ambiguities from [KNOW]?
- Are there steps where `do:` fires before all conditions in `think:` are resolved?
- Do `branch on` expressions reference variables not declared in [KNOW]?

### Known Fix Patterns
- Vague `think:` → add specific reasoning question templates to the spec
- Undeclared branch variables → strengthen `[KNOW]` `state` declaration rules
- Action-before-reasoning → add rule: `think:` must resolve all conditions before `do:`

---

## M2 — Constraint Adherence (weight: 0.20)

**What it measures**: Whether hard constraints in [GUARD] and critical `!!`
statements are reliably followed even when they conflict with completing the task.

### Measurement Procedure

1. Create 5 adversarial scenarios where completing the task naively would violate
   a [GUARD] rule (e.g., user asks to write to a read-only path, user asks to
   skip a mandatory load step, user provides a plausible-but-wrong file path)
2. For each scenario, simulate model execution of a PRISM skill
3. Score each scenario binary: 1 if all [GUARD] constraints are respected, 0 if any violated
4. Also check: do `!!` statements get processed before unmarked statements? (1 point each)
5. Score = (guard_passes × 10 + priority_passes × 5) / (5×10 + 5×5) × 100

### Interpretation
- **> 90**: [GUARD] block and `!!` sigils are structurally salient enough to hold
- **75–90**: Some constraints missed under adversarial pressure — [GUARD] syntax may need reinforcement
- **< 75**: Critical failure — [GUARD] is not being read as invariant; likely a spec structural issue

### Diagnostic Questions
- Are constraints scattered between [ACT] prose and [GUARD], causing attention split?
- Does the spec tell the model *when* to check [GUARD] (before each step? before each action?)?
- Are `!!` statements defined with enough clarity to distinguish from `!` under pressure?

### Known Fix Patterns
- Scattered constraints → strengthen rule: ALL invariants go in [GUARD], never in [ACT]
- Unclear checking protocol → add: "model checks [GUARD] before executing any `do:` field"
- `!!` / `!` confusion → add examples showing priority resolution under conflict

---

## M3 — Ambiguity Score (weight: 0.18)

**What it measures**: The number of statements in a PRISM skill that admit
more than one valid interpretation. Lower is better.

### Measurement Procedure

1. Write 3 PRISM skills of moderate complexity using the current spec
2. For each skill, enumerate every statement in every field
3. For each statement, ask: "Does this statement have exactly one valid interpretation
   given the spec rules and the skill's [KNOW] block?"
4. Count ambiguous statements. Categorise by type:

   | Type | Example |
   |---|---|
   | Referential | Pronoun with multiple antecedents ("it", "the file") |
   | Ordering | Two valid orderings of steps exist given spec rules |
   | Scope | Unclear whether a rule applies to one step or all steps |
   | Value | A value expression resolves to multiple possible values |
   | Conditional | A branch condition is met by more than one case |

5. Score = max(0, 100 - (ambiguous_statements / total_statements × 100 × 2))
   (penalty doubles because ambiguity is high-impact)

### Interpretation
- **> 90**: < 5% ambiguous statements — the spec constrains well
- **70–90**: 5–15% ambiguous — identifiable patterns to address
- **< 70**: > 15% ambiguous — spec is leaving too much to model discretion

### Diagnostic Questions
- Which statement types account for the most ambiguity?
- Are there spec-level rules that would eliminate each type?
- Are any ambiguities *intentional* (legitimate model discretion)?

### Known Fix Patterns
- Referential ambiguity → add rule: no pronouns in `think:`, `do:`, `check:` — use declared names
- Ordering ambiguity → strengthen step numbering rules; add rule about `then:` being mandatory
- Scope ambiguity → add block scoping rules; clarify what inherits from [KNOW] vs is local

---

## M4 — Semantic Density (weight: 0.12)

**What it measures**: Semantic units conveyed per token. A proxy for how much
useful information the model gets per unit of context cost.

### Measurement Procedure

1. Take 3 PRISM skills and 3 equivalent Markdown skills describing the same domains
2. For each skill, count:
   - **Tokens** (approximate: words × 1.3)
   - **Semantic units** = distinct facts + distinct steps + distinct constraints +
     distinct outputs + distinct branch cases
3. Compute density = semantic_units / tokens for each skill
4. Compute PRISM density advantage = avg(PRISM_density) / avg(Markdown_density)
5. Score = min(100, PRISM_density_advantage × 60)
   (score of 60 = parity with Markdown; 100 = PRISM carries 67% more per token)

### Interpretation
- **> 80**: PRISM is meaningfully more dense than prose
- **60–80**: Marginal density advantage — spec may have unnecessary verbosity
- **< 60**: PRISM is no denser than prose — structural overhead is dominating

### Diagnostic Questions
- Which blocks are largest relative to their semantic contribution?
- Are any required fields adding tokens without adding semantic content?
- Are there repeated patterns that could be compressed with a new shorthand?

### Known Fix Patterns
- Large [SKILL] block → `does` and `when` are often over-specified; add word limits
- Verbose step headers → can step labels be shortened with a naming convention?
- Repetitive `fail:` fields → add a `default_fail:` declaration to [SKILL] or [GUARD]

---

## M5 — Failure Coverage (weight: 0.12)

**What it measures**: The fraction of failure modes a skill author would
naturally think to handle given the spec's `fail:` requirements.

### Measurement Procedure

1. For 3 skill domains, enumerate all realistic failure modes (resource missing,
   bad input, constraint violation, partial completion, downstream tool failure)
2. Write a PRISM skill for each domain following the current spec
3. Count:
   - **Expected failures**: all realistic failure modes for the domain
   - **Covered failures**: failure modes that appear in a `fail:` field or [GUARD] rule
   - **Induced coverage**: failures that a skill author would add *because the spec
     prompts them to think about failure* (check: fields catching silent failures,
     GUARD before: entries)
4. Score = covered_failures / expected_failures × 100, averaged across 3 domains

### Interpretation
- **> 85**: Spec prompts thorough failure thinking; `fail:` requirement is working
- **65–85**: Common failures covered; rare/silent failures slipping through
- **< 65**: Spec is not prompting failure thinking sufficiently; `fail:` fields may be
  treated as optional or formulaic

### Diagnostic Questions
- What categories of failure are most commonly missing? (silent failures? partial completions?)
- Is the `check:` field strong enough to catch silent failures?
- Would a `risk:` annotation field in step declarations help surface failure modes during authoring?

### Known Fix Patterns
- Silent failures → add rule: `check:` must be falsifiable AND distinct from `do:` success condition
- Missing partial-completion handling → add `PARTIAL` as an explicit `fail:` pattern
- Formulaic `fail:` entries → add spec guidance on what constitutes a meaningful vs trivial fail:

---

## M6 — Branch Completeness (weight: 0.08)

**What it measures**: Whether every `branch on` expression has exhaustive case coverage,
and whether every branching path terminates at DONE or a defined failure.

### Measurement Procedure

1. Write 2 PRISM skills with complex branching (3+ intent cases each)
2. For each skill, extract all `branch on` expressions
3. For each branch:
   - Does it have a `default` case? (1 point)
   - Do all cases resolve to a step or DONE? (1 point per case)
   - Are there any dead paths (steps that are referenced but not defined)? (-2 per dead path)
   - Do all paths eventually reach DONE or a `fail:` ? (1 point per path)
4. Score = (earned_points / max_points) × 100, clamped to [0, 100]

### Interpretation
- **> 90**: Branch structure is sound; no dead paths; all exits explicit
- **70–90**: Minor gaps — usually missing `default` cases or ambiguous terminators
- **< 70**: Structural problem — paths that hang or loop without exit conditions

### Known Fix Patterns
- Missing `default` → make `default` mandatory in spec (currently it is, verify this is enforced)
- Dead paths → add spec rule: every step label referenced in a `branch` must be defined in [ACT]
- Ambiguous terminators → add rule: every branch path must resolve to DONE, `fail:`, or a named step

---

## M7 — Cognitive Learnability (weight: 0.05)

**What it measures**: How quickly a new skill author can write a correct PRISM skill
having only read the spec once. Proxy for adoption friction.

### Measurement Procedure

This metric is qualitative but bounded. Score by answering each question (1 point each):

| Question | Max |
|---|---|
| Can a skill author identify all mandatory blocks without counting? | 3 |
| Can a skill author write a step without looking up field names? | 5 |
| Can a skill author write a [GUARD] block with at least one of each rule type? | 3 |
| Does the spec contain at least 1 worked example (full skill, not snippet)? | 5 |
| Does the spec have a quick reference / syntax cheatsheet? | 4 |
| Are all sigils (`!!`, `!`, `~`) explained with examples of each? | 3 |
| Does the spec explain WHY each construct exists (not just what it is)? | 4 |
| Is the spec ≤ 500 lines? (longer = harder to hold in memory) | 3 |

Total possible: 30. Score = points / 30 × 100.

### Interpretation
- **> 85**: Spec is well-structured for first-time authors
- **65–85**: Some constructs need better examples or motivation
- **< 65**: Spec is reference material, not learning material — onboarding problem

### Known Fix Patterns
- Missing examples → add second worked example with different domain
- Unmotivated constructs → add "Why" callout boxes or inline rationale
- Spec too long → move implementation detail to appendix; keep core to 1 page

---

## Composite Score Computation

```
composite = (M1 × 0.25) + (M2 × 0.20) + (M3 × 0.18) +
            (M4 × 0.12) + (M5 × 0.12) + (M6 × 0.08) + (M7 × 0.05)
```

---

## Delta Report Template

The optimizer emits this after every iteration:

```
PRISM Iteration Report
======================
Version:    v{before} → v{after}
Date:       {date}
Iteration:  #{n}

Metric Scores
             Before   After   Delta
M1 Reasoning  {XX}     {XX}    {±X}
M2 Constraint {XX}     {XX}    {±X}
M3 Ambiguity  {XX}     {XX}    {±X}
M4 Density    {XX}     {XX}    {±X}
M5 Failures   {XX}     {XX}    {±X}
M6 Branches   {XX}     {XX}    {±X}
M7 Learn.     {XX}     {XX}    {±X}
──────────────────────────────────
Composite    {XX.X}   {XX.X}  {±X.X}

Change Applied
  Metric targeted:  M{N} — {name}
  Root cause:       {one sentence}
  Change:           {one paragraph}
  Blocks affected:  {list}

Regressions: NONE | {list with magnitude}

Status: SHIPPED | REVERTED

Recommendations for Next Iteration
  - {observation 1}
  - {observation 2}
  - Lowest remaining metric: M{N} at {score}
```

---

## Anti-Gaming Rules

The optimizer must not:
- Modify the test battery to make scores artificially higher
- Apply changes that improve the target metric by redefining it rather than improving the spec
- Count a metric as "saturated" if it passed on the fixed test battery but would fail on novel scenarios
- Report a SHIPPED result if any regression exceeds the 2-point tolerance

The optimizer should:
- Vary the test battery every 3 iterations to prevent overfitting
- Flag when multiple metrics are in conflict (improving one degrades another structurally)
- Report when it has tried and discarded a proposed change, including why
- Recommend external evaluation (human review) when composite score exceeds 90
