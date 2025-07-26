# Lango MVP – Milestone 1 Roll‑out Development Plan  
*Version 1.1 – 22 July 2025*

---

## Table of Contents
1. [Purpose & Scope](#purpose--scope)  
2. [Roll‑out Sequence](#roll-out-sequence)  
3. [Stage‑by‑Stage Instructions](#stage-by-stage-instructions)  
   * A‑1 – Merge FR‑2 Code & Persist Counts  
   * A‑2 – Stabilise CI  
   * B‑1 – Introduce `TestFixtures.kt` DSL Enhancements  
   * B‑2 – Migrate Data‑layer Tests  
   * B‑3 – Refactor ViewModel & UI Tests  
   * B‑4 – Remove Obsolete Snapshots  
   * B‑5 – CI Guard – Fixture Enforcement  
4. [Appendix A – File/Directory Reference](#appendix-a--filedirectory-reference)  
5. [Appendix B – Acceptance‑Criteria Mapping](#appendix-b--acceptance-criteria-mapping)  
6. [Appendix C – Additional Details & References](#appendix-c--additional-details--references)

---

## Purpose & Scope

This document is the **single source of truth** to prepare for Milestone 1 (F‑1 → F‑5) with the two‑step strategy agreed in grooming:

1. **FR2 patch** – land FR‑2 (session initialization + prompt builder) with minimal test edits, including count reset and persistence.  
2. **Fixture refactor** – replace brittle, asset‑driven tests with builder‑style fixtures before Sprint 3 work begins.

Everything you need—file paths, Gradle tasks, Kotlin snippets, test IDs and exit criteria—is enumerated below.

---

## Roll‑out Sequence

| Seq   | Stage                          | Objective                                                                    | Sprint target | Primary owners |
|-------|--------------------------------|------------------------------------------------------------------------------|---------------|---------------|
| **A‑1** | Merge FR‑2 Code & Persist Counts | Implement `StartSessionUseCase.dequeueAndReset()`, `GenerateDialogueUseCase.buildInitialPrompt()`, increment `newTarget` counts, and persist via repository. | Sprint 2      | Backend       |
| **A‑2** | Stabilise CI                   | Amend failing FR‑1 tests; add new FR‑2 specs (B1‑UT, PS‑1); update count persistence tests. | Sprint 2      | QA            |
| **B‑1** | Introduce Fixtures DSL         | Extend `TestFixtures.kt` to support counter parameters                          | Sprint 2/3    | Backend + QA  |
| **B‑2** | Migrate Data Tests             | Rewrite FS‑1…FS‑6 (including persistence) to use fixtures                     | Sprint 3      | QA            |
| **B‑3** | Refactor UI Tests              | Replace literal‑string assertions with state assertions; stub persistence    | Sprint 3      | Mobile        |
| **B‑4** | Remove Snapshots               | Delete obsolete scripted dialogue fixtures                                   | Sprint 3      | All           |
| **B‑5** | CI Guard – Fixture Enforcement | Lint rule: forbid direct asset reads in tests                                | Sprint 3      | DevEx         |

---

## Stage‑by‑Stage Instructions

### A‑1 · Merge **FR‑2** Code & Persist Counts  
**Goal:** Ship dequeue/reset, prompt-builder logic, count reset, and persistence behind existing tests.

1. **Branch:**  
   ```bash
   git checkout -b feature/fr2-integration
   ```
2. **Domain implementation**  
   *File:* `domain/src/main/kotlin/com/example/domain/StartSessionUseCase.kt`  
   ```kotlin
   fun startSession(): Result<Queues> = runCatching {
       val queues = repository.loadQueues().getOrThrow()
       val target = queues.newQueue.removeFirst().apply {
           presentationCount = 0
           usageCount = 0
       }
       repository.saveQueues(queues).getOrThrow() // persist reset counts
       queues        // return mutated reference
   }
   ```
3. **Prompt builder orchestration**  
   *File:* `domain/src/main/kotlin/com/example/domain/GenerateDialogueUseCase.kt`  
   ```kotlin
   fun buildInitialPrompt(q: Queues): String { /* load instructions from PROMPT.md */ }
   ```
4. **Count reset & persistence**  
   - After `startSession`, verify `newTarget.presentationCount == 0` and `usageCount == 0`.  
   - Call `LearningRepository.saveQueues(queues)` to persist bootstrap state.
5. **Unit specs**  
   - `StartSessionUseCaseTest` covers **B1‑UT**, including persistence via mock repository.  
   - `GenerateDialogueUseCaseTest` covers **PS‑1** snapshot (strip `sessionId` via regex).
6. **Gradle**  
   ```bash
   ./gradlew :domain:testDebugUnitTest
   ```
7. **Exit criteria**  
   - Both new tests green.  
   - No other FR‑1 tests fail after minimal assertion patches.  
   - Verify `saveQueues` invocation (mock verification).

### A‑2 · Stabilise CI  
**Goal:** Ensure main branch passes with FR‑2 logic and count persistence.

- **Relax brittle assertions**  
  - `BootstrapSmokeTest`: adjust `assertEquals` → `assertTrue(… >= …)`.  
  - Update tests to stub and verify repository persistence.
- **Add new CI test**  
  - FS‑6: verify persistence of reset counts in repository tests.

### B‑1 · Introduce `TestFixtures.kt` DSL Enhancements  
**Goal:** Provide reusable DSL decoupled from raw JSON with counter support.

1. **Create file**  
   `shared-test/src/main/kotlin/com/example/testing/TestFixtures.kt`  
   ```kotlin
   object TestFixtures {
       fun dummyItem(
           id: String = "ID_001",
           token: String = "token",
           presentation: Int = 0,
           usage: Int = 0,
           learned: Boolean = false
       ) = LearningItem(id, token, presentation, usage, learned)

       fun queuesFixture(
           new: List<LearningItem> = listOf(dummyItem()),
           learned: List<LearningItem> = emptyList()
       ) = Queues(new.toMutableList(), learned.toMutableList())
   }
   ```
2. **Shared gradle module**  
   Add `implementation project(":domain")` to `shared-test/build.gradle`.
3. **Exit criteria**  
   - Fixture compiles; supports presentation/usage counts.

### B‑2 · Migrate Data‑layer Tests  
**Goal:** Rewrite FS‑1…FS‑6 using fixtures and remove JSON assets.

1. **Touch each test**  
   - Replace JSON file loads with `queuesFixture()` & `dummyItem()`.  
   - Parameterize counts via constants.  
2. **Example FS‑6**  
   ```kotlin
   coEvery { repository.saveQueues(any()) } returns Result.success(Unit)
   // assert that saveQueues called with reset counts.
   ```
3. **Exit criteria**  
   - All FS‑x tests green; no JSON references.

### B‑3 · Refactor ViewModel & UI Tests  
**Goal:** Remove literal strings, stub persistence.

1. **MainViewModelTest**  
   - Stub `StartSessionUseCase` to include persistence; verify `saveQueues` call.  
2. **SessionIntegrationTest**  
   - Use fixtures and stub `saveQueues`.  
3. **Exit criteria**  
   - UI tests green; persistence stubbed.

### B‑4 · Remove Obsolete Snapshots  
1. Delete `src/test/resources/long_dialogue_*` files.  
2. Remove Gradle `testResources` linkage.  
**Exit criteria:** Only `initial_prompt.txt` remains.

### B‑5 · CI Guard – Fixture Enforcement  
1. **ktlint rule**: flag `File("assets/…")` in `src/test`.  
2. **GitHub Action**: fail on direct asset reads.  
**Exit criteria:** PRs with direct reads blocked.

---

## Appendix A – File/Directory Reference

| Path | Description |
|------|-------------|
| `domain/src/main/.../StartSessionUseCase.kt` | Dequeue/reset + persistence |
| `domain/src/main/.../GenerateDialogueUseCase.kt` | Prompt builder orchestration |
| `shared-test/src/.../TestFixtures.kt` | DSL with counters |
| `domain/src/test/.../StartSessionUseCaseTest.kt` | B1‑UT & persistence |
| `domain/src/test/resources/fixtures/initial_prompt.txt` | PS‑1 snapshot |

## Appendix B – Acceptance‑Criteria Mapping

| Spec ID | Implemented in | Verification |
|---------|----------------|--------------|
| **B1‑UT** | `StartSessionUseCaseTest` | Stage A‑1 |
| **PS‑1** | `GenerateDialogueUseCaseTest` | Stage A‑1 |
| **FS‑1…FS‑5** | Data tests migration | Stage B‑2 |
| **FS‑6** | Persistence tests | Stage A‑2/B‑2 |
| **VM‑1** | `MainViewModelTest` | Stage B‑3 |
| **UI‑1** | Espresso test | Stage B‑3 |

## Appendix C – Additional Details & References

- **Count Increment & Persistence**  
  - Requirement from MS1_FR2: after introduction, increment `presentationCount` and persist fileciteturn0file1L11-L15.  
- **PROMPT.md Orchestration**  
  - Ensure orchestration sections loaded correctly as per MS1_FR2 scope fileciteturn0file1L5-L9.  
- **DI Testability**  
  - Confirm `SaveQueues` can be injected and stubbed via Hilt modules.

---

# Summary of Changes

| Section            | Change Description                                              |
|--------------------|-----------------------------------------------------------------|
| A‑1                | Added count reset & `saveQueues` for persistence                |
| A‑1 step 3         | Stub orchestration load from `PROMPT.md`                        |
| A‑1 Exit criteria  | Verify `saveQueues` invocation                                  |
| A‑2                | Added FS‑6 persistence test                                     |
| B‑1                | Extended DSL to include counter parameters                      |
| B‑2                | Added migration step for FS‑6                                   |
| Appendix B         | Added FS‑6 mapping                                              |
| Appendix C         | New section linking MS1_FR2 requirements                        |
