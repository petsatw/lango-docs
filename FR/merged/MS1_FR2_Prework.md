### TL;DR – **The current codebase is *not yet ready* to start MS-1 FR-2.**

Two key assumptions in **MS1\_FR2.md** clash with what actually exists in the snapshot:

| FR-2 assumption                                                                                                        | Snapshot reality                                                                                                   | Impact                                                                                                          |
| ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| **StartSessionUseCase dequeues + resets the first `new_target`.**                                                      | It only *resets* counts in place; the item remains in `newQueue`                                                   | B1-UT will fail unless we add a dequeue step.                                                                   |
| **GenerateDialogueUseCase exposes `buildInitialPrompt()` that emits the JSON schema (header + body) defined in FR-2.** | Only a `generatePrompt()` method exists; it builds a **plain string** prompt and sends it straight to `LlmService` | PS-1 will fail; there is no JSON builder, no orchestration-file logic, nor a place to snapshot-test the result. |

Everything else (entities, repository bootstrap, DI for `AssetManager`, kotlinx-serialization, etc.) already lines up with FR-2 expectations, so only the **two mis-alignments above require pre-work** before starting the feature.

---

## 1  Minimal Pre-work Needed (no FR-2 logic yet)

> These are scaffolding steps only—they don’t build the final feature, they just make the codebase *able* to accept the FR-2 changes without breaking existing tests.

### 1.1 Refactor **StartSessionUseCase** to allow dequeue

*Goal:* Expose a helper you can re-use later without changing behaviour today.

1. **Add** an internal extension on `Queues` – `fun dequeueAndReset(): LearningItem?`
   *File:* `domain/.../Queues.kt` inside the existing `Queues` class.

   * Remove and return `newQueue.firstOrNull()`; zero both counters.

2. **Modify** `StartSessionUseCase.startSession()` to *call* that helper but *still* return the same `Queues` object so FR-1 tests stay green.
   (You will wire the new logic in FR-2.)

### 1.2 Create a placeholder interface for the initial-prompt builder

*Goal:* Introduce the API surface FR-2 will flesh out, while keeping GenerateDialogueUseCase untouched for now.

```kotlin
interface InitialPromptBuilder {
    fun build(queues: Queues, sessionId: UUID = UUID.randomUUID()): String
}
```

1. **Make a thin default implementation** that simply throws `NotImplementedError()`.
   *File:* `domain/.../NoopInitialPromptBuilder.kt`

2. **Inject** this builder into `GenerateDialogueUseCase` constructor but *do not use it yet*—just store it for later.
   This changes no behaviour today but lets FR-2 drop in its real logic without further signature changes.

3. **Update DI module** to bind `InitialPromptBuilder` to the noop impl for now.

### 1.3 Guard tests

Add a quick unit test that verifies the noop builder throws; mark it `@Ignore` so CI remains green until FR-2 lands.
This documents the intentional gap.

---

## 2  Checked Assumptions That DO Match the Snapshot

| Area                                                                                                | Verification |
| --------------------------------------------------------------------------------------------------- | ------------ |
| **Domain entities** (`LearningItem`, `Queues`) already expose the counters FR-2 cares about         |              |
| **Repository** provides `loadQueues()` / `saveQueues()` exactly as FR-2 cites                       |              |
| **AssetManager DI** is configured, letting future code read `PROMPT.md` (seen in `di/AssetsModule`) |              |
| **kotlinx-serialization-json** already declared in `libs.versions.toml` and domain `build.gradle`   |              |

No gaps or version conflicts here.

---

## 3  Dependency & Sequencing Notes

* **No new external libraries** are required for the scaffolding; UUID is from the JDK.
* Perform *1.1* first (safe, self-contained), then *1.2* (signature change), then *1.3*.
  This keeps merge pain low and isolates behavioural changes.

---

## 4  Documentation Updates After Pre-work

| Doc               | Update                                                                                                                 |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **BUILD SPEC.md** | Note the new `InitialPromptBuilder` interface and the `Queues.dequeueAndReset()` helper under Domain layer.            |
| **MS1\_Spec.md**  | Mark that FR-2 coding will plug into these placeholders; no business behaviour changes yet.                            |
| **TDD.md**        | Add entry for ignored “InitialPromptBuilder not implemented” test so future devs know to un-ignore when FR-2 is built. |

---

## 5  Build Points (hand-offs)

1. **After the helper + DI refactor compiles and existing FR-1 tests pass.**
2. **When FR-2 branch starts**, swap in the real `InitialPromptBuilder` and activate dequeue logic; un-ignore the test and add B1-UT / PS-1 fixtures.

---

With just these light refactors in place, the codebase will align structurally with the assumptions in **MS1\_FR2.md**, yet still preserve all FR-1 behaviour—putting you on solid ground to implement the actual FR-2 features next.
