# Milestone 1 – Context & Development Guide

This document is **everything the team needs** (aside from the current *dev* branch itself) to implement Milestone 1:  
*“Application initialised and ready (UAT mode) – loads queues, shows the first prompt, receives one fake LLM reply, persists state on End Session.”*

---

## 1 Pain Points & Gaps in the current code base

> The excerpts below are quoted **verbatim** from the *dev* branch so you don’t need to open the files.  
> File‑paths are given as `⟦path/to/File.kt⟧`.

| # | Area | What you will see in code | Why it must be fixed for Milestone 1 |
|---|------|---------------------------|--------------------------------------|
| P‑1 | **Repository – asset copy & persistence** | The repository opens assets directly and never writes to `filesDir`.<br><br>```kotlin
// ⟦data/LearningRepositoryImpl.kt⟧ lines 19‑27
val newQueuePath = filePaths?.first ?: "core_blocks.json"
val learnedQueuePath = filePaths?.second ?: "learned_queue.json"

val newQueueStream = context.assets.open(newQueuePath)
val learnedQueueStream = context.assets.open(learnedQueuePath)
return loadQueues(newQueueStream, learnedQueueStream)
``` | The very first UAT criterion is *“file load is successful …”* **and** later calls to `saveQueues` must persist progress. Without copying assets to real files we will overwrite progress on every launch. |
| P‑2 | **Wrong filenames & inconsistent field names** | *Filename* mismatch: `core_blocks.json` vs spec “**new_queue.json**”.<br>*Field* mismatch between JSON and data class:<br>JSON → `"presentation_count"`; Kotlin → `presentationCount`.<br><br>```json
// ⟦app/assets/core_blocks.json⟧ (first object)
{ "id": "german_CP001", "token": "Entschuldigung", "presentation_count": 0, "usage_count": 0 }
```<br>```kotlin
// ⟦domain/LearningItem.kt⟧
@SerialName("presentation_count") var presentationCount: Int = 0,
``` | Align naming convention (snake_case in JSON, camelCase in Kotlin with `@SerialName`) and rename the asset from core_blocks.json to `new_queue.json`. |
| P‑3 | **Prompt builder incomplete / corrupt** | The strings in `GenerateDialogueUseCase` are truncated (`…`).<br><br>```kotlin
// ⟦domain/GenerateDialogueUseCase.kt⟧ lines 14‑21
promptBuilder.append("Explain what '${newTarget.toke... in a very short sentence that a 5-year-old would understand. ")
``` | These ellipsis strings do **not compile**. A correct prompt builder must insert the new target, the learned pool list, **and** the JSON output‑schema instruction required by later milestones. |
| P‑4 | **No speech mocks** | No `FakeLlmService` or `FakeTtsService` is present; the interface exists only:<br>```kotlin
interface LlmService { suspend fun generateDialogue(prompt: String): String }
``` | Milestone 1 requires a *pre‑defined* LLM reply so the domain flow can proceed offline. |
| P‑5 | **UI skeleton missing** | `MainActivity` sets no layout and there is no XML file.<br>```kotlin
// ⟦app/MainActivity.kt⟧
// setContentView(R.layout.activity_main) // No layout yet
``` | The milestone calls for a scrollable text area plus a **Start/End** button. |
| P‑6 | **Dependency‑injection switch absent** | No Hilt module provides fake vs real services, and there is no `BuildConfig.UAT_MODE` flag. | Without a build‑time switch we cannot run Milestone 1 in offline UAT mode while keeping the real implementations ready for Milestone 2. |
| P‑7 | **Tests give false positives** | `SessionIntegrationTest` only checks that `saveQueues()` is **called**; it never inspects the file written. | A green test suite at HEAD does **not** prove that loading/saving actually works; extra tests are required. |

Keep these pain‑points visible while you work through the tasks below.

---

## 2 Units of Work (modular tasks & their integration checks)

Tackle the units **in the order listed**.  
After the **Exit check✓** of a unit passes (locally & on CI) you may move to the next unit.

### A Data layer

| ID | Task | Input / spec notes | Exit check ✓ |
|----|------|-------------------|--------------|
| **A‑1** | **Asset‑to‑file copy & load**<br>• On first launch copy `new_queue.json` and `learned_queue.json` from `assets/` into `context.filesDir`.<br>• Implement `LearningRepositoryImpl.loadQueues()` that: (a) prefers the *file* in `filesDir` if it exists, else copies assets and parses; (b) returns a `Queues(newQueue, learnedPool)` object.<br>• Use Kotlinx Serialization with `ignoreUnknownKeys = true` for forwards‑compat. | Replace all hard‑coded `"core_blocks.json"` strings with `"new_queue.json"`. | **Instrumentation test** (emulator): delete app data → launch → `loadQueues()` returns non‑empty queues identical to the asset contents. |
| **A‑2** | **Persistent save + mutex**<br>• Add `private val ioMutex = Mutex()`; ensure `saveQueues()` and `loadQueues()` are wrapped in `withContext(Dispatchers.IO) { ioMutex.withLock { … } }`.<br>• Save via temp‑file + atomic rename to avoid corruption. |  | **Unit test**: perform 5 concurrent coroutine calls to `saveQueues()`; assert no `ConcurrentModificationException` and final file parses. |
| **A‑3** | **Graceful error paths**<br>• On JSON `SerializationException` fallback to empty list and emit `Result.failure`.<br>• When file missing, copy from assets. |  | **Unit test**: feed malformed JSON string `"{bad"` → repository returns empty queues and non‑fatal `Result.failure`. |

