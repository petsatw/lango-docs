# Step 2: Paste & Parse `session.yaml` + Minimal Runner  
**Goal:** Validate FSM spec, parser, and runner in pure‑JVM.  
1. Copy `session.yaml` to `domain/src/main/resources/`  
2. Add Jackson‑YAML deps in `domain/build.gradle.kts`  
3. Define spec data‑classes (`FsmSpec`, `StateDefinition`, `ActionDefinition`)  
4. Stub out `StateMachineRunner` to load spec, apply `config`, and invoke guards/actions  
5. Golden‑path JUnit: drive `start`,`tts_done`,`asr_correct×3` → expect `SESSION_DONE`.  

> **See SM_REFACTOR_SPEC §2.1** “FSM Spec → Data Classes”  
> **See SM_REFACTOR_SPEC §2.2** “Must‑Implement Actions”  
> **See SM_REFACTOR_SPEC §2.3** “Guard Logic & Unhandled Events”

# Working Contract
---

## 1) Canonical `session.yaml`

* **Where it comes from:** Copy the YAML from `ARCH_TL` v0.2 into `domain/src/main/resources/session.yaml`. Keep `session: DialogueSession` and `version: 0.2` at the top.
* **Owner & bumps:** Architecture owns behavioral changes; Domain owns loader/runtime. Any behavior change → bump `version` in YAML + adjust tests (NEXT\_STEP already calls for this paste+parse step).

## 2) YAML → Kotlin fidelity (Step-2 scope)

* **Parse the whole v0.2 shape** (guards, `on_true/on_false`, remediation states), but **execute only the golden path in tests**. The plan/spec for Step-2 asks for a minimal runner and golden-path test.
* **Data classes:** Start with the minimal spec models in TDD\_F §2.1 (`FsmSpec`, `StateDefinition`, `ActionDefinition`).
  **Loader rule:** if a state has `guard`/`on_true`/`on_false`, keep those fields in an **extended** in-memory representation (wrapper around `StateDefinition`). We are not required to *execute* remediation branches in Step-2 tests, but we **must** validate and retain them at parse time to stay faithful to ARCH\_TL.

## 3) Event names (hard-coded in Step-2)

* **External events:** `start`, `tts_done`, `asr_result`, `timeout`.
* **Internally emitted by actions:** `usage_correct`, `usage_incorrect`, `next_word_available`, `no_more_words`, `done`.
* **Producer of `usage_*`:** the runner, inside `evaluate_usage` (per TDD\_F §2.2 semantics).
* **Test input event shape:** `runner.on("asr_result", meta = mapOf("text" to "...", "isCorrect" to true))`.

> **Change from earlier:** Calling `asr_result` (with meta) replaces the shorthand `asr_correct` that appears in DP\_F’s one-liner. That one-liner remains a mnemonic, not the literal API.

## 4) Public state ids (stable contract)

Exactly these, matching ARCH\_TL:
`INIT`, `INTRO_WORD`, `PROMPT_PRACTICE`, `WAIT_STUDENT`, `EVALUATE_ATTEMPT`, `CHECK_MASTERY`, `COMPLETE_WORD`, `SESSION_DONE`.

## 5) Entry action semantics (what mutates now)

Implement the minimal semantics described in TDD\_F §2.2 for Step-2:

* `coach_prompt_dialogue`: increment presentation counter on the current item.
* `evaluate_usage`: read last ASR payload; if correct ⇒ `usage++`, `failures=0`, **emit** `usage_correct`; else ⇒ `failures++`, **emit** `usage_incorrect`.
* `add_word_to_learned_pool`: mark current item `learned=true`; in Step-2 keep it **in-memory only** (persistence is Step-4 per plan).

All other `coach_*` actions: **no-op + trace** in Step-2.

## 6) Action registry shape

```kotlin
fun interface Action { suspend fun invoke(ctx: Ctx): Effects }
data class Effects(val emit: List<String> = emptyList())

interface ActionRegistry { fun get(name: String): Action }
```

No parameters needed in Step-2 (parse and ignore any `params`).

## 7) Runner context fields (authoritative names)

```kotlin
data class Ctx(
  var usage: Int = 0,
  var failures: Int = 0,
  var stateLoopCount: Int = 0,
  var presentationCount: Int = 0,     // mirrors LearningItem field for tests
  var currentItem: LearningItem,
  var queues: Queues,
  var lastAsrText: String? = null,
  var lastAsrCorrect: Boolean = false
)
```

Map YAML counters → `Ctx` directly (`usage`, `failures`, `stateLoopCount`).
LearningItem counters (`presentationCount`, `usageCount`) remain your domain model (TDD\_F §1).

## 8) Thresholds—single source of truth

All comparisons use `SessionConfig(requiredUsages, maxRepeatLoops, repeatModeThreshold)`; **no literals**.

## 9) Queues in Step-2

* **No repository I/O** yet (Step-4 adds persistence).
* Provide a tiny fixture in the test (or `test/resources`) with `schemaVersion: 1`, `new_queue` and `learned_pool` fields as in TDD\_F §1.

## 10) Mastery side-effects (Step-2)

