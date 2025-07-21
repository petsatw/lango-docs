\# Technical Design Document (TDD) for Lango App Phase Zero MVP

\#\# Version 1.0  
\*\*Date:\*\* July 15, 2025    
\*\*Author:\*\* Grok (based on interview responses)    
\*\*Purpose:\*\* This document serves as the fundamental building block for the Lango app's phase zero MVP. It defines the core requirements, behaviors, and test cases to ensure clarity for development. The focus is on Test-Driven Development (TDD) principles, where test cases accurately represent the desired functionality for personal use as the first user. This MVP emphasizes a minimal voice-based coaching session for learning German as a beginner, with extensibility in mind for other languages. All requirements are derived from the provided PDF logic (e.g., core\_blocks.json for new objectives, learned\_queue.json for progress tracking with usage\_count and presentation\_count) and interview details.

The app is an Android application requiring internet for OpenAI APIs (ChatGPT for dialogue generation, Whisper for STT, TTS for coach voice). It uses modular design: a data access module for JSON handling (agnostic to source/format) and a speech module for STT/TTS (using OpenAI initially).

Phase zero is strictly for personal testing: single-user, local storage, no advanced features like user profiles, dashboards, reminders, or monetization. Success is measured by functional turns in sessions, correct mastery transitions, and accurate count tracking, verified via conversation logs.

\#\# 1\. Overall Product Vision and Scope  
\#\#\# Requirements  
\- Target audience: Beginners learning German, no location restrictions.  
\- Language support: German only for MVP, but architecture must be extensible (e.g., configurable language codes, separate JSON per language).  
\- Features: Core voice coaching via dialogues prompting use of new targets; track usage\_count (times user correctly uses item) and presentation\_count (times item is presented by coach) separately for learned items; bias dialogues toward less-used learned items (e.g., weight selection by lower counts).  
\- No multi-user or cloud sync; local only.  
\- Key metrics: Number of completed turns, successful mastery (usage\_count \>=3), correct count increments; log coach/user text for verification.

\#\#\# Test Cases  
\- \*\*Test Vision Extensibility:\*\* Given app config for German, when loading JSON for another language (simulated), then core logic processes without hard-coded German assumptions.  
  \- Expected: No errors; dialogues generate in the new language if TTS/STT supports it.  
\- \*\*Test Metrics Logging:\*\* Given a session with 5 turns, when session ends, then log file contains text of all coach prompts and user responses, with final usage/presentation counts.  
  \- Expected: Log matches expected increments (e.g., presentation \+1 per coach use, usage \+1 per detected user use).

\#\# 2\. User Interface and Experience (UX/UI)  
\#\#\# Requirements  
\- Minimal UI: Home screen with "Resume Session" button (changes to "End Session" on click); separate "Speak" button for user input; menu with "Load New Session" to prompt file pickers for core\_blocks.json (new\_queue) and learned\_queue.json (learned\_pool with counts).  
\- Visual feedback: Indicator (e.g., icon/dial) for listening mode (user speaking); waiting indicator for coach response (due to API delay); display coach's spoken text on screen.  
\- Session handling: Tap buttons only (no voice commands); session ends on button tap or phone call interruption (no pause/resume for other events like backgrounding).  
\- Onboarding: Request microphone permissions on first launch; start with predefined bundled JSON files (in app assets) presenting the "Resume Session" screen; no intro screens, assume user knows to use menu for loading new files (overwrites existing).  
\- Accessibility: None specified for phase zero; focus on core voice/text.  
\- Error display: If internet error or recognition fail, show error message on screen (e.g., "Connection issue" or "Didn't hear—repeat").

\#\#\# Test Cases  
\- \*\*Test UI Simplicity:\*\* Given app launch, when permissions granted, then screen shows "Resume Session", "Speak" button, and menu; no other elements.  
  \- Expected: Buttons function as taps; menu opens file pickers for two JSON files.  
\- \*\*Test Visual Feedback:\*\* Given a turn start, when coach speaks, then text appears on screen and audio plays; during user input, listening indicator shows; during API wait, waiting dial animates.  
  \- Expected: Indicators toggle correctly; text matches TTS output.  
\- \*\*Test Onboarding:\*\* Given first launch without permissions, when app starts, then microphone permission dialog appears; on grant, loads predefined JSON and shows main screen.  
  \- Expected: Session resumable with bundled data; no additional setups.  
\- \*\*Test Interruption Handling:\*\* Given active session, when simulated phone call, then session ends and data writes to local JSON.  
  \- Expected: No resume; next launch loads from saved state.  
\- \*\*Test Error Display:\*\* Given no internet during turn, then screen shows "Ich habe das nicht verstanden – bitte wiederholen" in German) and "I didn't hear that—please repeat" in English; session continues if retried.  
  \- Expected: Message uses bilingual format; logs the error.

