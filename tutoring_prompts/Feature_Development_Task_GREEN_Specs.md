Act as a world class mobile application architect

# Context

I am the lead developer on my team, we have been tasked with TDD_F and DP_F and I want to make sure we have all the details and context we need to implement the current task:

### Step 1: Define & Expose Core Settings  
**Goal:** Centralize all thresholds in a single config.  
1. **Create** `SessionConfig` in domain (`domain/src/main/kotlin/com/example/domain/config/SessionConfig.kt`)  
2. **Provide** via Hilt `ConfigModule` (`app/src/main/java/com/example/lango_coach_android/di/ConfigModule.kt`)  
3. **Define** BuildConfig fields in `app/build.gradle.kts`  

> All code must reference `SessionConfig`—no literals.  
> **See SM_REFACTOR_SPEC §1** “Must‑Be‑True Invariants & Core Domain Model”

In order to meet the high compile and build requirements, we have very little margin for error, I need as much detailed context and clarity as possible, so I would like to define a highly detailed specification specifically for this task.  

# Criteria

Follow a first principles approach to the design and execution of this application including this task. Your goal first and foremost is to understand with as much clarity possible the objective and **what must be true** in order to meet the objectives. Everything else is a distraction. 

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

# Examples

[add examples here]

# Instructions

[add here]

## [input variable]

[add inputs here]