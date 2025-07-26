# Technical Design & Test Plan (TDD) – Lango MVP


## 1 Architecture in One Minute
*MVVM + Clean* split into 5 Gradle modules:

| Layer (module) | Key runtime classes | Test entry points |
| --- | --- | --- |
| **Data (:data)** | `LearningRepositoryImpl` | FS‑* |
| **Domain (:domain)** | *Use‑cases* `StartSessionUseCase`, `ProcessTurnUseCase`, `GenerateDialogueUseCase`, etc. | B‑*, PS‑* |
| **Speech (:speech)** | `LlmServiceImpl`, `TtsServiceImpl`, `SttServiceImpl` | C‑* |
| **Presentation (:app)** | `MainViewModel` | VM‑* |
| **UI (:app)** | `MainActivity` & Espresso hooks | UI‑* |
| **Cross‑module** | Hilt DI bindings | DI‑* |
| **E2E** | full stack (fakes vs live) | INT‑* |

All tests are **white‑box** within their layer unless tagged `INT‑*`.

---

## 2 Taxonomy & Naming Convention  *(unchanged)*

| Tag | Module | Purpose | Gradle Task |
| --- | --- | --- | --- |
| **FS‑n** | :data | File‑system bootstrap & persistence | `:data:test` |
| **B‑n** | :domain | Pure business rules | `:domain:test` |
| **PS‑n** | :domain | Prompt snapshot / golden files | `:domain:test` |
| **C‑n** | :speech | Offline speech fakes | `:speech:test` |
| **DI‑n** | :app (+`di-test`) | DI binding switches | `connectedAndroidTest` |
| **VM‑n** | :app | View‑model state machine | `:app:test` |
| **UI‑n** | :app | Espresso / Robolectric UI flows | `connectedAndroidTest` |
| **INT‑n** | :app | End‑to‑end with fakes or live | `connectedAndroidTest` |

File naming: `<Tag><number>_<behaviour>.kt` (e.g., `FS1_ColdStartCopyTest.kt`).

---

## 3 Detailed Test Specifications   
*Each block follows the canonical FS table (Scenario / Pre‑Conditions / Steps / Expected / Implementation / Dependencies).*  

### 3.1 Data‑Layer – File‑System (FS‑*)

| ID | Scenario | Pre‑Conditions | Steps | Expected | **Implementation** | **Dependencies** |
| --- | --- | --- | --- | --- | --- | --- |
| **FS‑1** | Cold‑start copy & load | `queues/` absent | Call `LearningRepositoryImpl.loadQueues()` | Returns `Result.success`, asset JSON copied, queue sizes match assets | `data/src/test/kotlin/.../LearningRepositoryImplTest.kt` `@Test fun FS_1_ColdStart_load()` | Robolectric 4.12, Jimfs (`com.google.jimfs:jimfs`), `Json` (kotlinx), Truth |
| **FS‑2** | Persist & reload | In‑mem queues mutated | `saveQueues()` → `loadQueues()` | Mutations persisted | same class method `FS_2_Persist_reload` | Jimfs, Coroutines‑Test |
| **FS‑3** | Concurrent load/save | 2 coroutines R/W | `async {{load}}` + `async {{save}}` | No race / exception | same class `FS_3_Concurrent_io` | Kotlinx Coroutines‑Test |
| **FS‑4** | Malformed JSON recovery | Corrupt `new_queue.json` | `loadQueues()` | Recovers defaults | same class `FS_4_Malformed_json` | Robolectric |
| **FS‑5** | Missing files | Delete one/both files | `loadQueues()` | Recreates defaults | same class `FS_5_Missing_files` | Robolectric |
| **FS‑6** | Reset counts persisted | After dequeue reset | Reload file | Counts==0 | same class `FS_6_Reset_counts` | Jimfs |

### 3.2 Domain‑Layer – Business Rules (B‑*)

| ID | Scenario | Pre‑Conditions | Steps | Expected | Implementation | Dependencies |
| --- | --- | --- | --- | --- | --- | --- |
| **B‑1** | *StartSession resets & dequeues* | Repository returns queues | `StartSessionUseCase.startSession()` | First `new_item` counts reset; queues persisted | `domain/src/test/.../StartSessionUseCaseTest.kt` «startSession returns queues…» | Mockito‑Kotlin, Coroutines‑Test |
| **B‑2** | *ProcessTurn mastery transition* | `usageCount==2` | `ProcessTurnUseCase.processTurn()` with user uses token | Moves item to `learnedPool`, `usageCount==3`, queues saved | `ProcessTurnUseCaseTest.kt` «processTurn moves newTarget…» | Mockito‑Kotlin |
| **B‑3** | *Usage not incremented when absent* | Any turn | processTurn(no token) | `usageCount` unchanged | same file | — |

### 3.3 Domain – Prompt Snapshot (PS‑*)

| ID | Scenario | Pre‑Conditions | Steps | Expected | Implementation | Dependencies |
| --- | --- | --- | --- | --- | --- | --- |
| **PS‑1** | Initial prompt JSON matches golden | Fixed UUID `000…` & fixture queues | `InitialPromptBuilderImpl.build()` | JSON identical to `initial_prompt.txt` | `InitialPromptBuilderTest.kt` | Snapshot file `domain/src/test/resources/initial_prompt.txt`, kotlinx‑serialization |
| **PS‑2** | Dialogue prompt fuzzer (100 runs) | Learned‑pool randomised | build→regex validate | No vocabulary outside new_target + learned_pool | *TODO – not yet in code* | Kotest‑Property |

### 3.4 Speech Infrastructure (C‑*)