\#\# 3\. Core Functionality and Logic  
\#\#\# Requirements  
\- Session Structure: A session is the full conversation; turns are individual exchanges (coach prompts 1-3 sentences in simple German, user responds using new\_target).  
\- Dialogue Generation: Prompt LLM with new\_target, learned\_pool (incl. counts), instructions to use only allowed vocab, create natural dialogues prompting new\_target use, bias to low-count items, explain new\_target simply (like for a 5-year-old) on introduction with example sentence.  
\- Mastery: Dequeue new\_target to learned\_pool when usage\_count \>=3 (detected via STT matching in user response); transition immediately in same session—next turn introduces new target (say in German, explain briefly in German then English, example in German).  
\- Parametric Phrases: Coach uses natural examples with learned\_pool fillers; no validation, user completes as best they can.  
\- Recognition: Rely on OpenAI STT; no pronunciation feedback; if failure, recover with bilingual prompt to repeat; no accuracy thresholds beyond API confidence.  
\- Tracking: In-memory during session; increment presentation\_count when coach presents item, usage\_count when user uses it correctly (per STT match); write to JSON at session end or app close.  
\- Edge Cases: If new\_queue empty, show "Congratulations\! You've completed your learning objectives." screen.

\#\#\# Test Cases  
\- \*\*Test Session Flow:\*\* Given new\_queue with 2 targets, learned\_pool empty, when session starts and user completes 3 uses for first target, then next turn seamlessly introduces second target with explanation (German \+ English).  
  \- Expected: Usage\_count \==3 triggers move; logs show bilingual intro only for new target.  
\- \*\*Test Turn Completion:\*\* Given coach prompt, when user speaks and STT succeeds, then counts increment if match; coach generates next prompt via API.  
  \- Expected: Turn ends on response; API call includes updated pool/counts for bias.  
\- \*\*Test Mastery Transition:\*\* Given usage\_count=2 for target, when third correct use detected, then target moves to learned\_pool; presentation/usage tracking continues.  
  \- Expected: No interruption; next dialogue biases to the new learned item if low counts.  
\- \*\*Test Parametric Handling:\*\* Given parametric new\_target like "Ich heiße \_\_", when coach prompts, then uses learned filler (e.g., name from pool); user response accepted without validation.  
  \- Expected: STT text matches for usage increment if contains target structure.  
\- \*\*Test Recognition Error:\*\* Given poor STT, when failure, then bilingual repeat prompt; session continues.  
  \- Expected: No count increments; log shows error prompt.  
\- \*\*Test Empty Queue:\*\* Given last target mastered, when queue empty, then session ends with congrats screen.  
  \- Expected: No further turns; data saved.

\#\# 4\. Technical Integrations and Requirements  
\#\#\# Requirements  
\- STT: OpenAI Whisper API; send audio, get text; handle German beginner accents leniently via API.  
\- TTS: OpenAI TTS API; casual German accent (Linz if possible, else standard); display text alongside audio.  
\- Connectivity: Requires internet; error message if disconnected during session.  
\- Persistence: Local JSON via data module; read on load, write on end/close; predefined assets for initial.  
\- Android: No min API specified; require microphone; no other constraints.  
\- Security/Privacy: None for phase zero; transient audio processing only.

\#\#\# Test Cases  
\- \*\*Test API Integration:\*\* Given user audio, when sent to STT, then returns text; TTS generates audio from text.  
  \- Expected: Successful calls; German accent in output.  
\- \*\*Test Offline Handling:\*\* Given no internet, when turn starts, then error message; app doesn't crash.  
  \- Expected: Session halts gracefully.  
\- \*\*Test Persistence:\*\* Given session with updates, when ended, then JSON files overwritten locally with new counts.  
  \- Expected: Reload matches saved state; predefined files load if no custom.

\#\# 5\. Content and Data Management  
\#\#\# Requirements  
\- JSON Handling: core\_blocks.json (new\_queue: words/phrases); learned\_queue.json (learned\_pool with usage/presentation counts); load via file picker or predefined.  
\- Updates: App overwrites on save; no user-editable in-app.  
\- Scalability: Modular for future extensions (e.g., server fetch).

\#\#\# Test Cases  
\- \*\*Test Data Load:\*\* Given valid JSON files picked, when loaded, then new\_queue and learned\_pool populated; overwrites existing.  
  \- Expected: Session uses loaded data; invalid files show error.  
\- \*\*Test Save:\*\* Given count changes, when app closes, then JSON updated with accurate counts.  
  \- Expected: File contents match in-memory state.

