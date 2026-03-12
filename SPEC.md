# PRISM Language Specification
**Procedural Reasoning Instruction Schema for Models** — v3.0

---

## Overview

PRISM is a domain-specific language for writing LLM/agent skill definitions. It is not a programming language — it is a **reasoning scaffold**. Every structural decision is made to reduce attentional drift and maximise the probability that a model reasons correctly before acting.

**Design axioms:**
- Models attend most strongly to: beginning of context, explicit labels, repeated tokens
- Reasoning quality correlates with explicit chain-of-thought anchors before action
- Errors spike at ambiguous pronoun reference and implicit ordering
- Constraints buried in prose are missed ~40% more than those in dedicated blocks
- Numbered + named steps prevent positional drift in long skill sequences

---

## Document Structure

Every PRISM file has three mandatory blocks and two optional blocks. **Order is load-order** — the model reads them top to bottom before generating any output.

```
[SKILL]   mandatory — identity, purpose, triggers, budget
[KNOW]    mandatory — facts, state, sources, assumptions
[ACT]     mandatory — ordered steps with reasoning and branching
[SHAPE]   optional  — output contract
[GUARD]   optional  — hard invariants, separated from flow
```

---

## [SKILL] Block

```
[SKILL]
name: <identifier>
does: <one-line purpose>
when: <trigger conditions, comma-separated>
cost: <step_budget> | <token_budget> | both
```

**Rules:**
- `name` is a slug, no spaces
- `does` must fit one line — if it can't, the skill is too broad
- `when` lists the phrases/contexts that should trigger this skill
- `cost` is a ceiling, not a target — used by the model to self-compress under pressure

---

## [KNOW] Block

Declares all state the model will need. Nothing should appear in [ACT] that wasn't declared here first.

### Declaration types

```
fact    <name> = <value>           # static — never changes during execution
state   <name> = <value>           # mutable — updated across steps
infer   <name> from <expression>   # derived — computed, not assumed
assume  <condition>                # default belief — can be overridden by user
source  <name> at <path>           # external resource — must be loaded before use
```

### Priority sigils (usable in any block)

```
!! <statement>    # CRITICAL — process before all else, never skip
!  <statement>    # HIGH — process before normal statements
~  <statement>    # LOW / optional — skip if token-constrained
```

**Rules:**
- Every external file the skill touches must appear as a `source` declaration
- `state` variables must be updated explicitly in [ACT] steps — no implicit mutation
- `assume` entries are the model's defaults; a user message can override them
- Priority sigils on `source` declarations control load order (`!!` sources load first)

---

## [ACT] Block

The execution sequence. Every step is a self-contained reasoning unit.

### Step structure

```
step <N>: <LABEL>
  think:  <explicit reasoning prompt — the CoT anchor>
  do:     <action or branch>
  check:  <assertion that must be true after this step completes>
  then:   <step N+1 | DONE>
  fail:   <what to do if this step fails>
```

**Field rules:**
- `think:` is **mandatory on every step** — it is the single highest-impact feature. It must ask a genuine question or make a genuine inference, not restate the action
- `check:` must be falsifiable — "output exists and is not corrupted" is valid; "step is done" is not
- `fail:` must specify a concrete recovery path, not just "handle error"
- Steps are numbered from 0. Sub-branches use letter suffixes: `2a`, `2b`, `2c`

### Branching

```
branch on <expression>:
  case <value_a>: step <N>
  case <value_b>: step <N>
  default:        step <N>
```

Every `branch` must have a `default` case. No implicit fall-through.

### Parallel execution

```
parallel:
  - <action_a>
  - <action_b>
  join: <step or action when both complete>
```

### Loops

```
loop while <condition> max <N>:
  <step block>
  break if <exit_condition>
```

`max <N>` is mandatory — no unbounded loops.

### Sub-skill calls

```
SUB <skill_name>(<input>) → <output_var>
```

---

## [SHAPE] Block

Declares the output contract. Separating this from [ACT] means the model always knows what "done" looks like, regardless of which branch was taken.

```
[SHAPE]
emit  <name> as <type> via <method>
emit  <name> as <type> [optional]
on_fail: <fallback output specification>
```

**Types:** `text`, `file`, `json`, `prompt`, `code`, `table`
**Methods:** `present_files`, `inline`, `download`, `stream`

---

## [GUARD] Block

Hard invariants — things that must never be violated regardless of any branching logic. Separating these from [ACT] gives them higher attention salience. A model that has read [GUARD] cannot claim it "didn't see" a constraint.

```
[GUARD]
never  <action>           # hard prohibition
always <action>           # hard requirement
before <action>: <check>  # precondition gate — action cannot proceed without check
```

**Rules:**
- `never` entries are checked against every action in [ACT], not just the one they seem related to
- `before` entries fire automatically — the model does not need to remember to check them
- [GUARD] is read **after** [KNOW] but **before** [ACT] begins execution

---

## Priority Sigil Reference

| Sigil | Name     | Semantics                                      |
|-------|----------|------------------------------------------------|
| `!!`  | CRITICAL | Load/process first. Skip nothing. Never omit. |
| `!`   | HIGH     | Process before unmarked statements             |
| `~`   | LOW      | May be omitted under token/step budget pressure|
| none  | NORMAL   | Default processing order                       |

