<!--
  MIGRATION.md — the migration playbook produced as a byproduct of the process
  (CONV-03, PRD §MIGRATION.md). Final artifact for ALL scenarios.
  Branch: korza/benchmark/2a/eval-ref
-->

# Migration Playbook — 2a / eval-ref

Subject: spring-petclinic-migration (Java 11 → 17 → 21)
Scenario: **2a — Claude Code, NO OpenRewrite** (the AI performs the whole migration by hand).
Evaluator: eval-ref (reference shakedown — metrics throwaway per D-01; code edits real per D-02).

## Tools and recipes used

- **Claude Code (Anthropic Opus 4.8)** — performed the entire migration by hand.
- **No OpenRewrite** — scenario 2a deliberately uses zero recipes; every edit is manual.
- **build-adapter.sh** — `compile|test` per hop (Gradle 6.5.1 daemon on JDK 11, forked
  `--release {17,21}` javac/test JVM; the subject build files are edited by the
  migration but the adapter never adds suppression flags).
- **gate-runner.sh** — five blocking gates per checkpoint; pinned `SEMGREP_BIN=.venv/bin/semgrep` (1.86.0).

## Exact OpenRewrite recipe sequence

**None — scenario 2a runs no OpenRewrite recipes.** The migration below was done by hand.

### Java 11 → 17 (the heavy hop — Boot major + jakarta)
1. `build.gradle`: Spring Boot BOM `2.1.18.RELEASE` → `3.3.5` (MIGRATION-POLICY §1: 2.1→2.7→3.x).
2. `javax.* → jakarta.*` across 14 source files: `persistence`, `validation`,
   `validation.constraints`, `inject`, `xml.bind.annotation`. **`javax.cache`
   (JSR-107) left unchanged** — it is NOT part of the Jakarta rename.
3. Boot-3 dependency split-outs / renames:
   - add `spring-boot-starter-validation` (validation left the web starter in Boot 2.3+);
   - `jakarta.xml.bind:jakarta.xml.bind-api` (BOM-managed 4.x) + runtime `org.glassfish.jaxb:jaxb-runtime`;
   - `jakarta.inject:jakarta.inject-api:2.0.1` (jakarta namespace);
   - `mysql:mysql-connector-java:5.1.42` → `com.mysql:mysql-connector-j`; `ehcache 3.2.2` → `3.10.8`;
   - drop the forced `thymeleaf-spring4` (Boot 3 brings thymeleaf-spring6).
4. `src/main/resources/application.properties`: `spring.datasource.schema|data` →
   `spring.sql.init.schema-locations|data-locations` (the old keys were removed in
   Boot 2.5+ and were silently dropping schema/data → "object not found" at test time).
5. Tests: `test { useJUnitPlatform() }` + `testRuntimeOnly junit-vintage-engine` to run the
   existing JUnit 4 suite on Boot 3 (no @Test deleted/commented); Mockito runner import
   `org.mockito.runners.MockitoJUnitRunner` → `org.mockito.junit.MockitoJUnitRunner` (Mockito 5).
6. `sourceCompatibility` 11 → 17.
Result: clean `--release 17`, zero `--add-opens`/`--add-exports`, **42/42 tests pass**.

### Java 17 → 21 (language/runtime only — NO Boot major)
- **No source changes required.** The codebase uses none of the Java 17→21
  removed/deprecated-for-removal APIs: no `Thread.stop/suspend/resume`, no
  `finalize()` overrides, no `SecurityManager`, no `SSLSession.getPeerCertificateChain`,
  no `sun.*`/`com.sun.*`. Verified by whole-tree grep.
- Confirmed a clean `--release 21` build with **42/42 tests passing**.

## Checkpoint gate results

| Gate | Java 17 | Java 21 |
|------|---------|---------|
| 1 compile | PASS (clean --release 17, 0 --add-opens) | PASS (clean --release 21, 0 --add-opens) |
| 2 tests | **FAIL — framework-defect** (checkpoint tests 42/42 PASS; the java11 ground-truth re-run runs against the migrated tree, see RUN-REPORT BUG-03) | **FAIL — framework-defect** (same BUG-03; 42/42 checkpoint tests PASS; java17 ref present so no missing-diff-base failure) |
| 3 security | PASS (no net-new Critical/High vs Java-11 baseline) | PASS |
| 4 api hygiene | PASS (no sun.*/com.sun.*, no removal warnings) | PASS |
| 5 completeness | PASS (checklist attested; no `// TODO: migrate`) | PASS |
| overall | fail (justified — sole failure is framework-defect; migration correct, D-09 = success) | fail (justified — same) |

Wall clock per hop is recorded in each leaf's `metrics.csv` (`wall_clock_seconds`).
NOTE: metrics are throwaway for this reference shakedown (D-01); the code is real (D-02).

## Lessons learned

- **The Boot major upgrade is the whole java17 hop.** Petclinic itself is small; the
  difficulty is entirely Boot 2.1→3.x: validation/jaxb/inject split-outs, the
  `spring.sql.init.*` property rename (silent data-loss → confusing "table not found"),
  and the JUnit-4-on-Boot-3 setup (`useJUnitPlatform()` + vintage engine).
- **`javax.cache` is a trap** — JSR-107 keeps the `javax.cache` namespace; do NOT
  rename it to `jakarta.cache`.
- **java21 is a no-op for this codebase** — no removed/deprecated APIs are used, so the
  17→21 hop is pure validation. Expect other subjects to need Cleaner/Thread work.
- **A green migration is achievable by hand for 2a** — resolves MIGRATION-POLICY §5's
  open risk (2.1→3.x flag-free build viability) positively for petclinic.
- **Framework caveat (not a migration issue):** Gate 2's Java-11 ground-truth re-run is
  currently run against the migrated tree, so it fails at every migrated checkpoint
  (RUN-REPORT BUG-03/DEFER-FW-A); and `--gate N` re-runs must be replaced by full runs
  until DEFER-FW-B lands. Neither affects the correctness of the migration.

## Repeating the process for Java 25 (next LTS hop)

1. Provision a fresh benchmark branch off the java21 checkpoint (`korza/benchmark/java21/2a`).
2. Start at the language/runtime audit (java25 is expected to be language/runtime-level,
   like 21): whole-tree grep for that hop's removed/deprecated-for-removal APIs.
3. `build-adapter compile|test --hop java25` (add `temurin_25` to versions.yml + a
   `gates/java25.yml` seeded from the OpenRewrite UpgradeToJava25 banned-API table).
4. Replace any flagged APIs by hand (2a) or via `UpgradeToJava25` (OpenRewrite scenarios);
   reach a clean `--release 25`, zero `--add-opens`.
5. Run the five gates, attest the completeness checklist, create + push
   `korza/benchmark/java25/2a`, emit billing + metrics, and append the verdict here.
