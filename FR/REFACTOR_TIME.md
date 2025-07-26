# REFACTOR STRATEGY

Below is a **surgical, step-by-step refactoring plan** to rip out the old session logic/tests and replace it with a **YAML-first DSL** runtime FSM. Each step names the exact files (and line references) you’ll touch, so you can move fast without guesswork.

---

## A Remove the old turn-based code and tests

1. **Domain → delete old use-cases & orchestrator**
   Remove (or move to an `archive/` folder) these files under `domain/src/main/kotlin`:

   * `StartSessionUseCase.kt`
   * `GenerateDialogueUseCase.kt`
   * `ProcessTurnUseCase.kt`
   * `EndSessionUseCase.kt`
   * `CoachOrchestrator.kt`

2. **Domain tests → delete corresponding test classes**
   Under `domain/src/test/kotlin/com/example/domain/`, delete:

   * `StartSessionUseCaseTest.kt`&#x20;
   * `GenerateDialogueUseCaseTest.kt`&#x20;
   * `ProcessTurnUseCaseTest.kt`
   * `EndSessionUseCaseTest.kt`&#x20;

3. **App module → remove ViewModel orchestration**
   Edit `app/src/main/java/com/example/lango_coach_android/MainViewModel.kt` to strip out `startSession()`/`processTurn()` bodies (lines 59–118) , replacing them with a simple FSM driver call (added later).

4. **App tests → delete old integration & VM tests**
   Under `app/src/test/java/com/example/lango_mvp_android/`, delete:

   * `SessionIntegrationTest.kt`&#x20;
   * Any `EndToEndTest.kt` under `androidTest/…`
     Under `app/src/test/java/com/example/lango_coach_android/`, delete:
   * `MainViewModelTest.kt`&#x20;

---

## B Add the YAML spec to your codebase

1. **Copy the v0.2 spec** from the canvas into your repo at

   ```
   domain/src/main/resources/session.yaml
   ```

   (Just paste the full YAML block from the “Coaching Session State-Machine” doc.)

---

## C Runtime DSL interpreter

1. **Add YAML parsing dependency**
   In your `:domain` `build.gradle.kts`, add:

   ```kotlin
   implementation("com.fasterxml.jackson.dataformat:jackson-dataformat-yaml:2.15.2")
   implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.15.2")
   ```

2. **Define spec data classes**
   Create `domain/src/main/kotlin/com/example/domain/fsm/FsmSpec.kt`:

   ```kotlin
   package com.example.domain.fsm

   data class FsmSpec(
     val context: ContextSpec,
     val counters: List<String>,
     val states: Map<String, StateSpec>
   )
   data class ContextSpec(val required_usages: Int, /*…*/)
   data class StateSpec(
     val entry: List<ActionSpec>?,
     val on: Map<String, TransitionSpec>?,
     val guard: String?,
     val on_true: String?,
     val on_false: String?
   )
   data class TransitionSpec(val target: String?, val guard: String?)
   data class ActionSpec(val action: String)
   ```

3. **Load the YAML at startup**
   In `domain/src/main/kotlin/com/example/domain/fsm/FsmLoader.kt`:

   ```kotlin
   package com.example.domain.fsm

   import com.fasterxml.jackson.dataformat.yaml.YAMLMapper
   object FsmLoader {
     fun load(): FsmSpec = YAMLMapper()
       .registerModule(com.fasterxml.jackson.module.kotlin.KotlinModule())
       .readValue(
         this::class.java.getResource("/session.yaml")!!,
         FsmSpec::class.java
       )
   }
   ```

4. **Implement the interpreter**
   In `domain/src/main/kotlin/com/example/domain/fsm/StateMachineRunner.kt`:

   ```kotlin
   package com.example.domain.fsm

   class StateMachineRunner(
     private val spec: FsmSpec = FsmLoader.load()
   ) {
     private var state: String = "INIT"
     private val ctx = mutableMapOf<String, Int>().apply {
       spec.counters.forEach { put(it, 0) }
     }

     /** Call this to fire an event, returns new state */
     fun on(event: String): String {
       val stateSpec = spec.states[state]!!
       // 1) check guard transitions (CHECK_MASTERY / CHECK_REMEDIATION)
       // 2) else look up stateSpec.on[event]?.target
       val next = stateSpec.on?.get(event)?.target
         ?: throw IllegalStateException("No transition $state → $event")
       // 3) execute any entry actions: update counters in ctx
       spec.states[next]?.entry?.forEach { execAction(it.action) }
       state = next
       return state
     }

     private fun execAction(action: String) {
       when(action) {
         "evaluate_usage" -> TODO("hook into scoring engine")
         "reset_counters"  -> spec.counters.forEach { ctx[it] = 0 }
         // … handle all action names …
         else -> Unit
       }
     }

     fun currentState() = state
     fun context() = ctx.toMap()
   }
   ```

---

## D Wire it up in your ViewModel

1. **Inject the runner**
   Update `MainViewModel` constructor to take a `StateMachineRunner` instead of the old use-cases.

2. **Drive events → UI**

   ```kotlin
   class MainViewModel(
     private val fsm: StateMachineRunner,
     private val dispatcher: CoroutineDispatcher = Dispatchers.Main
   ) : ViewModel() {
     private val _uiState = MutableStateFlow<UiState>(UiState.Idle)
     val uiState: StateFlow<UiState> = _uiState

     fun startSession() = vmScope.launch {
       _uiState.value = UiState.Loading
       val next = fsm.on("start")
       renderState(next)
     }

     fun onTtsDone() {
       val next = fsm.on("tts_done")
       renderState(next)
     }

     fun onAsrResult(text: String) {
       // you’ll need to map text→“usage_correct”/“usage_incorrect” events
       val event = if (text.contains(fsm.context()["new_target"]!!.toString())) 
         "usage_correct" else "usage_incorrect"
       val next = fsm.on(event)
       renderState(next)
     }

     private fun renderState(state: String) {
       _uiState.value = when(state) {
         "INTRO_WORD"       -> UiState.CoachSpeaking(/*…use fsm.context…*/)
         "PROMPT_PRACTICE"  -> UiState.CoachSpeaking(/*…*/)
         "SESSION_DONE"     -> UiState.Congrats
         else               -> UiState.Waiting
       }
     }
   }
   ```

---

## E Replace tests with FSM-driven golden paths

1. **Write a single parameterized test** in `domain/src/test/kotlin/com/example/domain/fsm/StateMachineRunnerTest.kt` that:

   * Loads the same `session.yaml`.
   * Steps through each Gherkin scenario from the spec (e.g. `["start","tts_done","tts_done","asr_result"×3]`)
   * Asserts `runner.currentState()` and `runner.context()` at each step.

2. **Delete all the old integration & unit tests** around the use-cases—your one FSM test now covers the entire turn logic.

---

### Result

* **Zero reliance on build-time codegen** or unfamiliar Gradle plugins.
* **All session behavior driven by one `session.yaml`** that both humans and your LLM agents can read and edit.
* **Minimal code** for parsing and interpreting the DSL, easy to debug and maintain.
* **Clean TDD loop**: write one failing FSM-step test, implement one branch in `StateMachineRunner.on()`, repeat until green.

This plan preserves your existing JSON queue loading, prompt-builder, persistence wiring and Hilt setup, while giving you a **fresh, un-spaghetti’d** FSM core in under a day’s work.
