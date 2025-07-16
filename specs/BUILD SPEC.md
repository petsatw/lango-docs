\# Technical Specification for Lango MVP Android App

\#\# Version 1.0  
\*\*Date:\*\* July 15, 2025    
\*\*Author:\*\* Grok (based on recommended architecture, TDD, and PROMPT logic)    
\*\*Purpose:\*\* This document formalizes the recommended MVVM architecture into a detailed technical specification for the Lango MVP Android app. It defines the system's structure, components, data flows, interfaces, and implementation guidelines to ensure alignment with phase-zero requirements. The spec emphasizes Test-Driven Development (TDD), modularity for extensibility (e.g., multi-language support), and adherence to the PROMPT's orchestration logic (e.g., one-new-thing introduction, real-time tracking). All elements are derived from the provided TDD and PROMPT PDFs, focusing on a voice-only German coaching session for beginners, with local persistence and OpenAI integrations.

\#\# 1\. Introduction and Scope  
\#\#\# Overview  
Lango is a voice-based language coaching app for beginners, starting with German. The MVP (phase zero) is for personal use: single-user, local JSON storage, no cloud sync or advanced features. It runs continuous coaching sessions using dialogues to introduce and master one new language item (new\_target) at a time, tracking presentation and usage counts. Sessions are audio-only (user speaks/responses via STT/TTS), with minimal UI for control.

Key behaviors:  
\- Load new\_queue (core\_blocks.json) and learned\_pool (learned\_queue.json).  
\- Introduce new\_target simply (German pronunciation, 5-year-old English explanation, example).  
\- Generate natural dialogues using only learned\_pool \+ new\_target, biasing toward low-count items.  
\- Update counts: presentation++ on coach exposure; usage++ on correct user response.  
\- Master when usage \> 3: Move to learned\_pool, dequeue next.  
\- End when queue empty: Show congrats.

Scope Limitations:  
\- Android-only, minimal UI (buttons: Resume/End/Speak; indicators: listening/waiting; text display for coach).  
\- Internet-required for OpenAI (ChatGPT for dialogues, Whisper STT, TTS).  
\- Local JSON persistence; no user profiles, analytics, or monetization.  
\- Extensible: Language-agnostic (configurable via JSON/prompts).

Success Criteria: Functional turns, accurate mastery transitions, verifiable logs/counts (per TDD metrics).

\#\#\# Assumptions and Constraints  
\- Android API level: 26+ (Oreo) for modern features like Coroutines.  
\- Libraries: Kotlin, Jetpack (ViewModel, Flows), Hilt DI, Gson for JSON, OpenAI SDK.  
\- No offline mode; handle errors gracefully.  
\- Audio: Microphone permission required; no pronunciation feedback.

\#\# 2\. Architecture Overview  
The app adopts \*\*MVVM (Model-View-ViewModel)\*\* with clean architecture layers for separation of concerns:  
\- \*\*Presentation Layer\*\*: UI and ViewModel – Handles user interactions and state.  
\- \*\*Domain Layer\*\*: Business logic – Orchestration per PROMPT (e.g., mastery checks).  
\- \*\*Data Layer\*\*: Persistence – JSON handling and in-memory cache.  
\- \*\*Infrastructure Layer\*\*: External services – Speech APIs (STT/TTS/LLM).

Modular build: Separate Gradle modules (:app, :domain, :data, :speech) for testability.  
\- Dependency Injection: Hilt for injecting dependencies (e.g., mock APIs in tests).  
\- Async: Kotlin Coroutines and Flows for API calls/audio.  
\- State Management: In-memory during session; persist to JSON on end/close.