When `usage >= requiredUsages`, follow ARCH\_TL guard → `COMPLETE_WORD` and run `add_word_to_learned_pool` (in-memory).
**Then end the session for tests** by emitting `no_more_words` → `SESSION_DONE`. We do **not** chain a second word in Step-2 tests (multi-word flow belongs to later steps even though YAML has `pick_next_new_word`).

## 11) Strictness on unknown events

Throw `IllegalStateException("Unhandled event '$event' in state '$state'")` (as TDD\_F §2.3 requires).

## 12) Guard evaluation errors

Step-2 supports the guards present in v0.2:

* `counters.usage >= required_usages` and
* `counters.failures >= repeat_mode_threshold` (parse-time only; not exercised).
  If the YAML contains a different guard shape, **fail at load-time** with a clear message (aligns with NEXT\_STEP: “invoke guards/actions”).

## 13) YAML loader details

* **Library:** Jackson YAML + Kotlin module (already in MY\_CB deps).
* **Constraints:** No anchors/custom tags in v0.2 (straight YAML per ARCH\_TL snippet).

## 14) Resource location & CI

* Place the file at `domain/src/main/resources/session.yaml`. Step-2 is pure-JVM (no Android deps).

## 15) Golden-path test (lock this exact sequence & expected states)

**Config:** `requiredUsages=3, maxRepeatLoops=3, repeatModeThreshold=2` (as in TDD\_F §4.1).

**Events driven by the test:**

```
start
tts_done
tts_done
asr_result (isCorrect=true)
asr_result (isCorrect=true)
asr_result (isCorrect=true)
```

(First `tts_done` moves `INTRO_WORD`→`PROMPT_PRACTICE`; second moves `PROMPT_PRACTICE`→`WAIT_STUDENT` as per ARCH\_TL transitions.)

**Expected visited states (assert exact list):**

```
INIT
INTRO_WORD
PROMPT_PRACTICE
WAIT_STUDENT
EVALUATE_ATTEMPT
CHECK_MASTERY
PROMPT_PRACTICE
WAIT_STUDENT
EVALUATE_ATTEMPT
CHECK_MASTERY
PROMPT_PRACTICE
WAIT_STUDENT
EVALUATE_ATTEMPT
CHECK_MASTERY
COMPLETE_WORD
SESSION_DONE
```

These transitions correspond to ARCH\_TL’s state edges for intro→practice→evaluate→guard→complete-word.

> **Change from earlier:** I’ve removed the shorthand “asr\_correct×3” (DP\_F mnemonic) in favor of the actual event+meta the runner consumes and the ARCH\_TL event model.

## 16) ASR fixture shape

`asr_result` carries meta. The runner sets `ctx.lastAsrText` and `ctx.lastAsrCorrect`. `evaluate_usage` uses `isCorrect` if present; otherwise falls back to simple token match. Step-2 tests **should** pass `isCorrect=true` to avoid string-matching flakiness (this keeps things focused on the FSM wiring).

## 17) Legacy orchestrator

No changes to `MainViewModel`/DI in Step-2; all of that is Step-3 and beyond. Leave `CoachOrchestrator*` intact for now.

## 18) Package names (avoid collisions)

Put new code in:

* `com.example.domain.fsm` (runner/spec/loader)
* `com.example.domain.fsm.actions` (registry & built-ins)
  DP\_F also asks additions to follow `com.example.domain` conventions.

## 19) Minimal tracing (now) vs structured logging (later)

For Step-2: a tiny `TransitionLogger` that defaults to `println/Timber.d` is sufficient. Structured buffer & UI debug hook are explicitly **post-MVP** in the spec and DP\_F Step-6 adds the ring buffer.

---

# Step-2 build checklist (exact files & code stubs)

1. **Add file** `domain/src/main/resources/session.yaml` (copied from ARCH\_TL v0.2).
2. **Gradle:** ensure Jackson YAML + Kotlin module are present in `domain` (already declared in MY\_CB libs).
3. **Spec models:** implement TDD\_F §2.1 classes (`FsmSpec`, `StateDefinition`, `ActionDefinition`) and a thin wrapper for guard states.
4. **Loader:** parse the v0.2 YAML including `guard/on_true/on_false`. Fail fast on unknown guard shapes (Step-2 scope).
5. **Runner API:**

   ```kotlin
   fun initialize(queues: Queues): String // returns initial state id
   fun on(event: String, meta: Map<String, Any?> = emptyMap()): String
   ```

   On each transition, run `entry` actions (emit any internal events) and return the next state id. Unknown event ⇒ throw.
6. **Actions:** implement Step-2 subset per §5 above; others no-op.
7. **Tests:** one **golden-path** test and one **strictness** test (unhandled event throws). DP\_F/TDD\_F already outline both.

---

## Final sanity: does the updated context change anything else?

* **Yes (guard shape & event API):** We now require the loader to understand `guard/on_true/on_false`, and we standardize tests to call `asr_result` with `isCorrect=true`. Both are aligned to ARCH\_TL and DP\_F/TDD\_F language.
* **No (module boundaries & persistence):** Step-2 remains pure-JVM; persistence waits for Step-4.
* **No (strictness):** Unknown events **throw**—unchanged.

## END OF contract

After completing the work defined in this task. STOP. The team will review and test. Your work is complete at this point. 