## Original Request (Rough Concept)

I think there is a sequence of milestones that will help us iterate through these outstanding design elements. 

Milestone 1: Application Initialized and Ready
Mode: UAT
Initialization is Feature Complete When:
- upon application start, file load is successful for both new and learned queues
- the first new_item has been dequeued 
- the initial coach prompt has been prepared and contains the new_item, the learned_pool plus all relevant instructions for the coach LLM needed for performing the task, responding with all necessary elements to initiate the introduction with the user
- the initial coach prompt displays on the screen
- most of the screen dedicated to a large text window that is scrollable if the text exceeds the length of the window. 
- the screen displays the coach prompt in it's entirety
- there is one button labeled "Start Session"
- when "Start Session" is clicked it does not make an actual LLM call, but everything should be in place so that with one small change to the call from UAT to UAT-LIVE, it would make the call to the LLM instead. 
- when "Start Session" is  clicked in UAT mode: the button changes to display "End Session", a mocked, predefined LLM response is sent to the coach handling function, the coach handling function updates presentation_count on any relevant items, the screen updates with the coach text + the changes to learned_pool and new_item presentation_counts. 
- when "End Session" is  clicked, the changes to new and learned queues are saved to file and the application closes

Milestone 2: User Completes First Interaction with Live LLM
Mode: UAT-LIVE
First Learning Objective is Feature Complete When: 
- Application Initializes and Loads
- An additional button "Tap to Speak" displays, but is inactive
- "Start Session" button displays and the initial prompt for the coach LLM is on the screen
- Upon "Start Session" click the prompt is sent to an LLM and the button changes to display "End Session"
- When the LLM response is received, the coach text is parsed and sent to the TTS service
- The TTS audio is receieved and played to the user
- The "Tap to Speak" button becomes active and the user is able to record their response by tapping the button to begin recording and tapping the button to end recording with an obvious visual indicator for when the application is recording. If the user does not tap to end the recording, it will automatically end the recording at 30 seconds. 
- Once the recording ends, it sends the audio to the STT service and receives a response
- Once the response has been received, the prompt is prepared for the Coach LLM and that prompt is displayed on the screen and contains the user response, any updates to new_item usage__count, the supporting data for the next learning round. The prompt is displayed but is not sent. Milestone 2 UAT will be complete
- Files are saved with the updated progress
- User clicks "End Session" to close the application

Milestone 3: User Completes First Learning Objective with Live LLM
Mode: UAT-LIVE
- Instead of ending after the first user response is received, the coaching session continues with the next turn. The coach prompt is displayed and sent to the LLM. 
- User responds until the item is mastered
- Item is added to the learned_pool and the new_item is dequeued and the new_item prompt is prepared to send to the Coach LLM. The prompt is displayed on the screen (but does not send to the LLM)
- Files are saved with the updated progress
- User clicks "End Session" to close the application

Milestone 4: User Completes All Items from New Queue - Session Complete
- All previous interactions continue until the new queue is exhausted. 
- Screen displays congratulations message
- User clicks "End Session" and progress is saved

Milestone 5: User Loads Their Next Learning Session
- menu button is added and contains one item "load learning objectives"
- file picker is used to upload a new set of objectives
- once loaded successfully, the new objectives are saved as the new queue, the learned queue is loaded in and the session begins.  



### Implementation sequence (no calendar attached)

Below is a pure *order-of-execution* checklist.
Work through each milestone top-to-bottom; move to the next only when every “Exit-criteria” bullet is green.

---

## **Milestone 1 – Application initialised and ready (UAT mode)**

**Core objective:** run the app, load queues, show the first prompt, exercise the full turn flow with **mock** LLM/TTS.

1. **File layer**

   * Implement and mutex-guard `LearningRepositoryImpl.loadQueues()` and `saveQueues()`.
   * Copy bundled JSON from `assets/` to `filesDir` on first launch.
   * Negative-path handling: missing file ➜ load defaults; malformed JSON ➜ fail with `Result.Failure`.

2. **Domain layer**

   * `StartSessionUseCase` dequeues first `new_item`, resets session counters.
   * `GenerateDialogueUseCase` builds the initial coach prompt **including**:

     * `new_item`
     * current `learned_pool`
     * output-format instructions for the LLM (JSON header + “---” delimiter).

