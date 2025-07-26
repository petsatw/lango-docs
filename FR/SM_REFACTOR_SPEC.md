
# SM_REFACTOR Detail: State Machine Refactor — Detailed Specification

**Last Updated:** July 25, 2025

---

## §1. Must-Be-True Invariants & Core Domain Model

| Area                | Invariant / Rule                                                                                          | Source / Note                                                                                   |
| ------------------- | --------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Mastery Counter** | `counters.usage ≤ config.requiredUsages` — the coach never asks for more uses than the configured target. | Refer `SessionConfig.requiredUsages`                                                          |
| **Repeat-Loop Cap** | `state_loop_count ≤ config.maxRepeatLoops` — ensures remediation loops terminate.                         | Refer `SessionConfig.maxRepeatLoops`                                                          |
| **Failure → Remed.**| Enter `REPEAT_AFTER_ME` only if `counters.failures ≥ config.repeatModeThreshold`.                         | Refer `SessionConfig.repeatModeThreshold`                                                    |

### `newTarget` Semantics

- **Ownership**: Exactly one active `LearningItem`:
  - Popped from `new_queue` when session starts or previous item mastered.
  - Persisted in header JSON until moved to `learned_pool`.

- **Lifecycle Flags**:
  - `presentationCount` ⬆️ on each `INTRO_WORD` or `coach_prompt_dialogue`.
  - `usageCount` ⬆️ on every correct ASR match.
  - `learned = true` when `usageCount ≥ config.requiredUsages`.

- **Queue JSON Schema**:
  ```jsonc
  {
    "schemaVersion": 1,
    "new_queue": [{ "id": "...", "token": "...", "presentationCount": 0, "usageCount": 0, "learned": false }, ...],
    "learned_pool": [ ... ]
  }
  ```

---

## §2. Runtime DSL Interpreter

### §2.1 FSM Spec → Data Classes

- **FsmSpec**:
  ```kotlin
  data class FsmSpec(
    val initialState: String,
    val states: Map<String, StateDefinition>
  )
  ```

- **StateDefinition**:
  ```kotlin
  data class StateDefinition(
    val entry: List<ActionDefinition> = emptyList(),
    val transitions: Map<String, String> = emptyMap()
  )
  ```

- **ActionDefinition**:
  ```kotlin
  data class ActionDefinition(
    val name: String,
    val params: Map<String, Any> = emptyMap()
  )
  ```

### §2.2 Must-Implement Actions

List of core `entry:` actions (copied from conversation “Runtime DSL Interpreter”):

- `coach_introduce_word`
- `coach_explain_word`
- `coach_example_sentence`
- `coach_prompt_dialogue`
- `coach_slow_repeat`
- `evaluate_usage`
- `add_word_to_learned_pool`
- `reset_counters`
- `reset_failures_counter`
- `pick_next_new_word`
- `coach_positive_reinforce`

**Action Semantics**:

- **`evaluate_usage`**:  
  - Inspect last ASR result in `ctx.asrResult`.  
  - If correct: `ctx.usageCount++`.  
  - If `usageCount ≥ config.requiredUsages`, schedule `add_word_to_learned_pool`.  
  - Else if neg: `ctx.failures++`.

- **`add_word_to_learned_pool`**:  
  - Mark `ctx.currentItem.learned = true`.  
  - Invoke repository persistence.  
  - Reset `ctx` counters.

- **`reset_counters`**, **`reset_failures_counter`**:  
  - Zero out `ctx.usageCount` or `ctx.failures`.

- **`pick_next_new_word`**:  
  - Pop head of `new_queue` into `ctx.currentItem`.

- **`coach_*` actions**:  
  - Generate appropriate TTS prompt via LLM helper or template.

### §2.3 Guard Logic & Unhandled Events

- **Unhandled-event fallback**:  
  - Runner must `throw IllegalStateException("Unhandled event $event in $state")`.

- **Guard thresholds**:  
  - No magic numbers; compare against `config` values only.

### §2.4 Context Mutation & Logging

