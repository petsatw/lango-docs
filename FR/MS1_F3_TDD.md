
# Feature-Level TDD: Speech Mocks (Milestone 1 F-3)

## 1. Feature Overview
- **Feature Name**: Mock Speech Infrastructure  
- **ID**: F-3  
- **Summary**:  
  Provide offline “fake” implementations of the LLM and TTS services so that the app can exercise the full turn flow without making real network calls. This enables UAT-mode testing of end-to-end session logic (dequeue, prompt construction, count tracking, UI updates) using canned responses.  
- **Scope**:  
  - **In-scope**:  
    - `FakeLlmService` returns a predefined JSON payload (`sample_turn_1.json`).  
    - `FakeTtsService` is a no-op (succeeds immediately).  
    - DI switch (`BuildConfig.UAT_MODE`) to inject fakes instead of live services.  
    - Integration with ViewModel/UI so that “Start Session” → fake LLM → UI updates + count mutations.  
  - **Out-of-scope**:  
    - Any real OpenAI or network calls.  
    - STT remains unimplemented or real; user audio not processed.  
    - Live TTS playback.  
- **Owner**: Tech Lead  

## 2. Architecture Notes
- **Module(s) affected**:  
  - `:speech` (fake implementations)  
  - `:app` (DI wiring, MainViewModel injection)  
- **Entry points & interactions**:  
  - **DI**: When `BuildConfig.UAT_MODE=true`, Hilt provides `FakeLlmService` and `FakeTtsService`; otherwise, live implementations.  
  - **ViewModel**: `MainViewModel.startSession()` → `StartSessionUseCase` → `GenerateDialogueUseCase.generateDialogue()` → `LlmService.generateDialogue()`. Under UAT, this resolves to the fake service. After receiving text, the ViewModel calls `TtsService.speak(...)` (no-op in fake).  
- **Relevant interfaces / classes**:  
  - `com.example.domain.LlmService`  
    - Live: `OpenAiLlmServiceImpl`  
    - Fake: `FakeLlmService` (reads `sample_turn_1.json`)  
  - `com.example.domain.TtsService`  
    - Live: `OpenAiTtsServiceImpl`  
    - Fake: `FakeTtsService` (immediate success)  
  - **DI Test Module**:  
    - `app/src/di-test/TestAppModule.kt` binds fakes for instrumentation tests.

## 3. NA
## 4. NA

