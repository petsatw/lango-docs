# DP\_F: State Machine Refactor (Revised)

**Last Updated:** July 25, 2025

---

## Overview

This document supersedes the original DP\_F plan. It integrates the architectural decisions, context, and code alignment from the existing codebase (MY\_CB) and our in-depth design session, organizing the work into a six-step development sequence.

### Business Objectives & Success Criteria

- **Learning Experience:** Beginner → mastery via spoken, natural dialogue.
- **Conversational Architecture:** Reflect dialogue flow in code while tracking progress.
- **Compile/Test Metrics:** 90% first-compile success; 80% first-green-test on every PR.

## Scope

### In Scope

- FSM-driven session orchestration: YAML spec + interpreter.
- Configurable thresholds (usage targets, loop caps, failure thresholds).
- Queue and `LearningItem` management; LLM prompt helpers.
- Persistence of mastered items via `LearnedPoolRepository`.

### Out of Scope

- Legacy orchestrator and use-case classes: `StartSessionUseCase`, `GenerateDialogueUseCase`, `ProcessTurnUseCase`, `EndSessionUseCase`, `CoachOrchestrator`, and their tests.

## Architecture Alignment

Leverage existing modules in MY\_CB:

- **domain/**: Holds core business logic, DI interfaces, and FSM runner.
- **app/**: Android-specific code, Hilt modules, `MainViewModel`, UI state mapping.
- **RepositoryModule**: Provides `LearningRepository` / `LearnedPoolRepository`.
- **SpeechModule**: TTS/ASR abstractions (unused in stubs).

Additions should follow current package conventions under `com.example.domain` and `com.example.lango_coach_android`.

## Development Sequence

Each step is a self-contained PR, designed to compile and pass its tests in isolation.

### Step 1: Define & Expose Core Settings

**Goal:** Centralize all thresholds in an injectable config.

1. **Create **`` in domain:

   ```text
   domain/src/main/kotlin/com/example/domain/config/SessionConfig.kt
   ```

   ```kotlin
   package com.example.domain.config

   data class SessionConfig(
     val requiredUsages: Int,
     val maxRepeatLoops: Int,
     val repeatModeThreshold: Int
   )
   ```

2. **Add **`` in app:

   ```text
   app/src/main/java/com/example/lango_coach_android/di/ConfigModule.kt
   ```

   ```kotlin
   @Module
   @InstallIn(SingletonComponent::class)
   object ConfigModule {
     @Provides fun provideSessionConfig(): SessionConfig =
       SessionConfig(
         requiredUsages       = BuildConfig.REQUIRED_USAGES,
         maxRepeatLoops       = BuildConfig.MAX_REPEAT_LOOPS,
         repeatModeThreshold  = BuildConfig.REPEAT_THRESHOLD
       )
   }
   ```

3. **Define BuildConfig fields** in `app/build.gradle.kts`:

   ```kotlin
   android {
     defaultConfig {
       buildConfigField("int", "REQUIRED_USAGES", "3")
       buildConfigField("int", "MAX_REPEAT_LOOPS", "3")
       buildConfigField("int", "REPEAT_THRESHOLD",  "2")
     }
   }
   ```

> All code must reference `SessionConfig`, never literals.

---

### Step 2: Paste & Parse `session.yaml` + Minimal Runner

**Goal:** Prove FSM spec, parser, and runner wiring in pure JVM.

1. **Copy **`` to domain resources:

   ```text
   domain/src/main/resources/session.yaml
   ```

2. **Add Jackson-YAML** dependencies in `domain/build.gradle.kts`:

   ```kotlin
   implementation("com.fasterxml.jackson.dataformat:jackson-dataformat-yaml:2.15.2")
   implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.15.2")
   ```

3. **Define spec classes** in `com.example.domain.fsm`:

   ```text
   domain/src/main/kotlin/com/example/domain/fsm/
     - FsmSpec.kt
     - FsmLoader.kt
     - Transition.kt
     - Action.kt
   ```

4. **Implement **``** stub**:

   ```text
   domain/src/main/kotlin/com/example/domain/fsm/StateMachineRunner.kt
   ```

   ```kotlin
   class StateMachineRunner(
     private val spec: FsmSpec,
     private val config: SessionConfig
   ) {
     private var state = spec.initialState
     private val ctx = mutableMapOf<String, Any>()

     fun on(event: String): String {
       val t = spec.transition(state, event)
         ?: throw IllegalStateException("Unhandled event $event in $state")
       // Update ctx (usageCount, presentationCount) using config
       state = t.nextState
       return state
     }
   }
   ```

5. **Golden-path JUnit** in `domain/src/test/...`:

   - Inject `SessionConfig(requiredUsages=3, …)`.
   - Drive `listOf("start", "tts_done", repeat("asr_correct", 3))`.
   - Assert final state == `SESSION_DONE`.

> If this passes, parsing and runner basics are validated.

---

### Step 3: Wire Into ViewModel Stubs

**Goal:** Replace legacy orchestrator with FSM runner in `MainViewModel`.

1. **Modify** `MainViewModel` constructor:

   ```text
   app/src/main/java/com/example/lango_coach_android/MainViewModel.kt
   ```

   ```diff
   - class MainViewModel @Inject constructor(
   -   private val coachOrchestrator: CoachOrchestrator
   + class MainViewModel @Inject constructor(
   +   private val runner: StateMachineRunner,
   +   private val config: SessionConfig
   ) : ViewModel() {
   ```

2. **Replace calls**:

   ```diff
   - coachOrchestrator.startSession()
   + transition("start")
   ```

   ```diff
   - coachOrchestrator.processTurn(asrResult)
   + transition(if (isCorrect) "asr_correct" else "asr_incorrect")
   ```

3. **Map state → **`` using existing sealed classes:

   - `INTRO_WORD` → `CoachSpeaking(text)`
   - `WAIT_STUDENT` → `Waiting`
   - `SESSION_DONE` → `Congrats`

> No actual TTS/ASR calls—UI stubs only.

---

### Step 4: Implement Persistence for Mastered Items

**Goal:** Persist on mastery per crash-resume rules.

1. **Reuse** `LearnedPoolRepository` from `RepositoryModule`:

   ```kotlin
   interface LearnedPoolRepository {
     fun saveQueues(headerJson: HeaderJson)
     fun loadQueues(): HeaderJson
   }
   ```

2. **In **``, on `ADD_TO_LEARNED_POOL` action:

   ```kotlin
   if (action == "add_word_to_learned_pool") {
     repository.saveQueues(ctx.headerJson)
   }
   ```

3. **On app start**, `MainViewModel.init()` should call:

   ```kotlin
   val header = repository.loadQueues()
   runner.initialize(header)
   ```

> Mastered items survive crashes; in-progress items reset.

---

### Step 5: Delete Old Orchestrator & Tests

**Goal:** Remove legacy code in one atomic PR after Step 4 green.

1. **Delete** these files under `domain/src/main/kotlin/com/example/domain/`:

   - `StartSessionUseCase.kt`
   - `GenerateDialogueUseCase.kt`
   - `ProcessTurnUseCase.kt`
   - `EndSessionUseCase.kt`
   - `CoachOrchestrator.kt`

2. **Delete** corresponding test classes under `domain/src/test/...` and `app/src/test/...`.

3. **Ensure** CI still compiles and Step 2’s golden-path test remains green.

---

### Step 6: Expand Test Coverage to “Sad Paths”

**Goal:** Validate exception paths and config-driven logic.

1. **Parameterized tests** for unhandled events:

   ```kotlin
   @ParameterizedTest
   @ValueSource(strings = ["tts_done", "asr_error"])
   fun `unhandled events throw`(event: String) {
     assertThrows<IllegalStateException> { runner.on(event) }
   }
   ```

2. **Config variability tests**:

   - Inject `SessionConfig(requiredUsages=5, …)`.
   - Drive 5 correct hits → assert mastery only after the 5th.

3. **CI Enforcement**:

   - Block merges if any of these tests fail.

---

## Roll-out, CI & Metrics

- **CI Setup:** Each PR must pass:
  1. Gradle compile (`./gradlew assemble`)
  2. Unit tests (`./gradlew test`)
- **Metrics:** Instrument CI to capture:
  - First-compile success rate
  - First-green-test rate
- **Monitoring:** Maintain a dashboard; alert if metrics fall below targets.

---

*End of Revised DP\_F*

