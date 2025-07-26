Act as a world class mobile application architect

# Context

You have been tasked with refactoring MY_CB in order to implement ARCH_TL, following DP_F. 

# Criteria

Follow a first principles approach to the design and execution of this application including this refactor. Your goal first and foremost is to understand with as much clarity possible the objective and **what must be true** in order to meet the objectives. Everything else is a distraction. 

## Make your requirements less dumb.
- Every requirement, especially from “smart” people or organizations, should be rigorously questioned.
- Most errors come from bad assumptions that were never challenged.
- Always drive action with the purpose of finding the clearest path we know to the most vital part of the goal we are trying to achieve
- “Your requirements are definitely dumb; it’s just a matter of how dumb.”

## Try very hard to delete parts of the process or product.
- If something is not absolutely necessary, remove it.
- “If you’re not occasionally adding things back in, you’re not deleting enough.”
- Simplicity is a feature. Complexity is a bug.

## Optimize the design only after doing steps 1 and 2.
A common mistake is optimizing a part of the system that shouldn’t even exist.
Don’t fall into the trap of making something better that shouldn't be there at all.

## Speed Up the Process
First fix the what, then improve the how fast.

## Automate last.
Don’t automate something until it has been validated and optimized.
Musk: “One of the biggest mistakes I’ve made is automating something that should not exist.”

# Instructions

In order to ensure we are prioritizing the right things, here are the Business Objectives & Success Criteria for this build. 

* A learning experience for a beginner to master language quickly through spoken, natural dialogue
* Architecture that embodies the conversational interactions, while clearly tracking progress towards learning objectives. 
* Success metrics for the State Machine refactor should be a significant decrease in test debugging and corrections. At each build point, the chance of successfull compile on first build should be 90% and passing tests on first build should be higher than 80%

In order to scope out a more detailed context for this development effor please respond to the following given everything you know about the Lango mission and the context for these build objectives. If any of these sections do not add any value or introduce added confusion or unecessary complexity to our objectives, feel free to identify them as LOW PRIORITY and leave them unanswered. 

### “Must-Be-True” Invariants & Domain Model

* **Which domain invariants can never be violated?** For example:

  * `counters.usage ≤ required_usages`
  * `state_loop_count ≤ max_repeat_loops`
* **What are the exact semantics of “newTarget”** throughout the session?
* **What level of backwards compatibility** is required for existing saved queue JSON?

### Scope of Refactor & “Delete First” Mindset

* DP_F says to “remove the old turn-based code and tests” .

 **Which pieces of that code are genuinely dead?** Could any still hold unique logic we need?
 **What’s the minimal slice we can refactor first** to validate the YAML-DSL approach before ripping out everything?


### Runtime DSL Interpreter

* **What “action” hooks absolutely must exist** (e.g. `evaluate_usage`, `coach_slow_repeat`)?

* Which of these require synchronous vs. asynchronous behavior?
* **What performance and memory constraints** must the runner satisfy on a low-end Android device?
* **How will we trace and debug** state transitions in production? (logging, telemetry)

### ViewModel & UI Integration

* DP\_F suggests replacing `MainViewModel.startSession()`/`processTurn()` with simple `fsm.on(event)` calls .

  * **What “renderState” mapping** from FSM states → `UiState` must be one-to-one?
  * **Which UI states are non-blocking** (e.g. waiting for TTS) vs. terminal?

### Testing & Quality

* **What’s the golden-path test coverage** we need before deleting existing tests?
  * Will one parameterized FSM test cover both happy path and repeat-after-me flows?
  * **Which existing contract tests (e.g. FS-1…FS-5) still add value** and can be retained?
* **How do we continuously validate** that the YAML spec and code stay in sync?

### Automation & Tooling

* **What parts of this process should remain manual** initially (e.g. spec editing, test approval)
* **What could we automate later** (e.g. regenerating test vectors from YAML, CI checks on spec validity)?

---

By answering these, we ensure we only keep what’s absolutely necessary, validate the spec, and build a minimal, debuggable FSM core that truly meets our objectives.


## [input variable]

[add inputs here]