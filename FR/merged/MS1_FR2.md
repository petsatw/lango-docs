# Feature Request: Session Initialization & Prompt Builder

### Title
Session Initialization & Prompt Builder

### Problem
On session start there is no logic to (a) select and reset the next “new” learning item, and (b) assemble the very first coach prompt—including both orchestration instructions and the bundle of `new_target` + `learned_pool`—in the precise JSON-schema format needed downstream. Without this, the app cannot deterministically launch a practice session or validate against snapshot tests.

### Scope
- **Dequeue & Reset**  
  - In `StartSessionUseCase`, call `repository.loadQueues()`, dequeue the first item from `newQueue`, and set its `presentationCount = 0` and `usageCount = 0`.  
- **Prompt Construction**  
  - In `GenerateDialogueUseCase.buildInitialPrompt()`, emit a coach-ready JSON payload that:  
    1. Loads instructions from `PROMPT.md` orchestration sections.  
    2. Embeds the selected `new_target` (id + token).  
    3. Embeds the full `learned_pool` with each item’s counters.  
    4. Includes the JSON header and delimiter markup per schema.  
- **Count Increment**  
  - After introduction, increment `presentationCount` on the `new_target` so that subsequent turns follow the dialogue loop rules.

### Non-Goals
- UI wiring or button handling (covered under F-4/F-5).  
- Actual LLM or TTS invocation—it remains stubbed behind `FakeLlmService`/`FakeTtsService`.  
- Persistence of modified counts—handled by F-1 and F-5.

### Acceptance Criteria
| Test ID | Given                                                                      | When                                             | Then                                                                                                                           |
|---------|----------------------------------------------------------------------------|--------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| **B1-UT**  | A `newTarget` exists in the loaded queues with non-zero counts.           | `StartSessionUseCase.invoke()`                   | Returns `Queues` where the dequeued `new_target` has `presentationCount == 0` and `usageCount == 0`.                             |
| **PS-1**   | A clean `Queues` with one dequeued `new_target` and the remaining pool.   | `GenerateDialogueUseCase.buildInitialPrompt()`   | Returns a string exactly matching the approved fixture `initial_prompt.txt`.                                                  |

### Affected Modules
- **domain**  
  - `StartSessionUseCase`  
  - `GenerateDialogueUseCase`  
- **data**  
  - `LearningRepository` (provides `loadQueues()`)  
- **di**  
  - Ensure use cases are injected for testability.

### Data Contract
```json
{
  "header": {
    "sessionId": "<uuid>",
    "newTarget": { "id": "<id>", "token": "<text>" },
    "learnedPool": [
      { "id": "<id1>", "token": "<text1>", "presentationCount": 2, "usageCount": 1 }
    ],
    "delimiter": "—"
  },
  "body": [
    "You are now “Lango,” a voice-only coach…",
    "Explain what '<new_target>' means in a very short sentence.",
    "Give one simple example with '<new_target>'."
  ]
}
```

### UX Assets
- `initial_prompt.txt` – approved snapshot fixture in test resources.

### Test Guidance
1. **B1-UT**: Arrange a stubbed `Queues` with known `new_target` counts; invoke `StartSessionUseCase`; assert counts reset.  
2. **PS-1**: Arrange a minimal `Queues`; call `buildInitialPrompt()`; diff against `initial_prompt.txt`.  