3. **Speech mocks**

   * `FakeLlmService` returns a canned JSON payload (`sample_turn_1.json`).
   * `FakeTtsService` is a no-op.

4. **UI / presentation**

   * Single-screen layout:

     * 90 % height scrollable `TextView` bound to `uiState.coachText`.
     * `Button id=btnStartEnd` (label toggles “Start Session” ⇄ “End Session”).
   * Click “Start Session” → inject fake LLM reply, apply it, update counts, show new coach text.
   * Click “End Session” → save queues, finish activity.

5. **Exit-criteria**

   * App launches and shows the prepared prompt.
   * “Start Session” produces the canned coach reply and count mutations.
   * “End Session” persists queues and closes the app.
   * All Milestone-1 unit, instrumented and Espresso tests pass.

---

## **Milestone 2 – First live turn with LLM (UAT-LIVE mode)**

**Core objective:** replace mocks with real OpenAI LLM + TTS and add a single STT-enabled user reply.

1. **Run-mode switch**

   * `BuildConfig.UAT_MODE` (`true`=mock, `false`=live). Only DI providers branch on this flag.

2. **Live Speech services**

   * `OpenAiLlmService` (model `gpt-4o-mini`)
   * `OpenAiTtsService` (voice `de-DE`)
   * `OpenAiSttService` (Whisper)

3. **Response parser**

   * `LlmTurnParser` reads first line JSON → `coachText`, `presentationIds`, `usageIds`.
   * Fall-back to offline heuristics if JSON missing.

4. **UI additions**

   * `Button id=btnSpeak` (initially `gone` / disabled).
   * Recording indicator; auto-stop after 30 s.
   * After TTS playback completes, make `btnSpeak` visible and active.

5. **Flow**

   * Click “Start Session” → send prompt to LLM, parse reply, speak audio.
   * User records reply; send audio to STT; on text back, build *next* prompt and display it (do **not** send).

6. **Exit-criteria**

   * Coach speaks via TTS; user can record; STT returns transcript; next prompt shows.
   * Counts mutate per LLM JSON.
   * Session may be ended manually; queues persist.

---

## **Milestone 3 – First learning objective mastered**

1. **Looping logic**

   * Keep calling LLM / TTS / STT until `usageCount ≥ masteryThreshold` (e.g., 3).

2. **Transition**

   * When mastered, move item to `learned_pool`, dequeue next `new_item`, build and display prompt (but don’t yet send).

3. **Exit-criteria**

   * User can practise until first item mastered and promoted to `learned_pool`.
   * New target prepared and shown.

---

## **Milestone 4 – Session completes when new queue empty**

1. **Continuation**

   * Repeat Milestone 3 loop until `newQueue.isEmpty()`.

2. **Completion UI**

   * Display “Congratulations” message; disable `btnSpeak`, keep “End Session”.

3. **Exit-criteria**

   * After last item mastered, congrats screen appears; “End Session” saves state and closes.

---

## **Milestone 5 – Load a fresh set of learning objectives**

1. **Menu & file picker**

   * Overflow menu → “Load learning objectives”.
   * Invoke Android picker; return URI(s) for new & learned queues.

2. **Repository hook**

   * `LearningRepositoryImpl.loadQueues(uriPair)` to replace current queues and persist them.

3. **Exit-criteria**

   * User selects files; app loads them, shows first prompt of the new set, and is ready for Milestone 1 flow again.

---

### Supporting artefacts & tests (shared across milestones)

| Artefact / Test block                                     | Purpose                                                        |
| --------------------------------------------------------- | -------------------------------------------------------------- |
| **PROMPT.md – Output format section**                     | Defines mandatory JSON + “---” contract for every LLM reply.   |
| **sample\_turn\_1.json fixture**                          | Drives `FakeLlmService` in Milestone 1.                        |
| **Contract tests (MockWebServer)**                        | Assert that parser tolerates correct & malformed LLM replies.  |
| **File-system tests FS-1 … FS-5**                         | Cover cold-start, corruption, concurrent write, atomic save.   |
| **Regex matcher unit tests**                              | Verify parametric phrase detection (`Ich heiße ___`, etc.).    |
| **Live smoke test** (optional, opt-in via `-PwithOpenAI`) | Nightly call to GPT & Whisper ensures contract hasn’t drifted. |

---

Proceed through the milestones in order; each succeeding stage depends on the previous one’s exit-criteria being satisfied.