High-Level Diagram (Textual Representation):  
\`\`\`  
\[UI: MainActivity\] \<-\> \[ViewModel: MainViewModel\] \<-\> \[Use Cases: Domain\]  
                                           ↑↓  
\[Repository: LearningRepository\] \<-\> \[Speech Services: STT/TTS/LLM\]  
\`\`\`

\#\# 3\. Data Models (Entities)  
Defined in :domain module as Kotlin data classes.

\- \*\*LearningItem\*\*:  
  \`\`\`kotlin  
  data class LearningItem(  
      val id: String,  
      val token: String,  // Token/phrase  
      val category: String? \= null,  
      val subcategory: String? \= null,  
      var usageCount: Int \= 0,  
      var presentationCount: Int \= 0,  
      var isLearned: Boolean \= false  
  )  
  \`\`\`

\- \*\*Queues\*\*:  
  \- \`newQueue: MutableList\<LearningItem\>\` – From core\_blocks.json; FIFO dequeue.  
  \- \`learnedPool: MutableList\<LearningItem\>\` – From learned\_queue.json; bias selection by min(usageCount \+ presentationCount).

JSON Mapping: Use Gson for serialization/deserialization, matching provided structures (e.g., "token" → text).

\#\# 4\. Detailed Components

\#\#\# 4.1 Presentation Layer  
\- \*\*MainActivity\*\* (Single Activity):  
  \- Layout: XML with TextView (coach text), Button (Resume Session, End Session, Speak), ProgressBar (waiting/listening indicators), menu for file picker.  
  \- Lifecycle: Request microphone permission onCreate; observe ViewModel states via Flows.  
  \- Events: Button clicks trigger ViewModel functions; handle interruptions (e.g., onDestroy → save).  
  \- Error Handling: Toast for API/permission errors.

\- \*\*MainViewModel\*\*:  
  \- Dependencies: Injected LearningRepository, Use Cases, Speech Services.  
  \- State: \`uiState: StateFlow\<UiState\>\` where UiState is a sealed class (Idle, Loading, Listening, Waiting, Error(message), CoachSpeaking(text), Congrats).  
  \- Functions:  
    \- \`resumeSession()\`: Invoke StartSessionUseCase; generate initial dialogue.  
    \- \`startSpeaking()\`: Start audio recording; invoke SttService → processResponse(text).  
    \- \`processResponse(userText: String)\`: Invoke ProcessTurnUseCase (update counts, check mastery); if mastered, dequeue next; generate next dialogue.  
    \- \`endSession()\`: Invoke EndSessionUseCase; save state.  
    \- Logging: Maintain \`sessionLog: MutableList\<String\>\` (e.g., "Coach: \[text\]\\nUser: \[text\]") for verification.  
  \- Bias Logic: When prompting LLM, include learnedPool with counts; instruct to prefer low-count items.

\#\#\# 4.2 Domain Layer  
\- \*\*Use Cases\*\* (Suspend functions for async):  
  \- \*\*StartSessionUseCase\*\*:  
    \- Load queues via repository.  
    \- If newQueue not empty, dequeue first as newTarget; reset counts to 0\.  
    \- Return initial state.

  \- \*\*GenerateDialogueUseCase\*\*:  
    \- Input: Current newTarget, learnedPool.  
    \- Build LLM prompt per PROMPT:  
      \- If presentationCount \== 0: "Say \[newTarget\] alone. Explain meaning in simple English (5-year-old level). Give one example using learned items."  
      \- Else: "Generate natural German dialogue using only learned\_pool \+ new\_target. Prompt user to use new\_target. Bias to items with low counts."  
    \- Invoke LlmService; return generated text.  
    \- Update: presentation++ for used learned items \+ newTarget.

  \- \*\*ProcessTurnUseCase\*\*:  
    \- Input: User text (from STT).  
    \- Detect: If contains newTarget text (case-insensitive match), usage++.  
    \- For each learned item in coach's last prompt repeated in user text, usage++.  
    \- Mastery Check: If newTarget.usageCount \> 3, move to learnedPool (set isLearned=true); dequeue next newTarget.  
    \- Handle parametric: Loose matching (e.g., "Ich heiße \[any\]").

  \- \*\*EndSessionUseCase\*\*:  
    \- If newQueue empty, set UiState.Congrats.  
    \- Save queues via repository.

\#\#\# 4.3 Data Layer  
\- \*\*LearningRepositoryImpl\*\*:  
  \- Dependencies: Gson, Android Context (for resources/files).  
  \- \`loadQueues(optionalPaths: Pair\<String?, String?\>)\`: If paths null, load from resources ("core\_blocks.json", "learned\_queue.json"); else from picked files. Parse to entities.  
  \- \`saveQueues(...)\`: Serialize to local files (e.g., app's internal storage); overwrite.  
  \- Cache: Hold queues in-memory; thread-safe with Mutex if needed.  
  \- Error: Return Result\<Queues\> with exceptions (e.g., FileNotFound → use defaults).

\#\#\# 4.4 Infrastructure Layer  
\- \*\*Speech Module\*\* (:speech module):  
  \- \*\*LlmService\*\*: Retrofit/OpenAI SDK wrapper. \`generate(prompt: String): String\` – ChatGPT call (model: gpt-4o-mini for cost/speed).  
  \- \*\*SttService\*\*: \`recognizeAudio(): String\` – Use MediaRecorder; send to Whisper API (language: "de"); handle low confidence (\<0.5) → "Please repeat" bilingual prompt.  
  \- \*\*TtsService\*\*: \`speak(text: String)\` – Send to TTS API (voice: de-DE casual); play via MediaPlayer; callback on complete.  
  \- Config: API keys in buildConfig; German language code.

\#\# 5\. Data Flows and Sequences  
\#\#\# Session Start Flow:  
1\. Activity → ViewModel.resumeSession() → StartSessionUseCase → Repository.loadQueues().  
2\. Dequeue newTarget → GenerateDialogueUseCase → LlmService → TtsService.speak(text) → Update UiState.CoachSpeaking.  
3\. presentation++ for exposed items.

\#\#\# Turn Flow:  
1\. User taps Speak → SttService.recognizeAudio() → ViewModel.processResponse(text).  
2\. ProcessTurnUseCase: Update counts, check mastery.  
3\. If mastered, dequeue next → Generate next dialogue.  
4\. TTS plays response → Loop.

\#\#\# Error Flow:  
\- API fail → UiState.Error → Bilingual retry prompt via TTS.

\#\# 6\. Testing and Verification  
Align with TDD:  
\- \*\*Unit Tests\*\*: Per layer (e.g., repository parsing, use case mastery logic) using JUnit, Mockito (mock APIs/files).  
\- \*\*Integration Tests\*\*: End-to-end sequences (e.g., full turn with mocked APIs).  
\- \*\*UI Tests\*\*: Espresso for button interactions/indicators.  
\- \*\*Logs\*\*: Export sessionLog to file for manual verification (turns, counts).  
\- Coverage: Aim for 80%+ on domain/data; include edge cases (empty queue, recognition errors).

\#\# 7\. Dependencies and Build  
\- \*\*Libraries\*\*:  
  \- AndroidX: Core, AppCompat, Material.  
  \- Jetpack: Hilt, ViewModel, Coroutines, Flows.  
  \- JSON: Gson.  
  \- OpenAI: Official SDK or Retrofit \+ OkHttp.  
  \- Audio: MediaRecorder, MediaPlayer.  
\- \*\*Build Config\*\*: Multi-module Gradle; minSdk=26, targetSdk=35.  
\- \*\*Security\*\*: API keys encrypted in build; no sensitive data stored.

This spec provides a blueprint for implementation. For code generation or refinements, provide additional details.