# Technical Design & Test Plan - Feature Level (TDD) 

## 1. Feature Overview
- **Feature Name**:  
- **ID**: If available
- **Summary**: What is this feature and why are we building it?  
- **Scope**: In-scope and out-of-scope behaviors  
- **Owner**: Tech Lead  

---

## 2. Architecture Notes (if applicable)
- Module(s) affected (`:app`, `:domain`, etc.):  
- Entry points and interactions with existing layers:  
- Relevant interfaces, services, or view models to update:  

---

## 3. Design & UX
- **Screens Affected**:  
- **Navigation Flow**:  
- **Design Spec / Figma Link**:  
- **Accessibility Considerations**:  
- **Animation / Transition Requirements**:

---

## 4. Data & State
- **New Models or DTOs**:  
- **Persistence Required?** (`:data` layer impact):  
- **Local vs Remote State Handling**:  

---

## 5. API / Backend
- **Endpoints to Integrate**:  
- **Schema / Contract** (request/response):  
- **Error Handling Requirements**:  
- **API Mocks or Fakes Needed**:  

---

## 6. Analytics & Instrumentation
- **Events to Track** (name, parameters):  
- **Trigger Points**:  
- **Tooling** (Firebase, Segment, etc.):  
- **Tagging Format Compliance**:  

---

## 7. Feature Flags / Rollout Plan
- **Flag Name**:  
- **Default State**:  
- **Gating Conditions**:  
- **Remote Config Integration?**  
- **Rollback Strategy**:  

---

## 8. Test Specifications – Canonical Format

### 📂 Tag: `[Insert correct test tag — e.g., B-*, FS-*, PS-*]`
Use the following format for each planned test, and ensure files are named like:  
**`<Tag><n>_<behavior>.kt`**  
> Example: `B1_StartSessionResetsQueues.kt`

### Test Layer Naming
- **FS-*** – File system / data persistence  
- **B-*** – Pure business logic  
- **PS-*** – Prompt snapshot  
- **C-*** – Speech fake testing  
- **DI-*** – Dependency injection  
- **VM-*** – ViewModel logic  
- **UI-*** – Espresso / UI testing  
- **INT-*** – End-to-end tests  

#### ✅ Example Test Entry

| ID | Scenario | Pre‑Conditions | Steps | Expected | **Implementation** | **Dependencies** |
|----|----------|----------------|-------|----------|--------------------|------------------|
| **B‑1** | StartSession resets & dequeues | Repository returns queues | `StartSessionUseCase.startSession()` | First `new_item` counts reset; queues persisted | `domain/src/test/.../StartSessionUseCaseTest.kt` `@Test fun B_1_StartSession()` | MockK, Coroutines‑Test |

> Add more rows per tag as needed for unit, integration, or snapshot testing.

---

## 9. Testing Strategy Summary

### 🎯 Test Strategy Overview
- Unit tests: Required? Who owns them?
- Integration tests: Needed?
- Snapshot / Golden file tests: Required for prompts (e.g., `PS-*`)?
- Manual testing: When, and on what devices?

---

## 10. Test Data & Fixtures
- Test asset files (e.g., JSON in `/fixtures`)  
- UUIDs or golden IDs (use `00000000-0000-0000-0000-000000000000` for snapshot normalization)  
- Mock payloads or local stubs required

---

## 11. Tooling & Coverage Requirements
- Libraries: JUnit5, MockK, Kotest, Jimfs, Espresso, etc.  
- Coverage requirements   
- CI constraints   

---

## 12. Manual QA Requirements
- Platforms / devices:  
- RTL / Localization / Accessibility checks:  
- Performance / Animation testing:  
- Known limitations:  

---

## 13. Open Issues & Risks
- Dependencies blocked (e.g., API readiness, design spec WIP)  
- Test instability or CI flakiness concerns  
- Technical debt or shortcuts taken  

---

## 14. Final Deliverables Checklist

- [ ] All test cases defined in canonical table format
- [ ] Test IDs assigned with tag prefix (e.g., B‑1, FS‑2)
- [ ] Coverage requirements met (≥80% domain/data)
- [ ] Snapshot test written if prompt-related (PS‑*)

---

_Last updated: [Insert Date]_
