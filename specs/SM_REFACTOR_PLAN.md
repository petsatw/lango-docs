# SM_REFACTOR: State Machine Refactor (Summary)

**Last Updated:** July 25, 2025  
This is the concise, actionable six‑step plan. For full technical details and deep rationale, see **SM_REFACTOR_SPEC.md**.

---

## Overview  
This summary is the development plan for the State Machine Refactor. It structures our work into six self‑contained PRs, aligned to Lango codebase conventions, and references the detailed spec (SM_REFACTOR_SPEC) for all low‑level requirements.

### Business Objectives & Success Criteria  
- **Learning Experience:** Beginner → mastery via spoken, natural dialogue.  
- **Conversational Architecture:** Reflect dialogue flow in code while tracking progress.  
- **Compile/Test Metrics:** 90% first‑compile success; 80% first‑green‑test on every build point (PR).  
  > See **SM_REFACTOR_SPEC §6** “Roll‑out & CI Metrics”

## Scope  

### In Scope  
- FSM orchestration via YAML spec + interpreter.  
- Configurable thresholds (`requiredUsages`, loop caps, failure thresholds).  
- Queue & `LearningItem` lifecycle management; LLM prompt helpers.  
- Persistence of mastered items via `LearnedPoolRepository`.  
  > See **SM_REFACTOR_SPEC §1** “Must‑Be‑True Invariants & Core Domain Model”  

### Out of Scope  
- Legacy orchestrator/use‑case classes (`StartSessionUseCase`, `GenerateDialogueUseCase`, `ProcessTurnUseCase`, `EndSessionUseCase`, `CoachOrchestrator`) and their tests.  

---

## Architecture Alignment  
Use existing MY_CB modules and DI patterns:  
- **domain/**: core logic, `SessionConfig`, FSM runner.  
- **app/**: Android code, Hilt modules, `MainViewModel`.  
- **RepositoryModule** & **SpeechModule**.  
  > See **SM_REFACTOR_SPEC §5** “Persistence Integration” and §3 “ViewModel & UI Integration”

Additions follow package conventions under `com.example.domain` and `com.example.lango_coach_android`.

---

## Development Sequence  
Each step is a standalone PR.  After each, CI must compile and pass tests before moving on.

### Step 1: Define & Expose Core Settings  
**Goal:** Centralize all thresholds in a single config.  
1. **Create** `SessionConfig` in domain (`domain/src/main/kotlin/com/example/domain/config/SessionConfig.kt`)  
2. **Provide** via Hilt `ConfigModule` (`app/src/main/java/com/example/lango_coach_android/di/ConfigModule.kt`)  
3. **Define** BuildConfig fields in `app/build.gradle.kts`  

> All code must reference `SessionConfig`—no literals.  
> **See SM_REFACTOR_SPEC §1** “Must‑Be‑True Invariants & Core Domain Model”

---

### Step 2: Paste & Parse `session.yaml` + Minimal Runner  
**Goal:** Validate FSM spec, parser, and runner in pure‑JVM.  
1. Copy `session.yaml` to `domain/src/main/resources/`  
2. Add Jackson‑YAML deps in `domain/build.gradle.kts`  
3. Define spec data‑classes (`FsmSpec`, `StateDefinition`, `ActionDefinition`)  
4. Stub out `StateMachineRunner` to load spec, apply `config`, and invoke guards/actions  
5. Golden‑path JUnit: drive `start`,`tts_done`,`asr_correct×3` → expect `SESSION_DONE`.  

> **See SM_REFACTOR_SPEC §2.1** “FSM Spec → Data Classes”  
> **See SM_REFACTOR_SPEC §2.2** “Must‑Implement Actions”  
> **See SM_REFACTOR_SPEC §2.3** “Guard Logic & Unhandled Events”

---

### Step 3: Wire Into ViewModel Stubs  
**Goal:** Replace legacy orchestrator in `MainViewModel` without real TTS/ASR.  
1. Modify constructor to inject `StateMachineRunner` & `SessionConfig`  
2. Replace `coachOrchestrator.startSession()`/`processTurn()` calls with `runner.on(event)`  
3. Map returned FSM states → existing `UiState` sealed classes  

> **See SM_REFACTOR_SPEC §3.1** “FSM State → UiState Mapping”  
> **See SM_REFACTOR_SPEC §3.2** “ViewModel Stub Example”

---

### Step 4: Implement Persistence for Mastered Items  
**Goal:** Persist only at mastery, per crash‑resume rules.  
1. Reuse `LearnedPoolRepository` from `RepositoryModule`  
2. In runner’s `add_word_to_learned_pool` action, call `repository.saveQueues(headerJson)`  
3. On app init, load via `repository.loadQueues()` and `runner.initialize(header)`  

> **See SM_REFACTOR_SPEC §1** for `schemaVersion` and JSON shape  
> **See SM_REFACTOR_SPEC §5** “Persistence Integration”

---

### Step 5: Delete Old Orchestrator & Tests  
**Goal:** Remove legacy code in one atomic PR after Step 4 green.  
1. Delete `StartSessionUseCase`, `GenerateDialogueUseCase`, `ProcessTurnUseCase`, `EndSessionUseCase`, `CoachOrchestrator` under `domain/src/main/kotlin/...`  
2. Delete corresponding tests in `domain/src/test/...` and `app/src/test/...`  
3. Confirm CI still compiles and Step 2’s golden‑path test remains green.  

> Ensures no hidden dependencies remain.

---

### Step 6: Expand Test Coverage to “Sad Paths”  
**Goal:** Verify exception paths and config‑driven guards.  
1. Parameterized tests asserting `runner.on(badEvent)` throws `IllegalStateException`  
   > **See SM_REFACTOR_SPEC §2.3**  
2. Tests with alternative `SessionConfig` values (e.g. `requiredUsages=5`) to assert correct thresholds  
   > **See SM_REFACTOR_SPEC §2.2**  
3. Block merges on CI if any tests fail.  

---

## Roll‑out, CI & Metrics  
- **CI Checklist per PR:**  
  - Domain compiles: `./gradlew :domain:compileKotlin`  
  - Domain tests pass: `./gradlew :domain:test`  
  - App compiles: `./gradlew :app:compileKotlin`  
  - UI stubs render without error  
- **Metrics:** Track first‑compile and first‑green‑test rates  
  > See **SM_REFACTOR_SPEC §6** “Roll‑out & CI Metrics”

---

*End of DP_F_Summary.md*

