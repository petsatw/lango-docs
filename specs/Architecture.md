\#\#\# Recommended Architecture for Lango MVP Android App

Based on the Technical Design Document (TDD) and the provided PROMPT PDF, I recommend an \*\*MVVM (Model-View-ViewModel) architecture\*\* with modular components, following clean architecture principles for separation of concerns. This aligns well with Android best practices, the TDD emphasis (e.g., unit testable layers, mocks for APIs and files), and the MVP's simplicity (minimal UI, local persistence, API integrations). MVVM promotes testability by isolating UI from business logic and data, making it easy to write the specified unit and integration tests.

The architecture is designed to be:  
\- \*\*Modular\*\*: Separate modules for data (JSON handling), speech (STT/TTS/LLM), and core logic (session orchestration per PROMPT rules).  
\- \*\*Extensible\*\*: Language-agnostic (e.g., configurable via JSON), with no hard-coded German assumptions.  
\- \*\*Testable\*\*: Use dependency injection (DI) for mocking APIs, files, and audio in tests.  
\- \*\*Efficient\*\*: Handle async operations (API calls, audio) with Kotlin Coroutines and Flows.  
\- \*\*Minimal\*\*: Focus on phase zero requirements—no cloud, multi-user, or complex features.

I'll describe the high-level structure, key components, data flow, and how it supports TDD. Use Kotlin as the language, Android Jetpack libraries (e.g., ViewModel, LiveData/Flow, Navigation if needed), and Hilt for DI.

\#\#\#\# 1\. High-Level Architecture Overview  
The app is structured into layers:  
\- \*\*Presentation Layer (View \+ ViewModel)\*\*: Handles UI (buttons, indicators, text display) and user interactions (e.g., speak button tap starts recording).  
\- \*\*Domain Layer (Use Cases \+ Entities)\*\*: Core business logic (e.g., mastery checks, count tracking, dialogue generation per PROMPT orchestration).  
\- \*\*Data Layer (Repositories \+ Sources)\*\*: JSON persistence and in-memory state.  
\- \*\*Infrastructure Layer (Modules)\*\*: Speech (STT/TTS/LLM via OpenAI APIs), utilities (e.g., logging).

