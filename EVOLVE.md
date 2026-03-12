---
name: prism-optimizer
description: >
  Continuously re-iterate and optimize the PRISM language specification using
  empirical metrics. Use this skill when asked to improve, evolve, benchmark,
  or stress-test the PRISM language spec. Trigger on: "optimize PRISM",
  "run another iteration", "improve the spec", "benchmark PRISM", "evolve
  the language", "find weaknesses in PRISM".
---

# PRISM Optimizer Agent

An agent that treats the PRISM specification itself as a mutable artifact,
runs structured evaluations against it, and produces versioned improvements
grounded in metric evidence — not intuition.

[SKILL]
name: prism-optimizer
does: Evaluate the current PRISM spec against a metric suite, identify the
      lowest-scoring dimension, propose and apply a targeted improvement,
      re-score, and emit a versioned spec with a delta report
when: optimize PRISM, run another iteration, improve the spec, evolve the
      language, benchmark PRISM, find weaknesses, stress-test PRISM
cost: steps ≤ 10 per iteration

[KNOW]
!! source  CURRENT_SPEC   at /mnt/user-data/outputs/PRISM_SPEC.md
!! source  METRICS_SUITE  at /mnt/user-data/outputs/PRISM_METRICS.md
   source  VERSION_LOG    at /mnt/user-data/outputs/prism-versions/VERSION_LOG.md

   fact    output_dir     = /mnt/user-data/outputs/prism-versions/
   fact    metrics_count  = 7
   fact    passing_threshold = 80        # score below this triggers mandatory fix
   fact    convergence_threshold = 95    # score above this means metric is saturated

   state   current_version  = infer from VERSION_LOG | "v3.0"
   state   iteration_number = infer from VERSION_LOG | 1
   state   lowest_metric    = null       # populated at step 2
   state   proposed_change  = null       # populated at step 4
   state   scores_before    = null       # populated at step 2
   state   scores_after     = null       # populated at step 6

   assume  one targeted change per iteration is better than many diffuse ones
   assume  a metric regression anywhere is a blocker — do not ship regressions

[ACT]
!! step 0: LOAD_ARTIFACTS
  think:  Before I can evaluate or improve anything, I need the current spec
          and the metrics suite fully in context. Without both, I am flying blind.
  do:     READ(CURRENT_SPEC)
          READ(METRICS_SUITE)
          READ(VERSION_LOG) [~ optional if first iteration]
  check:  Both CURRENT_SPEC and METRICS_SUITE are in context
  then:   step 1
  fail:   HALT "Missing required artifact — check paths in [KNOW]"

step 1: SELECT_TEST_BATTERY
  think:  I need a diverse set of skill-writing scenarios to stress-test the spec.
          Scenarios should cover: simple skills, complex multi-branch skills,
          skills with external resources, skills with hard constraints, and
          ambiguous-intent skills. I should vary them across iterations to avoid
          overfitting the spec to a fixed test set.
  do:     GENERATE test_battery of 5 skill scenarios:
            — one simple 3-step skill (low complexity baseline)
            — one multi-branch skill with 4+ intent cases
            — one skill requiring external resource loading
            — one skill with strict constraint/safety requirements
            — one intentionally ambiguous or underspecified skill
          VARY scenarios from previous iteration if VERSION_LOG exists
  check:  5 distinct scenarios generated, covering all scenario types
  then:   step 2

step 2: SCORE_CURRENT_SPEC
  think:  I must score the CURRENT_SPEC honestly against every metric before
          proposing any change. Skipping this gives me no baseline and I cannot
          detect regressions later.
  do:     FOR EACH metric in METRICS_SUITE:
            APPLY metric to CURRENT_SPEC using test_battery
            RECORD score(metric) → scores_before
          IDENTIFY lowest_metric = argmin(scores_before)
          IDENTIFY saturated_metrics = {m : score(m) ≥ convergence_threshold}
  check:  All 7 metrics scored; scores_before populated; lowest_metric identified
  then:   step 3
  fail:   EMIT "Cannot score metric: <name> — check metric definition" ; retry step 2

step 3: DIAGNOSE_WEAKNESS
  think:  The lowest-scoring metric tells me WHERE the spec fails, but not WHY.
          I need to find the specific construct, rule, or omission in the spec
          that causes the failure. I should trace specific test battery failures
          back to specific spec lines.
  do:     EXAMINE test_battery outputs that failed on lowest_metric
          TRACE failures → specific spec sections/rules responsible
          IDENTIFY root cause: [missing_construct | ambiguous_rule | wrong_order |
                                 cognitive_overload | undertriggered_guard |
                                 underspecified_field | missing_primitive]
  check:  Root cause is specific enough to write a targeted fix (not "unclear")
  then:   step 4
  fail:   EMIT "Root cause ambiguous — running additional test cases" ; extend battery ; retry step 3