- **Context Map (`ctx`)**:
  ```kotlin
  data class Context(
    var usageCount: Int,
    var presentationCount: Int,
    var failures: Int,
    var currentItem: LearningItem,
    var headerJson: HeaderJson
  )
  ```

- **Structured Logging** on each `.on(event)`:
  ```json
  {
    "ts": "...",
    "prevState": "PROMPT_PRACTICE",
    "event": "asr_correct",
    "nextState": "WAIT_STUDENT",
    "ctxDiff": { "usageCount": { "from": 2, "to": 3 } }
  }
  ```

- **Debug Menu Hook**:
  - Hidden **︙ Debug** in UI to dump last 50 transitions from in-memory ring buffer.

---

## §3. ViewModel & UI Integration

### §3.1 FSM State → `UiState` Mapping

| FSM State            | UiState                              | Blocking?         | Notes                           |
| -------------------- | ------------------------------------ | ----------------- | ------------------------------- |
| `INTRO_WORD`         | `CoachSpeaking(text)`                | Yes (wait TTS)    | TTS triggered automatically    |
| `PROMPT_PRACTICE`    | `CoachSpeaking(text)`                | Yes               |                                 |
| `REPEAT_AFTER_ME`    | `CoachSpeaking(text, slow=true)`     | Yes               |                                 |
| `WAIT_STUDENT`       | `Waiting` (`Tap to Speak`)           | No                | ASR or timeout fires next event|
| `WAIT_REPEAT`        | `Waiting`                            | No                | User repeats phrase            |
| `SESSION_DONE`       | `Congrats`                           | Terminal          |                                 |

### §3.2 ViewModel Stub Example

```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
  private val runner: StateMachineRunner,
  private val repository: LearnedPoolRepository
) : ViewModel() {
  private val _uiState = MutableStateFlow<UiState>(UiState.Idle)
  val uiState: StateFlow<UiState> = _uiState

  init {
    val header = repository.loadQueues()
    runner.initialize(header)
  }

  fun onEvent(event: String) {
    val next = runner.on(event)
    _uiState.value = mapToUiState(next)
  }
}
```

---

## §4. Testing & Quality Gates

### §4.1 Golden-Path Parameterized Test

- **Location:** `domain/src/test/com/example/domain/fsm/StateMachineRunnerTest.kt`
- **Inject** `SessionConfig(requiredUsages=3, maxRepeatLoops=3, repeatModeThreshold=2)`
- **Test Sequence:**  
  ```kotlin
  val events = listOf("start","tts_done","asr_correct","asr_correct","asr_correct")
  events.forEach { state = runner.on(it) }
  assertEquals("SESSION_DONE", state)
  ```

### §4.2 Sad-Path Tests

- **Unhandled events throw**:
  ```kotlin
  @ParameterizedTest
  @ValueSource(strings = ["tts_done","asr_error","unknown_event"])
  fun `runner throws on bad events`(evt: String) {
    assertThrows<IllegalStateException> { runner.on(evt) }
  }
  ```
- **Config variability**:
  ```kotlin
  val cfg5 = SessionConfig(5,3,2)
  val runner5 = StateMachineRunner(spec,cfg5)
  // drive 5 correct ASR → SESSION_DONE only after 5th
  ```

---

## §5. Persistence Integration

- **HeaderJson** holds `new_queue` and `learned_pool`.
- **LearnedPoolRepository** interface:
  ```kotlin
  interface LearnedPoolRepository {
    fun saveQueues(headerJson: HeaderJson)
    fun loadQueues(): HeaderJson
  }
  ```
- **Runner Hooks**:
  - On `add_word_to_learned_pool` → call `saveQueues`.
  - On `initialize(header)` → load into runner context.

---

## §6. Roll-out & CI Metrics

- **PR Checklist**:
  - [ ] Domain module compiles (`./gradlew :domain:compileKotlin`)
  - [ ] Domain tests pass (`./gradlew :domain:test`)
  - [ ] App module compiles (`./gradlew :app:compileKotlin`)
  - [ ] UI stubs render without error
- **Monitor**:
  - Track first-compile and first-green-test rates via CI dashboards.
  - Alert if rates drop below 90%/80% thresholds.

---

*End of SM_REFACTOR_Detail.md*