Modules can be Android modules (e.g., \`:data\`, \`:speech\`, \`:domain\`, \`:app\`) for build separation and reusability.

\`\`\`  
\[User Interaction (UI: Activity/Fragment)\]  
          ↑↓ (Events/States via LiveData/Flow)  
\[ViewModel (Orchestrates Use Cases, Holds UI State)\]  
          ↑↓ (Requests/Results)  
\[Domain (Use Cases, Entities: Session Logic, Tracking)\]  
          ↑↓ (Data Access)  
\[Data Layer (Repository: JSON Read/Write, In-Memory Cache)\]  
          ↑↓ (API Calls)  
\[Infrastructure (Speech Module: STT/TTS/LLM, Logging)\]  
\`\`\`

\#\#\#\# 2\. Key Components and Responsibilities

\- \*\*Entities (Domain Layer)\*\*:  
  \- Simple data classes for queues and items, mirroring JSON structures.  
  \- \`LearningItem\`: id, text (token), usage\_count, presentation\_count (renamed from presentation\_count in TDD for consistency with PROMPT), category, subcategory.  
  \- \`NewQueue\`: List\<LearningItem\> (from core\_blocks.json, initial usage/presentation \= 0).  
  \- \`LearnedPool\`: List\<LearningItem\> (from learned\_queue.json, with existing counts).  
  \- These are immutable where possible for thread safety.

\- \*\*Repositories (Data Layer)\*\*:  
  \- \`LearningRepository\`: Abstracts data access.  
    \- Methods: \`loadQueues(filePaths: Pair\<String?, String?\>)\`: Loads from predefined assets or picked files; parses JSON into entities. Handles errors (e.g., invalid JSON → empty lists \+ error state).  
    \- \`saveQueues(newQueue: List\<LearningItem\>, learnedPool: List\<LearningItem\>)\`: Writes to local JSON files at session end/close.  
    \- \`getCurrentState()\`: Returns in-memory queues (cached after load).  
    \- Implementation: Use Kotlinx-Serialization-Json for JSON parsing. Predefined files in \`/assets\`. File picker via Android's \`ActivityResultLauncher\`.  
  \- Testable: Mock file I/O for unit tests (e.g., Unit Test 1 in TDD).

\- \*\*Use Cases (Domain Layer)\*\*:  
  \- Encapsulate PROMPT orchestration logic (Initialization, Main Loop, Dialogue Stage).  
  \- \`StartSessionUseCase\`: Loads queues, dequeues first new\_target if needed.  
  \- \`ProcessTurnUseCase\`: Given user response (STT text), updates counts (usage++ if new\_target detected; presentation++ for coach-exposed items repeated by user). Checks mastery (usage \> 3 → move to learned\_pool, dequeue next).  
  \- \`GenerateDialogueUseCase\`: Builds prompt for LLM (ChatGPT) per PROMPT rules: Introduce new\_target (if presentation=0: say alone, simple 5-year-old explanation in English, example with learned\_pool items); otherwise, natural dialogue using only learned\_pool \+ new\_target, bias to low-count items (e.g., weight selection inversely by counts).  
    \- Prompt construction: "You are Lango... \[insert PROMPT principles\] ... Current new\_target: \[text\]. Learned pool: \[list with counts\]. Generate a short coach response in German that prompts the user to use the new\_target."  
    \- Handles parametric phrases (e.g., fill with learned items).  
  \- \`EndSessionUseCase\`: Saves state, shows congrats if queue empty.  
  \- These are suspend functions for async, injectable for testing.

\- \*\*Speech Module (Infrastructure Layer)\*\*:  
  \- Wrappers for OpenAI APIs (use official OpenAI Kotlin SDK or Retrofit).  
  \- \`SttService\`: Records audio (Android MediaRecorder), sends to Whisper API, returns text. Handles errors (e.g., low confidence → bilingual repeat prompt).  
  \- \`TtsService\`: Sends text to TTS API, plays audio (MediaPlayer), returns success. Config: German voice (standard if Linz not available).  
  \- \`LlmService\`: Sends prompt to ChatGPT, returns generated text.  
  \- All async (Coroutines). Mockable for tests (e.g., Unit Tests 2-4 in TDD).

\- \*\*ViewModel (Presentation Layer)\*\*:  
  \- \`MainViewModel\`: Manages session state (e.g., current new\_target, queues via repository).  
    \- Exposes UI states via StateFlow: \`uiState\` (e.g., loading, listening, waiting, error, coachText).  
    \- Handles events: onResumeSession() → invoke StartSessionUseCase.  
    \- onSpeak() → start recording, invoke SttService → ProcessTurnUseCase → GenerateDialogueUseCase → TtsService (play \+ display text).  
    \- onEndSession() → invoke EndSessionUseCase.  
    \- Bias logic: When generating, select learned items with lowest counts.  
    \- Logs: Maintain in-memory log of coach/user texts for verification.

\- \*\*View (Presentation Layer)\*\*:  
  \- Single \`MainActivity\` with Fragment if needed (minimal: buttons for Resume/End/Speak, menu for file picker).  
  \- UI elements: TextView for coach text, ProgressBar for waiting/listening indicators.  
  \- Observe ViewModel states: Update UI (e.g., show error toast on API fail).  
  \- Permissions: Request microphone on launch.  
  \- Interruptions: Override onDestroy/onPause for save if phone call (use lifecycle awareness).

\- \*\*Dependency Injection (Cross-Layer)\*\*:  
  \- Use Hilt: Inject repositories, services, use cases into ViewModel.  
  \- Enables mocking (e.g., fake API responses in tests).

\#\#\#\# 3\. Data Flow in a Typical Session Turn  
1\. User taps "Resume Session" → ViewModel loads queues via repository.  
2\. Dequeue new\_target if needed (per PROMPT Initialization).  
3\. Generate initial coach response (LLM prompt for introduction if new).  
4\. TTS plays \+ displays text (presentation++ for exposed items).  
5\. User taps "Speak" → Record audio → STT → Process user text (update counts, check mastery).  
6\. If mastery, move item; generate next response → Loop.  
7\. On end/empty queue: Save JSON, show congrats.

Async handling: Use Coroutines in ViewModel (e.g., viewModelScope.launch { ... }).


#### Data Layer / File I/O Strategy

+ ### Atomic Write & Robolectric Fallback
+ To ensure robust file persistence, `LearningRepositoryImpl.saveQueues()` uses an `atomicWrite` helper:
+ 
+ ```kotlin
+ try {
+     Files.move(
+         tempFile.toPath(),
+         targetFile.toPath(),
+         StandardCopyOption.ATOMIC_MOVE,
+         StandardCopyOption.REPLACE_EXISTING
+     )
+ } catch (e: AtomicMoveNotSupportedException) {
+     // Fallback for Robolectric & non-atomic filesystems
+     Files.move(
+         tempFile.toPath(),
+         targetFile.toPath(),
+         StandardCopyOption.REPLACE_EXISTING
+     )
+ }
+ ```
+ 
+ **Note:** Robolectric’s in‑memory FS (especially on Windows) may still throw `AccessDeniedException`. Tests FS‑2/FS‑3 are disabled in that environment; see TDD.md for details.


\#\#\#\# 4\. How This Supports TDD  
\- \*\*Unit Tests\*\*: Isolate layers.  
  \- Data: Mock file system, test parsing (TDD Unit Test 1).  
  \- Domain: Test use cases with mock entities (e.g., mastery transition).  
  \- Infrastructure: Mock APIs (e.g., return fixed text/audio for Unit Tests 2-4).  
  \- ViewModel: Test state changes with mocked use cases/repositories.  
\- \*\*Integration Tests\*\*: Combine layers (e.g., TDD End-to-End: Real APIs for full sequence, or mocked for speed).  
\- \*\*UI Tests\*\*: Use Espresso/Robolectric for button taps, indicator visibility.  
\- \*\*Mocks\*\*: Use Mockito for APIs/files; run tests with no internet via mocks.  
\- Development Flow: Write tests first (per TDD), implement minimal code to pass, refactor.

\#\#\#\# 5\. Potential Trade-offs and Extensions  
\- \*\*Pros\*\*: Clean, testable, aligns with Android (MVVM is Jetpack-recommended). Handles PROMPT's real-time tracking in-memory.  
\- \*\*Cons\*\*: Slight overhead for small MVP, but pays off for extensibility (e.g., add languages by swapping JSON/prompts).  
\- \*\*Extensions\*\*: Future phases can add cloud repositories, multi-language configs, or replace OpenAI with local models.  
\- \*\*Tools/Libs\*\*: Android Studio, Kotlin, Hilt, Coroutines, Kotlinx-Serialization-Json, OpenAI SDK, MediaRecorder/MediaPlayer.

This architecture ensures the app is buildable iteratively, starting with the core sequence test in the TDD. If you need code snippets, diagrams, or refinements, let me know\!