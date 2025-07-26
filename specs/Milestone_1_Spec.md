# Milestone 1 – Unified Specification & Roadmap for **Lango MVP**

_Last updated: 23 July 2025_

---

## 1 · Purpose & Context
Milestone 1 (MS1) demonstrates an **offline‑first coaching session** that loads the learner’s queues, shows the initial coach prompt, allows a single mocked turn, and persists state when the user ends the session. This document **merges** the previous _MS1_Roadmap_ and _MS1_Spec_ so that all teams can rely on **one source of truth** going forward.

---

## 2 · Milestone‑level Exit Criteria  
The milestone is complete when **all** of the following are true on a debug/UAT build:

1. App launches, copies bundled JSON to `filesDir`, and loads queues without error.  
2. First `new_item` is dequeued, counters reset, and the **initial coach prompt** is built from `PROMPT.md` and displayed.  
3. User taps **Start Session** → fake LLM reply is parsed, `presentationCount` increments and is **persisted**.  
4. User taps **End Session** → queues are written atomically and the activity finishes.  
5. All unit, integration, and UI tests listed in § 5 pass in CI.

---

## 3 · Feature Breakdown

| ID | Feature | Scope & Deliverables | Units of Work | Acceptance Tests |
|----|---------|----------------------|---------------|------------------|
| **F‑1** | **Bootstrap & Persist Queues** | • Copy `new_queue.json` & `learned_queue.json` from assets → `filesDir` (first launch).<br>• Load queues via Kotlinx‑Serialization (snake_case ↔ camelCase).<br>• Thread‑safe save with `Mutex` & atomic temp‑file rename.<br>• Fallback on malformed JSON or missing files.<br>• **Persist reset counts after StartSession**. | A‑1, A‑2, A‑3 | **FS‑1 – FS‑6** *(FS‑6 = verify reset counts persist)* fileciteturn11file6L7-L9 |
| **F‑2** | **Session Initialization Logic** | • `StartSessionUseCase`: dequeue first target, set `presentationCount` & `usageCount` → 0, **call `saveQueues()`**.<br>• `GenerateDialogueUseCase`: build initial prompt per template, including new target + full learned pool.<br>• Increment `presentationCount` on introduction and persist. | B‑1, B‑2 | **B1‑UT**, **PS‑1** (snapshot), FS‑6 coverage |
| **F‑3** | **Mock Speech Infrastructure** | Offline `FakeLlmService` & `FakeTtsService`; DI switch on `BuildConfig.UAT_MODE`. | C‑1, C‑2, E‑1 | C1‑UT, C2‑UT, DI‑1 |
| **F‑4** | **UI Foundation** | `activity_main.xml` (scrollable coach text + Start/End button); `MainViewModel` (StateFlow); wiring in `MainActivity`. | D‑1, D‑2, D‑3 | VM‑1, UI‑1 |
| **F‑5** | **Session Flow Integration** | Wire F‑2 logic & F‑3 fakes into F‑4 UI for full mocked turn. | Integrate B, C, D | UI‑1 (integration) |

---

## 4 · Detailed Feature Sections

### F‑1 — Bootstrap & Persist Queues (✅ Complete)
- **Implementation highlights**: mutex‑guarded atomic writes; resilient loader that restores assets on corruption; **new FS‑6 test ensures counts reset by `StartSessionUseCase` survive a second launch**.
- **Tests**: FS‑1 cold‑start, FS‑2 persist/reload, FS‑3 concurrency, FS‑4 malformed JSON, FS‑5 missing files, **FS‑6 reset‑count persistence**.

### F‑2 — Session Initialization Logic (✅ Complete)
- Dequeues first target, zeros counters, **saves queues immediately** for crash‑safety.  
- Builds initial prompt using `InitialPromptBuilderImpl`, pulling header/body templates from `PROMPT.md` and passing complete learned pool to the LLM.  
- Increments `presentationCount` after successful LLM reply.

### F‑3 — Mock Speech Infrastructure (🟡 In‑Progress)
Offline fakes compile in test source‑set; production‑side classes and DI wiring still to be added.

### F‑4 — UI Foundation (🟡 In‑Progress)
Layout skeleton pending; ViewModel tests green.

### F‑5 — Session Flow Integration (🟡 Blocked on F‑3 & F‑4)
End‑to‑end happy‑path test exists but UI wiring not yet complete.

---

## 5 · Sprint Plan
| Sprint | Goals |
|:--:|---|
| 1 | F‑1 complete + CI pipeline configured |
| 2 | F‑2 merged & prompt snapshot stabilised |
| 3 | F‑3 mocks & DI toggle |
| 4 | F‑4 UI layout & ViewModel |
| 5 | F‑5 integration + UI tests |
| 6 | Regression, bug‑fixes, demo prep |

---

## 6 · Risks & Mitigations
1. **Fixture DSL Drift**  
   *Risk*: `TestFixtures.kt` diverges from JSON schema → bogus snapshots.  
   *Mitigation*: CI job round‑trips DSL → JSON → DSL.
2. **DI Mis‑configuration**  
   *Risk*: Real services injected in UAT or vice‑versa.  
   *Mitigation*: Integration test toggling `UAT_MODE` (DI‑1).
3. **Prompt Template Drift**  
   *Risk*: Changes to `PROMPT.md` break PS‑1 snapshot.  
   *Mitigation*: Lock schema; require review.

---

## 7 · References
- `FR2QA_Build.md` (build log & tests)
- `PROMPT.md` (template)
- `FR2_And_Builder_Tests_Development_Plan.md`
- `TDD.md`
- Source code in `:domain`, `:data`, `:shared-test` modules

---

*End of unified Milestone 1 spec.*