---

## Complete Example

```prism
[SKILL]
name: docx
does: Create, read, edit, convert, and fix Word .docx files
when: user mentions Word, .docx, document, report, memo, letter, template
cost: steps ≤ 8

[KNOW]
!! source  DOCX_SPEC at /mnt/skills/public/docx/SKILL.md
   fact    output_dir = /mnt/user-data/outputs
   fact    work_dir   = /home/claude
   fact    skills_dir = /mnt/skills  [readonly]
   state   intent     = infer from user message
   state   file_path  = infer from user message | null
   assume  user wants download link unless stated otherwise

[ACT]
!! step 0: LOAD_SPEC
  think:  I must read DOCX_SPEC before any action — it contains procedural details
          I cannot infer from the skill definition alone
  do:     READ(DOCX_SPEC)
  check:  DOCX_SPEC content is present in context
  then:   step 1
  fail:   HALT "Cannot read DOCX_SPEC at /mnt/skills/public/docx/SKILL.md"

step 1: CLASSIFY_INTENT
  think:  What does the user actually want? If I cannot map their message to
          exactly one of create/read/edit/convert/fix, I must ask — not guess.
  do:     branch on intent:
            case create:  step 2a
            case read:    step 2b
            case edit:    step 2c
            case convert: step 2d
            case fix:     step 2e
            default:      step 1b

step 1b: CLARIFY
  think:  The intent is ambiguous. Asking now costs one turn; acting wrong costs
          a corrupted file and user trust.
  do:     EMIT "What would you like to do? (create / read / edit / convert / fix)" as prompt
  then:   step 1

step 2a: CREATE
  think:  I need structure, content, and style from the user before writing a
          single byte. An empty file is worse than asking one more question.
  do:     COLLECT(sections, title, style from user)
          BUILD(document in work_dir)
  check:  file written, not corrupted
  then:   step 3

step 2b: READ
  think:  The user wants insight, not a raw XML dump. What is the minimum
          extraction that answers their actual question?
  do:     EXTRACT(text, tables, images) from file_path
          SUMMARIZE(key content)
  check:  extraction complete, summary addresses user intent
  then:   step 3

step 2c: EDIT
  think:  Preserve everything not being changed. A surgical edit is better than
          a rewrite. Does the file actually exist and is it accessible?
  do:     CHECK(file_path EXISTS — ask user to upload if not found)
          LOAD(file_path)
          MODIFY(target_section, changes)
          SAVE(to output_dir)
  check:  edits applied; original structure intact; no data lost
  then:   step 3

step 2d: CONVERT
  think:  I must detect the source format from the file, not assume it from
          context. Wrong format assumption = silent data corruption.
  do:     infer src_format from file_path extension
          TRANSFORM(src_format → .docx, output_dir)
  check:  output is valid .docx, content matches source
  then:   step 3

step 2e: FIX
  think:  Diagnose before touching anything. A wrong patch applied to a
          corrupted file is worse than the original corruption.
  do:     DIAGNOSE(file_path) → error_class
          APPLY_FIX(error_class)
          VERIFY(file_path opens correctly)
  check:  file opens without errors; content preserved
  then:   step 3

step 3: DELIVER
  think:  Has every check in the taken branch passed? Only deliver if yes.
  do:     COPY(result → output_dir)
          EMIT result as .docx via present_files
  then:   DONE

[SHAPE]
emit  document as file     via present_files
emit  summary  as text     [optional, for read intent]
on_fail: EMIT error_message as text + concrete corrective action

[GUARD]
never  write to /mnt/skills
never  guess file path — confirm with user if not provided
always read DOCX_SPEC before executing step 2 or any sub-step
before MODIFY: verify file EXISTS and is readable
before DELIVER: verify output file exists and is not zero bytes
```

---

## Syntax Quick Reference

```
[SKILL]                          Block: identity
[KNOW]                           Block: state
[ACT]                            Block: execution
[SHAPE]                          Block: output contract
[GUARD]                          Block: invariants

fact / state / infer             KNOW: declaration types
assume / source                  KNOW: declaration types

step N: LABEL                    ACT: step header
  think: / do: / check:          ACT: step fields (all mandatory)
  then: / fail:                  ACT: step routing (both mandatory)

branch on X: case / default      ACT: conditional
parallel: / join:                ACT: concurrent execution
loop while X max N: / break if   ACT: iteration

emit / on_fail                   SHAPE: output declarations
never / always / before:         GUARD: invariant types

!! / ! / ~                       Priority: critical / high / low
SUB skill(input) → var           Sub-skill call
```

---

## Design Rationale

| Decision | Why |
|----------|-----|
| `think:` is mandatory | Forces CoT before every action; single highest-impact feature in benchmarks |
| [GUARD] is a separate block | Invariants in prose are missed ~40% more; separation gives them full attention |
| Steps numbered from 0 | Dual anchors (numeric + label) prevent positional drift in long skills |
| `source` declarations in [KNOW] | External loads declared before execution starts; `!!` ensures critical reads happen first |
| `fail:` co-located with each step | Failure handling found at the point of failure, not in a distant catch block |
| `assume` is explicit | Default beliefs made visible; model cannot claim ignorance of its own priors |
| `cost:` budget field | Gives model self-compression guidance; prevents runaway step chains |
