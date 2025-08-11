# Coaching Session State‑Machine Specification (v0.2)

## Purpose

This artifact is the **single source of truth** for the *turn‑based* flow that drives a single coaching session in **Lango**.  It is intentionally:

- **Machine‑readable** – the YAML spec can be parsed by an LLM or Kotlin/JVM test harness to auto‑generate contract tests or presenter code.
- **Human‑auditable** – tables & Gherkin examples let architects review, discuss and evolve the design quickly.

> **Key change in v0.2** – Aligns with `PROMPT.md` orchestration rules: (1) each new learning_item must be *used* by the student **three times** before it is_learned, (2) the very *first* introduction of the item follows a richer explanation template, (3) if correct usage stalls (either the student is not saying it in a way that is recognized or not saying it at all) the coach switches into a “repeat‑after‑me” remediation sub‑flow.

---

## 1  YAML State‑Machine Definition

```yaml
session: DialogueSession
version: 0.2
context:
  required_usages: 3            # student must use word correctly this many times
  repeat_mode_threshold: 2      # consecutive failures before entering RepeatAfterMe
  max_repeat_loops: 3           # safety net – prevent infinite loops

counters:
  usage: 0          # successful student uses of new target
  failures: 0       # consecutive incorrect attempts

states:
  INIT:
    on:
      start: INTRO_WORD

  # ─── 1 ░░ INTRODUCTION TURN ░░ ────────────────────────────────────────────────
  INTRO_WORD:
    entry:
      - action: coach_introduce_word      # LLMHANDLER_COACH_First_Use / step 1: say the word
      - action: coach_explain_word        # step 2: 5‑year‑old explanation
      - action: coach_example_sentence    # step 3: simple example with learned_pool
    on:
      tts_done: PROMPT_PRACTICE           # begin practice loop after TTS finished

  # ─── 2 ░░ PRACTICE LOOP (up to required_usages) ░░ ────────────────────────────
  PROMPT_PRACTICE:
    entry:
      - action: coach_prompt_dialogue     # LLMHANDLER_COACH_Practice_Dialogue / update counters.presentation / cue usage
    on:
      tts_done: WAIT_STUDENT

  WAIT_STUDENT:
    on:
      asr_result: EVALUATE_ATTEMPT
      timeout: PROMPT_PRACTICE            # student silent

  EVALUATE_ATTEMPT:
    entry:
      - action: evaluate_usage            # LLMHANDLER_STUDENT_Process_Response / update counters.usage, counters.failures
    on:
      usage_correct:
        target: CHECK_MASTERY
      usage_incorrect:
        target: CHECK_REMEDIATION

  CHECK_MASTERY:
    guard: counters.usage >= required_usages
    on_true: COMPLETE_WORD
    on_false:
      target: PROMPT_PRACTICE             # continue practice loop

  CHECK_REMEDIATION:
    guard: counters.failures >= repeat_mode_threshold
    on_true: REPEAT_AFTER_ME
    on_false: PROMPT_PRACTICE             # try again normally

  # ─── 3 ░░ REMEDIATION SUB‑FLOW ░░ ─────────────────────────────────────────────
  REPEAT_AFTER_ME:
    entry:
      - action: coach_slow_repeat         # LLMHANDLER_COACH_Repeat_After_Me
    on:
      tts_done: WAIT_REPEAT

  WAIT_REPEAT:
    on:
      asr_result: EVALUATE_REPEAT
      timeout: REPEAT_AFTER_ME            # keep nudging until ASR result or loop cap

  EVALUATE_REPEAT:
    entry:
      - action: evaluate_usage            # LLMHANDLER_STUDENT_Process_Response / update counters.usage, counters.failures
    on:
      usage_correct:
        target: RESET_REMEDIATION
      usage_incorrect:
        target: LOOP_OR_FAIL

  RESET_REMEDIATION:
    entry:
      - action: reset_failures_counter    # failures = 0
    on:
      done: CHECK_MASTERY

  LOOP_OR_FAIL:
      guard: counters.failures >= max_repeat_loops
      on_true: COMPLETE_WORD               # fail‑out safety‑net

  # ─── 4 ░░ WORD COMPLETE ░░ ────────────────────────────────────────────────────
  COMPLETE_WORD:
    entry:
      - action: add_word_to_learned_pool  # persist queue move + counts
      - action: coach_positive_reinforce  # "Gut gemacht!" example sentence
      - action: pick_next_new_word        # dequeue next NEW_QUEUE entry
    on:
      next_word_available: RESET_COUNTERS
      no_more_words: SESSION_DONE

  RESET_COUNTERS:
    entry:
      - action: reset_counters            # usage = 0, failures = 0 
    on:
      done: INTRO_WORD                    # start flow for the next word

  SESSION_DONE:
    type: final

invariants:
  - counters.usage <= required_usages
```