\#\# 6\. Testing and Verification  
\#\#\# Requirements  
\- Critical Scenarios: File read/write success; API call success; turn completion with correct increments; recognition errors; multiple transitions in long sessions.  
\- Logs: Text-only conversation log (coach/user) for manual verification; no audio storage.  
\- No monetization, analytics, or external integrations.

\#\#\# Test Cases  
\- \*\*Test File Operations:\*\* Given predefined files, when app starts, loads correctly; after changes, writes successfully.  
  \- Expected: No corruption; verifiable via external file check.  
\- \*\*Test API Calls:\*\* Given mock inputs, when calls made, then responses process without errors.  
  \- Expected: Turn completes in \<10s (accounting for delay).  
\- \*\*End-to-End Session Test:\*\* Given full queue, simulate user responses to master all; verify logs, counts, and congrats.  
  \- Expected: All metrics match (turns \= expected, transitions at \>=3). 

\#\# 7\. Core Functionality Tests  
\#\#\# Purpose  
This section prioritizes the foundational pipeline for iterative development and early validation. It breaks the core sequence (file read → API send with test instruction → API receive → audio play) into individual unit tests for each component. These unit tests allow independent verification and debugging. Additionally, an end-to-end integration test combines them to ensure the full flow works seamlessly. Use mocks (e.g., for APIs) in unit tests to isolate components; remove mocks in the integration test for real execution. Focus on a simple test instruction (e.g., a sample prompt like "Generate a simple greeting using the new\_target 'Hallo'") to simulate the initial coach response without full session logic.

\#\#\# Unit Tests  
\- \*\*Unit Test 1: File Read (Data Module)\*\*    
  Given predefined or picked JSON files (core\_blocks.json with sample new\_queue, learned\_queue.json with sample learned\_pool), when the data module's read function is called, then files are parsed into in-memory structures (e.g., lists/maps for queues and counts).    
  \- Expected: Structures match JSON content; handle invalid JSON with error (e.g., throw exception or return empty).

\- \*\*Unit Test 2: API Send (Speech/LLM Module)\*\*    
  Given in-memory data from files (e.g., new\_target from new\_queue, learned\_pool), when a test instruction prompt is constructed and sent to OpenAI ChatGPT API (mocked response), then the API call includes the correct prompt (e.g., incorporating new\_target and instructions for simple dialogue).    
  \- Expected: Prompt format is correct; mocked send succeeds without errors; log the sent prompt for verification.

\- \*\*Unit Test 3: API Receive (Speech/LLM Module)\*\*    
  Given a mocked API response (e.g., JSON with generated text like "Hallo\! Das ist ein Gruß."), when the receive function processes it, then the text is extracted and prepared for TTS (e.g., stored in a string variable).    
  \- Expected: Text matches mocked response; handle empty/invalid responses with fallback (e.g., error message).

\- \*\*Unit Test 4: Audio Play (TTS Module)\*\*    
  Given received text from API, when TTS API is called (mocked audio output), then audio is generated and played (simulate play with a callback or log). Display text on screen simultaneously.    
  \- Expected: TTS call uses correct text and German accent config; play succeeds; screen text updates.

### File I/O Tests (FS‑1 to FS‑5)

| Test ID | Level         | Scenario & Steps           | Expected Assertions                                                 | Status                              |
|---------|---------------|----------------------------|---------------------------------------------------------------------|-------------------------------------|
| FS‑1    | Instrumented  | Cold‑start load            | Result.success; queues created; contents match assets               | ✅ Active                           |
| FS‑2    | Instrumented  | Persist & reload           | Modified count persists after reload                                | ⚠️ Skipped (Robolectric/Windows)    |
| FS‑3    | JVM           | Concurrent read/write      | No races or errors when load + save run simultaneously              | ⚠️ Skipped (Robolectric/Windows)    |
| FS‑4    | JVM           | Malformed JSON             | Result.failure; fallback to bundled defaults                        | ✅ Active                           |
| FS‑5    | JVM           | Missing file               | Result.success; assets copied; queues created                       | ✅ Active                           |

\#\#\# End-to-End Integration Test  
\- \*\*Integration Test: Full Core Sequence\*\*    
  Given app launch with predefined JSON files, when a "Test Core Flow" button or command is triggered (temporary for development), then: read files into memory → construct and send test instruction to ChatGPT API (real call) → receive response text → send to TTS API (real call) → play audio and display text.    
  \- Expected: Entire sequence completes without errors; audio plays the response (e.g., "Hallo\! Das ist ein Gruß."); log full trace (file content, sent prompt, received text, TTS success); fail if any step breaks (e.g., no internet → error message). This test runs in \<15s, verifying the pipeline before adding session/turn logic.