## 5. API / Backend
- **API Mocks or Fakes Needed**  
  Fake implementations are bound in the test DI module so that instrumentation tests use mocks instead of real services:

  ```kotlin
  // app/src/di-test/java/com/example/lango_mvp_android/di/TestAppModule.kt
  @Module
  @TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [RepositoryModule::class]
  )
  object TestAppModule {
  
      @Provides @Singleton
      fun provideLlmService(): LlmService =
        mockk(relaxed = true)
  
      @Provides @Singleton
      fun provideTtsService(): TtsService =
        mockk(relaxed = true)
  }
````

These mocks satisfy the **C-1** and **C-2** scenarios in TDD.md:

* `FakeLlmService.generateDialogue()` returns the canned JSON from `sample_turn_1.json`.
* `FakeTtsService.speak()` is a no-op (always succeeds).&#x20;

## 6. NA

## 7. Feature Flags / Rollout Plan

* **Flag Name**: `BuildConfig.UAT_MODE`
* **Default State**: `true` in debug/test builds, `false` in production.
* **Gating Conditions**: When `UAT_MODE=true`, Hilt binds `FakeLlmService` and `FakeTtsService`; otherwise, binds live implementations.
* **Implementation**:

  ```kotlin
  // build.gradle.kts
  android {
    defaultConfig {
      buildConfigField("Boolean", "UAT_MODE", "true")
    }
    // ...
  }
  ```
## Test Data

- **sample_turn_1.json**  
  Location: `speech/src/test/resources/sample_turn_1.json`  
  Contains a single turn of mock LLM output used by `FakeLlmService.generateDialogue()`.


## 8. Test Specifications – Canonical Format

| ID      | Scenario                                                  | Pre‑Conditions                                          | Steps                                            | Expected                                                                                         | Implementation                                                          | Dependencies                                                    |
|---------|-----------------------------------------------------------|---------------------------------------------------------|--------------------------------------------------|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|----------------------------------------------------------------|
| **C-1** | Fake LLM returns canned JSON                             | `BuildConfig.UAT_MODE=true`                             | 1. Launch app with `UAT_MODE=true`<br>2. Retrieve `LlmService` from Hilt<br>3. Call `generateDialogue()` | Returns exact contents of `sample_turn_1.json` located at `speech/src/test/resources/sample_turn_1.json` :contentReference[oaicite:0]{index=0} | `speech/src/test/FakeLlmServiceTest.kt` | Ktor-Mock |
| **C-2** | Fake TTS is no-op                                         | `BuildConfig.UAT_MODE=true`                             | 1. Launch app with `UAT_MODE=true`<br>2. Retrieve `TtsService` from Hilt<br>3. Call `speak("any text")` | Returns `Result.success` immediately with no side-effects :contentReference[oaicite:0]{index=0} | `speech/src/test/FakeTtsServiceTest.kt`                             | MockK                                                          |
| **DI-1** | Hilt toggles mock vs live                                 | `BuildConfig.UAT_MODE=true` _and_ `false`               | 1. Launch app with `UAT_MODE=true`<br>2. Retrieve `LlmService` and `TtsService` from Hilt component<br>3. Assert instance types: `FakeLlmService`, `FakeTtsService`<br>4. Repeat with `UAT_MODE=false` and assert `OpenAiLlmServiceImpl`, `OpenAiTtsServiceImpl`<br>5. Cast to fakes, call `generateDialogue()` and `speak("test")` to verify canned JSON and immediate success | - **DI-1.a**: `assertThat(llmService).isInstanceOf(FakeLlmService::class.java)`<br>- **DI-1.b**: `assertThat(ttsService).isInstanceOf(FakeTtsService::class.java)`<br>- **DI-1.c**: `assertThat((llmService as FakeLlmService).generateDialogue()).isEqualTo(File("speech/src/test/resources/sample_turn_1.json").readText())`<br>  `assertThat((ttsService as FakeTtsService).speak("test")).isSuccess()` :contentReference[oaicite:1]{index=1} | `app/src/di-test/TestAppModule.kt` + `app/src/androidTest/HiltSwitchInstrumentationTest.kt` | Hilt-Testing |
                |
| **PS-1**| FIRST_TIME prompt matches golden                          | Fixed UUID & sample queues                              | `InitialPromptBuilderImpl.build(queues)`        | Prompt identical to `initial_first_time_prompt.txt`                                              | `domain/src/test/InitialPromptBuilderTest.kt`                           | Snapshot fixtures                                              |
| **PS-2**| NORMAL_DIALOGUE prompt matches golden                     | Standard learned_pool & new_item counts                 | `GenerateDialogueUseCase.generatePrompt(queues)` | Prompt identical to `normal_dialogue_prompt.txt`                                                 | `domain/src/test/GenerateDialogueUseCaseTest.kt`                        | Snapshot fixtures                                              |
| **PS-3**| PLEASE_REPEAT prompt matches golden                       | Simulated learner misuse                                | Invoke builder with misuse fixture              | Prompt identical to `please_repeat_prompt.txt`                                                   | `domain/src/test/PleaseRepeatPromptTest.kt`                            | Snapshot fixtures                                              |
| **VM-1**| ViewModel emits Loading→Waiting→CoachSpeaking with text   | Fakes bound; queues fixture                             | `MainViewModel.startSession()`                   | `uiState` sequence includes `Loading`, then `Waiting`, then `CoachSpeaking(text=…)`                | `app/src/test/MainViewModelTest.kt`                                     | Turbine, Coroutines-Test, MockK                               |

---

## 9. Testing Strategy Summary

### 🎯 Test Strategy Overview
- **Unit tests**  
  - **C-*** (speech fakes) to validate mock services.  
  - **DI-1** for dependency injection switching.  
  - **PS-*** (snapshot tests) for prompt content across three scenarios.  
  - **VM-1** for core ViewModel state transitions and final coach text.
- **Integration tests**  
  - Use existing end-to-end tests; no new INT-* tests this milestone.
- **Snapshot / Golden file tests**  
  - Three PS-* tests under `domain/src/test` with resource files.
- **Manual testing**  
  - Not required; behavior verified in automated tests.

---

## 10. Test Data & Fixtures
- **JSON fixtures**  
  - `sample_turn_1.json` in `speech/src/test/resources/`
- **Prompt fixtures** (`domain/src/test/resources/`):  
  - `initial_first_time_prompt.txt`  
  - `normal_dialogue_prompt.txt`  
  - `please_repeat_prompt.txt`
- **Queues fixtures**  
  - Shared helper `TestFixtures.queuesFixture(...)`  
- **UUID normalization**  
  - Use `00000000-0000-0000-0000-000000000000` in snapshots.

---

## 11. Tooling & Coverage Requirements
- **Libraries**: JUnit5, MockK, kotlinx-coroutines-test, Turbine, Ktor-Mock, Hilt-Testing, kotlinx-serialization  
- **Coverage**: Maintain ≥80% coverage in `:domain` and `:speech` modules  
- **CI**: All tests must pass; snapshot diffs reported via CI tools.

---

## 12. Manual QA Requirements
- **Platforms/Devices**: N/A  
- **RTL/Localization/Accessibility**: N/A  
- **Performance/Animation**: N/A  
- **Known Limitations**: Mock timing may differ from real-world.

---

## 13. Open Issues & Risks
- No negative-path tests for real service failures.  
- Potential duplication in prompt-builder logic.  
- VM test timing may require Turbine timeout tuning.

---

## 14. Final Deliverables Checklist
- [ ] All test cases defined in canonical table format  
- [ ] Test IDs assigned with proper tag prefixes (C-*, DI-*, PS-*, VM-*)  
- [ ] Snapshot tests for PS-1, PS-2, PS-3 in place  
- [ ] Coverage requirements met (≥80% in :domain and :speech)  
- [ ] Fixtures committed under `resources` directories  