### State Machine Notes

- **Counters** are explicit top‑level variables so both human and LLM can reason about guards easily.
- **Repeat‑After‑Me flow** is only entered after `repeat_mode_threshold` consecutive incorrect attempts (no usage increment).
- **Loop‑cap driver** is now `counters.failures`; the separate `state_loop_count` counter has been removed.
- Safety nets (`max_repeat_loops`) prevent a stuck session if ASR never recognises the student.

---

## 2  Event Glossary (additions)

| Event                 | Origin                | Meaning                                              |
| --------------------- | --------------------- | ---------------------------------------------------- |
| `usage_correct`       | LLM Handler, Student  | student’s utterance contained the exact `new_target` |
| `usage_incorrect`     | LLM Handler, Student  | student spoke but did **not** use `new_target`       |
| `next_word_available` | Queue manager         | NEW\_QUEUE not empty                                 |
| `no_more_words`       | Queue manager         | NEW\_QUEUE is empty → session ends                   |
| `done`                | Internal pseudo‑event | Entry/completion event used in simple choice states  |

> All previous events (`start`, `tts_done`, `asr_result`, `timeout`) remain unchanged.

---

## 3  Reference Scenarios (Gherkin)

```gherkin
Feature: New word cycle with mandatory three correct usages

  Scenario: Ideal path – student masters word on first three attempts
    Given the session starts and a new_target is dequeued
    When the coach introduces the word (INTRO_WORD)
    And the student repeats it correctly on attempt 1, 2 and 3
    Then counters.usage becomes 3
    And the word is moved to the learned_pool
    And the session moves automatically to the next new_target

  Scenario: student misses twice, enters repeat‑after‑me, then succeeds
    Given the session starts with new_target "Wie viel kostet das?"
    When the student does not use the word on two consecutive attempts
    Then the state machine enters REPEAT_AFTER_ME
    When the student repeats correctly inside remediation
    Then counters.usage = 1, failures reset to 0
    And normal practice continues until total usage = 3

  Scenario: Repeat‑after‑me loop cap safety
    Given the student never pronounces the word correctly
    And counters.failures reaches max_repeat_loops
    Then the session exits remediation and marks the word learned
    Note: in the dialogue, the coach will prefer items with lower usage, so if the student does not use the word correctly, it will continue to be presented in opportunities to continue and practice usage 
```

---

## 4  How to Use This Spec

1. **Code generation** – Parse the YAML to auto‑create a sealed‑class hierarchy or  `StateMachine<Context, Event>` instance in Kotlin.
2. **Golden‑path tests** – Use the Gherkin examples with a DSL to produce *living* regression tests; if behaviour changes you update this spec first, regenerate tests, then fix code.
3. **AI prompt derivation** –  LLM can read `state.name` + `context` to decide which *prompt template* (Intro, Normal Dialogue, Repeat‑After‑Me) to render.
4. **Architectural governance** – Pull requests that alter session behaviour must bump `version`, update YAML + scenarios, and include migration notes for queues if needed.

---

*(End of file)*