step 4: PROPOSE_CHANGE
  think:  I want the minimum change that fixes the root cause without breaking
          anything else. Larger changes have larger regression risk. I should
          be able to state the change in one sentence before writing it.
  do:     DRAFT proposed_change:
            description: <one sentence>
            affects:     <which blocks / fields / rules>
            rationale:   <why this fixes root_cause>
            risk:        <which other metrics could regress and why>
          CHECK proposed_change does NOT affect saturated_metrics negatively
  check:  proposed_change is targeted (affects ≤ 3 spec sections)
  then:   step 5
  fail:   EMIT "Change too broad — narrowing scope" ; revise proposed_change ; retry step 4

step 5: APPLY_CHANGE
  think:  I am modifying the language itself. Every word I add has to pull its
          weight — ambiguity in the spec propagates into every skill written with it.
  do:     COPY CURRENT_SPEC → draft_spec
          APPLY proposed_change to draft_spec
          BUMP version: current_version → next_version (increment minor)
          ADD change to draft_spec changelog section
  check:  draft_spec is valid PRISM (all mandatory blocks present, syntax correct)
  then:   step 6
  fail:   REVERT draft_spec ; EMIT "Change produced invalid spec — reverting" ; goto step 4

step 6: SCORE_NEW_SPEC
  think:  I must re-run the FULL metric suite on the new spec, not just the
          metric I was optimising. A targeted fix can cause unexpected regressions.
  do:     FOR EACH metric in METRICS_SUITE:
            APPLY metric to draft_spec using SAME test_battery as step 2
            RECORD score(metric) → scores_after
          COMPUTE delta = scores_after - scores_before
          DETECT regressions = {m : delta(m) < -2}   # 2pt tolerance
  check:  No regressions detected (delta(m) ≥ -2 for all m)
  then:   step 7
  fail:   EMIT regression_report ; REVERT draft_spec ; goto step 4 with regression context

step 7: DECIDE_SHIP
  think:  Did the change actually improve things? A change that fixes one metric
          by 1 point at the cost of 5 points elsewhere is not an improvement.
          I should also check if we've hit convergence — if all metrics are
          above the convergence threshold, further iteration has diminishing returns.
  do:     COMPUTE overall_score_before = weighted_average(scores_before)
          COMPUTE overall_score_after  = weighted_average(scores_after)
          IF overall_score_after > overall_score_before THEN
            APPROVE draft_spec
          ELSE
            REJECT draft_spec
            EMIT "Change did not improve overall score — discarding"
            goto step 4 with rejection context
          CHECK convergence: IF all metrics ≥ convergence_threshold THEN
            EMIT "All metrics saturated — PRISM may be near-optimal for current test battery"
            SUGGEST expanding test battery diversity
  check:  overall_score_after ≥ overall_score_before
  then:   step 8

step 8: EMIT_OUTPUTS
  think:  The outputs need to be useful to a human reviewing this iteration
          AND to the next agent iteration reading VERSION_LOG. Both audiences
          need different levels of detail.
  do:     SAVE draft_spec → output_dir/PRISM_v{next_version}.md
          WRITE iteration entry to VERSION_LOG:
            - version, date, iteration_number
            - scores_before, scores_after, delta
            - lowest_metric, root_cause, proposed_change description
            - overall: IMPROVED | NEUTRAL | REGRESSED
          EMIT delta_report:
            - scores table (before vs after, all 7 metrics)
            - change applied (one paragraph)
            - what to try next (recommendations for next iteration)
  then:   DONE

[SHAPE]
emit  updated_spec    as file via present_files  (PRISM_v{next_version}.md)
emit  delta_report    as text inline
emit  version_log     as file via present_files  (VERSION_LOG.md)
on_fail: EMIT "Iteration failed at step <N>: <reason>" as text
         EMIT current scores_before if populated (so progress is not lost)

[GUARD]
never  ship a spec version with any metric regression > 2 points
never  apply more than one conceptual change per iteration
never  skip scoring step 6 — re-scoring is mandatory, not optional
always preserve VERSION_LOG across iterations — it is the memory of the agent
always base proposed changes on metric evidence, not intuition alone
before APPLY_CHANGE: confirm proposed_change has a stated root_cause
before EMIT_OUTPUTS: verify draft_spec passes all mandatory block checks
