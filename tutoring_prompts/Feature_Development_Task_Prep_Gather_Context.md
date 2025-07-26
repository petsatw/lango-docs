Act as a world class mobile application architect

# CONTEXT

I am the lead developer tasked with delivering NEXT_STEP and after reviewing MY_CB, TDD_F, NP_F and ARCH_TL these are the questions that I need clarified in order to begin NEXT_STP

Here’s a focused list of the **key impact questions** we need to answer before diving into **Step 1 (Define & Expose Core Settings)** against our current build (MY\_CB):

1. **Kotlin Language Support for Inline Classes**
   SM\_REFACTOR\_CURRENT\_TASK proposes using

   ```kotlin
   @JvmInline value class PositiveInt(val value: Int) { init { require(value > 0) } }
   ```

   Does our Kotlin plugin (org.jetbrains.kotlin.android 2.2.0 / Kotlin 2.1.0) and Gradle setup fully support `@JvmInline` classes, or do we need to adjust compiler settings or fall back to a regular data class?&#x20;

2. **Domain Module Package Structure**
   We’ll add `SessionConfig` under
   `domain/src/main/kotlin/com/example/domain/config/SessionConfig.kt`.
   Is our current source-set configuration already picking up a `config/` subfolder, or do we need to tweak `domain/build.gradle.kts`?&#x20;

3. **BuildConfig Field Integration**
   We plan to inject three ints via

   ```kotlin
   buildConfigField("int", "REQUIRED_USAGES", "3")
   buildConfigField("int", "REPEAT_MODE_THRESHOLD", "2")
   buildConfigField("int", "MAX_REPEAT_LOOPS", "3")
   ```

   in `app/build.gradle.kts`. Will these end up correctly in `com.example.lango_mvp_android.BuildConfig`, and is there any existing build-flavor logic or CI caching that might conflict?&#x20;

4. **Hilt DI Module Discovery**
   Adding

   ```kotlin
   @Module @InstallIn(SingletonComponent::class)
   object ConfigModule { … }
   ```

   under `com.example.lango_coach_android.di`. Can Hilt already discover this alongside our existing modules (RepositoryModule, SpeechModule, etc.), or do we need to adjust component paths or qualifiers?&#x20;

5. **Legacy Orchestrator & Tests Isolation**
   Step 1 is scoped not to touch our legacy use-case classes (`StartSessionUseCase`, `ProcessTurnUseCase`, etc.) or their tests. How will we ensure no accidental refactors of those until Step 5? Can we add CI guard rails or mark them deprecated to prevent drift?&#x20;

6. **Constructor & Injection Impact on Existing Tests**
   Once `SessionConfig` enters DI, will any existing unit or integration tests break for lack of a provider? Should we pre-add a test-scoped `SessionConfig` binding or helper in our test setup?&#x20;

7. **Test Fixtures & Future FSM Coverage**
   Our forthcoming FSM golden-path tests (SM\_REFACTOR\_SPEC §4.1) will drive sequences against a runner that needs `SessionConfig`. Should we extend `shared-test/TestFixtures` now with helpers to build arbitrary `SessionConfig` instances?&#x20;

8. **JSON Persistence Schema Versioning**
   SM\_REFACTOR\_SPEC §1 describes a `schemaVersion` in the queue JSON, but our assets (`new_queue.json`, `learned_queue.json`) lack it. Will adding runtime config force us to bump or migrate our asset schema now, or can that wait until later steps?&#x20;

9. **CI Build & Test Metrics Monitoring**
   Our goal is ≥90% first-compile and ≥80% first-green-test. Which CI jobs (`:domain:compileKotlin`, `:domain:test`, `:app:compileKotlin`) should we watch most closely after merging the config PR to catch regressions early?&#x20;

10. **Documentation & Spec Alignment**
    The state-machine spec (ARCH\_TL) hard-codes thresholds (`required_usages: 3`, etc.). Should we update that document to reference `SessionConfig` defaults to keep everything single-sourced, or leave the spec static until full refactor?&#x20;

Answering these will surface any hidden dependencies or gaps, so the team can hit our “first‐compile green” and maintain our compile/test metrics on PR #1 with confidence.


# CRITERIA

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

# INSTRUCTIONS

Carefully read in MY_CB, TDD_F, NP_F, ARCH_TL and NEXT_STEP. Review these questions and answer them in order to guide the team on NEXT_STEP. Provide them a high level of context detail to understand the reasoning behind your design decisions and how that design decision is most relevant to MY_CB. You need to understand, first and foremost, that the development team will be laser focused on implementing NEXT_STEP. They have no time to comb back through all of this documentation to find all the details they need. This set of instructions and guidance, coupled with NEXT_STEP will be what they use to perform 99% of their work on this task. So it needs to be as detailed as necessary and very, very clear. 

Work off the assumption that they have very littel awareness of the technical details in  any other documents. So answer these questions and guide them well. 