
# Test-Driven Build Plan for Lango MVP Android App — Core Functionality

## Instructions for Following and Updating This Plan
**Documenting Status Updates:**  
As you progress through development, update the status of each task directly within this `development_plan_core.md` file. The status indicators are:
  
* | ✗ | Indicating the task has not been completed
* | ✓ | Indicating the task has been successfully completed and verified.

**File Editing Instructions for Status Updates:** For updating the status of individual tasks (e.g., `| ✗ |` to `| ✓ |`), use the `replace` tool. This ensures precise, in-place modifications without overwriting the entire file.  
**Content Modifications:** Adding new tasks, modifying task descriptions, or making other structural changes to the plan, use the `replace` tool to make edits line by line. If a whole document refactor is needed, first propose the content refactor and seek user approval before using the `write_file` tool to overwrite the document. `write_file` is only to be used for large rewrites and must be used carefully to ensure that there is no unintended loss of file contents.  
**Synchronization:** Before moving on to the next major task (as defined within this document), I will read the current file content to ensure my internal state is synchronized with the document on disk. If there is a descrepancy beyond the task status update instructions already provided, I will notify the user and recommend a course of action to resolve the descrepancy.

## Phase & Step Checklist (✓ = complete, ✗ = not yet)

| Phase | Step | Status |
|-------|------|--------|
| **0 Foundation** | Data/Domain classes implemented (`LearningItem`, `Queues`, Use-Case stubs, Repository stubs) | ✓ |
| | `LearningItem` unit tests written & passing | ✓ |
| | Other entity/model unit tests (`Queues`, param helpers) | ✓ |
| | Repository unit tests | ✓ |
| | Use-case unit tests (StartSession, ProcessTurn, GenerateDialogue, EndSession) | ✗ |
| | ViewModel unit tests (state transitions, error handling) | ✓ |
| **1 Core Pipeline** | Stage 0 — Project bootstrap (modules, deps, sample JSON, CI) | ✗ |
| | Stage 1 — Unit Test 1 (File Read) | ✗ |
| | Stage 2 — Unit Test 2 (API Send) | ✗ |
| | Stage 3 — Unit Test 3 (API Receive) | ✗ |
| | Stage 4 — Unit Test 4 (Audio Play) | ✗ |
| | Stage 5 — End-to-End Core Sequence Integration test | ✗ |
| **2 Session Loop & Mastery** | Implement session turn loop, mastery promotion, bias logic | ✗ |
| | Extend unit tests & new integration test suite | ✗ |
| **3 UX Hardening** | Minimal UI polish, interruption handling, error localisation | ✗ |
| **4 Readiness** | Full regression, 80 %+ coverage, release build config | ✗ |

> **Progress Note (July 16 2025):** All Phase 0 classes compile; `LearningItemTest` is green against the canonical test data located in `data/src/test/resources`. The remaining Phase 0 unit tests are pending and must pass before advancing to Stage 1.

---

### 1 Purpose (unchanged)
This document details the **Test-Driven Development (TDD) build plan** targeting the *Core Functionality* milestone. Updates in v1.1 align the plan with the canonical **test resources** found under `data/src/test/resources` and introduce an explicit **Phase 0 verification gate**: every class written for Phase 0 must have at least one meaningful unit test **green** before Stage 1 work begins.

### 2 Key Additions in v1.1
* **Test-Resource Alignment** — All unit tests must read fixtures directly from `data/src/test/resources` using Kotlin class-loader helpers (e.g. `val url = requireNotNull(javaClass.classLoader.getResource("core_blocks.json"))`). Remove any hard-coded `assets/` paths inside tests.
* **Phase 0 Class-Feature Tests** — Create new test classes:
  * `QueuesTest` — FIFO dequeue behaviour, immutability guarantees
  * `LearningRepositoryImplTest` — in-memory load/save using fixture files; error branch on malformed JSON
  * `GenerateDialogueUseCaseTest` — prompt construction sanity (mock LLM)
  * `StartSessionUseCaseTest`, `ProcessTurnUseCaseTest`, `EndSessionUseCaseTest` — happy-path logic using stubbed repository & services
  * `MainViewModelTest` — StateFlow emissions for resume, speak, and end paths (using Turbine)
* **Coverage Gate** — CI fails if Phase 0 coverage < 60 % on :domain + :data; target 80 % by Phase 2.

### 3 Revised Build Stages
The original Stage numbers are preserved, but **Stage 0 now explicitly requires Phase 0 tests to be green** before its PR may merge.

#### Stage 0 — Project Bootstrap (unchanged plus new exit criteria)
* **Exit criteria update:** ✓ All checklist items under _Phase 0 Foundation_ are green in CI.

#### Stage 1 — Unit Test 1 (File Read)
* **No behavioural change**, but tests must pull `core_blocks.json` / `learned_queue.json` from `data/src/test/resources`.
* Add a negative test case using `broken_core_blocks.json` (place an intentionally malformed file in the same resources dir).

#### Stage 2 — Unit Test 2 (API Send)
* Mock OpenAI Retrofit service; validate prompt contains **_exactly_** the new target + learned pool tokens present in fixtures.

#### Stage 3 — Unit Test 3 (API Receive)
* Supply fixture `sample_llm_response.json` from resources; assert extraction logic.

#### Stage 4 — Unit Test 4 (Audio Play)
* Provide `sample_tts_audio.mp3` fixture; use `ShadowMediaPlayer` (Robolectric) to simulate playback.

#### Stage 5 — End-to-End Core Sequence Integration Test
* Path-agnostic: use class-loader to locate fixtures; do _not_ rely on Android assets.

### 4 CI / Branch Conventions (minor tweak)
* New **`phase0/`** prefix for branches that only add tests (e.g. `phase0/queues-tests`).
* PR label `phase0-green` triggers coverage gate described above.

### 5 Dependencies Reminder (no change)
When adding libraries for tests (AssertJ, Turbine, Coroutine Test, etc.) follow the **Dependency Instructions** guide (`DEPENDENCY_INSTRUCTIONS.txt`).

---

> _End of v1.1 update — the remainder of the document (stages 1-4 details, integration test setup, and post-core roadmap) is unchanged from v1.0 and is omitted for brevity._