| ID | Scenario | Pre‑Con | Steps | Expected | Impl | Deps |
| --- | --- | --- | --- | --- | --- | --- |
| **C‑1** | Fake LLM returns canned JSON | — | `FakeLlmService.generateDialogue()` | Returns `sample_turn_1.json` string | `speech/src/test/FakeLlmServiceTest.kt` | Ktor‑Mock |
| **C‑2** | Fake TTS is no‑op | — | `FakeTtsService.speak()` | Returns `Result.success` | `FakeTtsServiceTest.kt` | MockK |

### 3.5 DI Switch (DI‑*)

| ID | Scenario | Pre‑Con | Steps | Expected | Impl | Deps |
| --- | --- | --- | --- | --- | --- | --- |
| **DI‑1** | Hilt toggles mock vs live | Set `BuildConfig.UAT_MODE=true/false` | Launch instrumentation test | Correct impl injected (`FakeLlmService` vs `OpenAiLlmService`) | `app/src/di-test/.../TestAppModule.kt` + `HiltSwitchInstrumentationTest.kt` | Hilt‑Testing |

### 3.6 Presentation (VM‑*)

| ID | Scenario | Pre‑Con | Steps | Expected | Impl | Deps |
| --- | --- | --- | --- | --- | --- | --- |
| **VM‑1** | uiState sequence for first turn | Mock use‑cases happy path | `MainViewModel.startSession()` | Idle→Loading→CoachSpeaking | `MainViewModelTest.kt` «startSession updates uiState…» | Turbine, MockK |

### 3.7 UI (UI‑*)

| ID | Scenario | Pre‑Con | Steps | Expected | Impl | Deps |
| --- | --- | --- | --- | --- | --- | --- |
| **UI‑1** | Espresso flow start→end | App launched in UAT | Tap *Start Session* then *End Session* | TextView updates, file saved | `SessionIntegrationTest.kt` (UI part) | Espresso‑Core, Hilt |

### 3.8 Integration (INT‑*)

| ID | Scenario | Pre‑Con | Steps | Expected | Impl | Deps |
| --- | --- | --- | --- | --- | --- | --- |
| **INT‑1** | Core happy path with fakes | Queues fixture (1 new,1 learned) | Full turn until congrats | JSON deltas verified | `EndToEndCoreSequenceIntegrationTest.kt` | Robolectric, Hilt |

---

## 4 Dependency Matrix *(supersedes §5 of v2)*

| Test Tag | Core runtime under test | Major test libraries | Fixtures |
| --- | --- | --- | --- |
| FS | `LearningRepositoryImpl` | Robolectric, Jimfs, Coroutines‑Test | `data/src/test/assets/queues/*.json` |
| B | Use‑case classes | Mockito‑Kotlin, Truth | `TestFixtures.kt` |
| PS | `InitialPromptBuilderImpl` | kotlinx‑serialization, Kotest‑Snapshot | `initial_prompt.txt` |
| C | Speech fake impls | Ktor‑Mock, MockK | `sample_turn_1.json` |
| DI | Hilt bindings | Hilt‑Testing, MockK | `di-test` module |
| VM | `MainViewModel` | Turbine, MockK | — |
| UI | Activity XML / Robolectric | Espresso, Idling Rules | Layout TBD |
| INT | full app | Robolectric, MockK | all above |

---

## 5 Execution & Gate‑keeping

*Unchanged from v2 except task names updated for new tests.*  
CI fails if **any** test here goes red or coverage in :data / :domain dips below 80 %.

---

## 6 Open Issues (authoritative)

The active issues list lives in **FR2QA_Build.md § “Known Gaps & Risks”** and is *not duplicated here* to avoid divergence. Key headlines (as of 2025-07-23):

| ID | Area | Risk / Gap | Owner | Target fix |
|----|------|-----------|-------|-----------|
| **OI-1** | Speech-LLM | Prompt length occasionally exceeds OpenAI 128 k-token cap on long learned pools | NLP Guild | Milestone 2 |
| **OI-2** | UI | Jetpack Compose migration blocked by Hilt 2.53 alpha bug | UI Team | Await Hilt 2.54 |
| **OI-3** | Test flakiness | `C-1 FakeLlmServiceTest` sporadically fails on macOS ARM runners | QA | Replace blocking I/O with coroutine fake |
| **OI-4** | Coverage | Domain layer stuck at 78 % vs 80 % gate when `kotlinc` updates | Architecture Guild | Add PS-2 fuzz & more edge-case B-tests |

*(For the canonical list, always consult **FR2QA_Build.md**.)*

---

## 7 Future-Proofing: Ideal Structure → Milestones

1. **Snapshot-first culture** – Golden-file tests (PS-*) for every prompt variant; integrate Kotest-snapshot.
2. **Isolate Speech layer** – Replace inline fakes with generated WireMock stubs for contract tests against live OpenAI nightly.
3. **Move UI to Compose** – Espresso → Compose-UITest; maintain tag continuity (`UI-*`) during transition.
4. **Parameterise queues** – Introduce fixture builder (`buildQueue(new=*, learned=*)`) to cut copy-paste.
5. **Milestone-gated CI lanes**

| Phase | Action | Owner | “Done when…” |
|-------|--------|-------|--------------|
| **FP1** | Rename existing tests to `<Tag><n>_…` and relocate to correct module folders | Dev Team | Gradle build green |
| **FP2** | Add Implementation column info to FS-* table (completed) | Arch Guild | This doc merged |
| **FP3** | Flesh out missing C-2, UI-1 IDs in code | QA | PR merged |
| **FP4** | Implement PS-2 fuzz test & snapshot harness | QA | Coverage ≥ 80 % |
| **FP5** | Introduce WireMock stub server for Speech live smoke tests | DevOps | Nightly job passes |
| **FP6** | Compose UI refactor; migrate Espresso → ComposeTest | UI Team | UI-tags updated |

---

*Last updated 2025-07-23*


