<!--
  MIGRATION.md — the migration playbook produced as a byproduct of the process
  (CONV-03, PRD §MIGRATION.md). Final artifact for ALL scenarios.

  In scenario 2f the GSD harness generates this automatically; in every other
  scenario the evaluator maintains it incrementally. Seeded EMPTY (D-09: zero
  migration answers) — fill each section as the migration is performed.

  Branch: korza/benchmark/2a/eval-ref
-->

# Migration Playbook — 2a / eval-ref

Subject: spring-petclinic-migration (Java 11 → 17 → 21)

## Tools and recipes used

<!-- Which tools and which OpenRewrite / framework recipes were applied. -->

## Exact OpenRewrite recipe sequence

<!--
  The precise, ordered recipe sequence actually run for each hop
  (Java 11 → 17, then Java 17 → 21). Record IDs and order as executed.
-->

### Java 11 → 17

### Java 17 → 21

## Checkpoint gate results

<!--
  Per-hop result of each of the five blocking gates (compile, tests, security,
  hygiene, completeness): PASS / FAIL and the wall-clock at the gate.
-->

## Lessons learned

<!-- Lessons specific to THIS codebase — what was surprising, what to watch for. -->

## Repeating the process for Java 25

<!--
  Step-by-step instructions to repeat this migration for the next LTS hop
  (Java 21 → 25), so the playbook is reusable.
-->
