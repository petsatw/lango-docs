### 1 · Is the “literal-string” PS-1 test still useful?

**Purpose of PS-1** in the spec is to guarantee that each *initial* prompt that leaves the domain layer is:

1. **Structurally correct** (header + body conform to the JSON schema).
2. **Semantically correct** (the orchestration text comes from *PROMPT.md* and the tokens from the queues are injected in the right places).
3. **Deterministic** (no hidden randomness or ordering drift).

Comparing the *entire* pretty-printed JSON string was a quick way to enforce all three when the prompt was hard-coded.
Now that:

* the body lines are read from **PROMPT.md** via `PromptTemplateLoader`,
* a real UUID is inserted for every session,

the “byte-for-byte” snapshot no longer serves #2 or #3 very well and **actively hurts #1** by failing on inconsequential whitespace or a legitimate change in the orchestration template.
So yes—the current test is **too literal and brittle**.

---

### 2 · What should we test instead?

| Layer                                         | Assertion goal                                | Recommended tactic                                                                                                                                                                                                                                                                                                                                                                               | Library need                                                                                                      |
| --------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| **Domain** (`InitialPromptBuilderImpl.build`) | Schema integrity + correct token substitution | 1. Parse the JSON back into the `Prompt` data class.<br>2. Assert:<br>  • `header.newTarget.token == queues.newQueue[0].token`<br>  • `header.learnedPool.size == queues.learnedPool.size`<br>  • `body[0]` starts with “You are now “Lango,”” (SYSTEM).<br>  • `body[1]` contains the substituted token.<br>3. **Ignore `header.sessionId`** by normalising it to a constant before comparison. | *No new libs.* Use Kotlin x-serialization (`Json.decodeFromString<Prompt>()`) which is already in the class-path. |
| **Approval / regression** (optional)          | Human-readable diff on orchestration edits    | Keep `initial_prompt.txt` as a *golden file* but compare **canonicalised JSON** (minified, UUID → 0000) so only meaningful changes break CI.                                                                                                                                                                                                                                                     | If you still want pretty diff: add `com.google.truth` (see § 4).                                                  |

This satisfies PS-1’s intent while eliminating flakiness from formatting, UUIDs, or future prompt-template rewrites.

---

### 3 · Concrete refactor steps

1. **Replace brittle string-equality in `InitialPromptBuilderTest`**

   ```kotlin
   val json = Json { ignoreUnknownKeys = true }
   val result = initialPromptBuilder.build(queues, UUID.fromString("00000000-0000-0000-0000-000000000000"))
   val prompt = json.decodeFromString<Prompt>(result)

   assertEquals(queues.newQueue.first().token, prompt.header.newTarget.token)
   assertEquals(queues.learnedPool.size, prompt.header.learnedPool.size)
   assertTrue(prompt.body[0].startsWith("You are now “Lango,”"))
   assertTrue(prompt.body[1].contains("Entschuldigung"))
   ```

   The body assertion stays stable because `PromptTemplateLoader` already substitutes `{newTarget.token}`.

2. **Normalise UUID for the optional snapshot file**
   Add a helper:

   ```kotlin
   fun String.normaliseUuid() =
       replace(Regex("\"sessionId\":\"[^\"]+\""), "\"sessionId\":\"00000000-0000-0000-0000-000000000000\"")
         .trim()
   ```

   Snapshot test then does

   ```kotlin
   assertEquals(loadFixture("initial_prompt.txt"), result.normaliseUuid())
   ```

3. **Drop `prettyPrint` config** in the production `Json` instance—minified JSON makes canonical comparison easier and cuts APK size.

4. **Fix `com.google.truth` compilation (optional)**

   * libs.versions.toml

     ```toml
     truth = "1.5.1"
     ```
   * build.gradle.kts (domain / shared-test)

     ```kotlin
     testImplementation(libs.truth)
     ```
   * Import: `import com.google.common.truth.Truth.assertThat`

   (The earlier “Unresolved reference” came from missing the dependency in the *shared-test* module.)

---

### 4 · Do we still meet the FR-2 acceptance criteria?

Yes. The spec says **“Returns a string exactly matching the approved fixture”**, but the fixture can be *canonicalised* (minified, fixed UUID).
Our revised test continues to load `initial_prompt.txt`; only the comparison method changes.

If product owners insist on a literal diff, keep both tests:

* **Structural test** (always runs).
* **Snapshot guard** (runs in CI, can be updated by `./gradlew approveSnapshots`).

Either way we eliminate the daily friction during development.

---

### 5 · Impact on other modules & build stability

| Affected area                   | Breakage risk                                               | Mitigation |
| ------------------------------- | ----------------------------------------------------------- | ---------- |
| **GenerateDialogueUseCaseTest** | None – uses mocks, not the real builder.                    | —          |
| **Integration & UI tests**      | Zero – they will pick up the new stable JSON automatically. | —          |
| **CI pipelines**                | Reduced flakiness; no new Android-specific test runners.    | —          |

No additional libraries are required unless you opt-in to Truth for nicer diffs.
Robolectric can still be removed as planned; the refactor does not re-introduce it.

---

### 6 · Recommended sequence to unblock the team

1. **Commit the structural rewrite of `InitialPromptBuilderTest`.**
2. Add the canonical-snapshot helper and update `initial_prompt.txt` (minified + UUID 0000).
3. (Optional) Wire in Truth for prettier messages.
4. Push → CI green.
5. Proceed with Step 2 (fixtures DSL) from the hardening plan.

This keeps PS-1 meaningful, removes the “brittle string” blocker, and requires *no* extra Android or heavy test libraries—exactly what we need to accelerate into Feature 3.

Commit & push; CI must stay green (all FR-1, B1-UT, PS-1).

Assess The Codebase to determine if these tasks remain or have been completed, complete those that are still remaining
2 Stabilise domain + data tests with the fixtures DSL (B-1/B-2)
Sub-task	Notes	Impact
2-A Extend DSL counters	Methods already drafted in roadmap—ensure default params cover new/learned counts.	shared-test module

Build remains green; JSON asset coupling is now gone.

3 Refactor UI / integration tests for DI (B-3)
Sub-task	Implementation strategy	Impact
3-A Introduce Hilt test modules	Annotate @TestInstallIn replacements for LearningRepository, LlmService, TtsService. These return in-memory fakes using the fixtures DSL.	New di-test source-set.
3-B MainViewModelTest rewrite	Use app.cash.turbine to assert
Loading → Prompt(initial) flow; stub saveQueues() verification matches roadmap B-3 bullets.	MainViewModelTest
3-C SessionIntegrationTest trim	Replace long JSON strings with queuesFixture() and inject fakes so it no longer touches assets or real persistence.	SessionIntegrationTest, EndToEndCoreSequenceIntegrationTest
3-D Library updates	• androidx.test:core-ktx + androidx.test:rules for Hilt instrumentation.
• app.cash.turbine (Flow testing).	build.gradle(:app) test deps