### B Domain layer

| ID | Task | Input / spec notes | Exit check ✓ |
|----|------|-------------------|--------------|
| **B‑1** | **StartSessionUseCase polish**<br>After dequeueing the first `new_target`, ensure its `presentationCount`/`usageCount` are **zeroed**.<br>Leave other counts unchanged. | Already half‑implemented but relies on unsafe cast `null as Pair<String,String>?`; replace with nullable param default. | **Unit test**: given a `Queues` where first target has non‑zero counts, after `startSession()` they are **0**. |
| **B‑2** | **Prompt builder v1**<br>Replace the truncation marks in `GenerateDialogueUseCase` with real sentences.<br>Add the mandatory output‑schema instruction **exactly**:<br>```text
Please answer on the first line with JSON:
{ "coach_text": "…", "presentation_ids": [], "usage_ids": [] }
Then type a line containing only ---
Then repeat coach_text for the learner.
```
Increment `presentationCount` **inside** the use‑case when introducing the new target. | The JSON header requirement anticipates Milestone 2—do **not** omit it. | **Snapshot test** (`assertEquals(expectedPrompt, generatePrompt())`). |

### C Infrastructure mocks

| ID | Task | Exit check ✓ |
|----|------|--------------|
| **C‑1** | **FakeLlmService**: reads `sample_turn_1.json` from `/res/raw/` and returns it.<br>`sample_turn_1.json` must include a `"presentation_ids"` array containing the first target’s id so the domain layer increments its count. | Unit test: `generateDialogue("any")` returns file contents exactly. |
| **C‑2** | **FakeTtsService**: implement `suspend fun speak(text:String): Result<Unit>` → always `Result.success(Unit)` immediately. | Unit test passes. |

### D Presentation layer

| ID | Task | Depends on | Exit check ✓ |
|----|------|------------|--------------|
| **D‑1** | **XML layout & ids**<br>`activity_main.xml`:<br>```xml
<ScrollView …>
  <TextView android:id="@+id/txtCoach" …/>
</ScrollView>
<Button android:id="@+id/btnStartEnd" … />
``` | – | Layout preview renders; ids compile. |
| **D‑2** | **ViewModel & UiState**<br>Define `sealed class UiState { object Loading; data class Prompt(val text:String) }`.<br>`MainViewModel` exposes `StateFlow<UiState>` and two events `onStartClicked()` and `onEndClicked()`. | A‑1, B‑1, B‑2, C‑1, C‑2 | Unit test: flow emits Loading → Prompt(initial) at launch; after `onStartClicked()` emits Prompt(fakeReply). |
| **D‑3** | **MainActivity wiring**<br>Set content view, observe `uiState`, bind text, toggle button label.<br>Handle End click: calls `saveQueues()` then `finish()`. | D‑1, D‑2 | **UAT TEST ONLY**: the user will launch → text visible; click Start → new text; click End → activity `isFinishing`. |

### E Dependency injection

| ID | Task | Exit check ✓ |
|----|------|--------------|
| **E‑1** | **Hilt module + build flag**<br>Add `const val UAT_MODE = true` in `BuildConfig` (makefile flag).<br>Provide bindings: when `UAT_MODE` `true` provide *fake* services else real.<br>Inject repository singleton. | Instrumentation test: set `UAT_MODE=true`, injected `LlmService` is `FakeLlmService`. |

---

## 3 Mandatory fixtures to add

| Path (suggested) | Contents |
|------------------|----------|
| `app/src/main/assets/new_queue.json` | Copy of old `core_blocks.json` but renamed. |
| `app/src/main/assets/learned_queue.json` | Existing file — keep. |
| `app/src/main/res/raw/sample_turn_1.json` | ```json
{ "coach_text": "Hallo! Ich bin dein Coach…", "presentation_ids": ["german_CP001"], "usage_ids": [] }
---
Hallo! Ich bin dein Coach…
``` |
| `app/src/test/resources/initial_prompt.txt` | Golden prompt for B‑2 snapshot test. |

---

## 4 Complete Definition of “Done” for Milestone 1

> Tick **every** item before merging the final PR.

1. App cold‑starts, copies `new_queue.json` & `learned_queue.json` to `filesDir`.  
2. Initial prompt matches `initial_prompt.txt` and is fully visible in the scrollable area.  
3. “Start Session” injects the fake LLM reply, increments `presentationCount` for the id in `presentation_ids`, updates the UI, and flips the button label to “End Session”.  
4. “End Session” persists queues (confirmed by reading the file) and closes the activity.  
5. All tests introduced in Units **A–E** pass on CI.  
6. Line coverage ≥ **70 %** in `:data` and `:domain` modules.  
7. No `TODO()` or compile warnings in changed files.  

Once the checklist is green, Milestone 1 is achieved and the repo is ready for Milestone 2 (live LLM & TTS).

---

*— End of Context & Development Guide —*
