### Step 1: Define & Expose Core Settings  
**Goal:** Centralize all thresholds in a single config.  
1. **Create** `SessionConfig` in domain (`domain/src/main/kotlin/com/example/domain/config/SessionConfig.kt`)  
2. **Provide** via Hilt `ConfigModule` (`app/src/main/java/com/example/lango_coach_android/di/ConfigModule.kt`)  
3. **Define** BuildConfig fields in `app/build.gradle.kts`  

> All code must reference `SessionConfig`—no literals.  
> **See SM_REFACTOR_SPEC §1** “Must‑Be‑True Invariants & Core Domain Model”

Not at all—I just had to dig back into the spec and current-build files so I could answer with the precision you need. Below is a **single-source guide** the team can lift straight into implementation for **Step 1: Define & Expose Core Settings**.

# CONTEXT

## 1  Threshold values & defaults

| Setting               | Authoritative default | Rationale                                                                               | Variant overrides                                                                 |
| --------------------- | --------------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `requiredUsages`      | **3**                 | Matches the learning invariant “use every new target three times before it’s mastered”  | Keep identical across build types; tests can inject lower numbers via constructor |
| `repeatModeThreshold` | **2**                 | FSM only enters *Repeat-after-me* after two consecutive failures                        | Same for all flavours                                                             |
| `maxRepeatLoops`      | **3**                 | Safety-net that caps the remediation loop                                               | Same for all flavours                                                             |

*Why no per-flavour tweaks?* These numbers encode pedagogy, not environment. If you ever need to shorten loops for demo/E2E tests, do it by injecting an **alternative `SessionConfig` in the test harness**, not by Gradle variant.

---

## 2  `SessionConfig` API shape

```kotlin
package com.example.domain.config

/**
 * Immutable runtime thresholds for a coaching session.
 */
@JvmInline value class PositiveInt(val value: Int) { init { require(value > 0) } }

data class SessionConfig(
    val requiredUsages: PositiveInt = PositiveInt(3),
    val repeatModeThreshold: PositiveInt = PositiveInt(2),
    val maxRepeatLoops: PositiveInt = PositiveInt(3),
)
```

*Location* `domain/src/main/kotlin/com/example/domain/config/SessionConfig.kt` — aligns with plan .

*Why only three fields?* Step 1’s goal is **centralisation**, not exhaustiveness. ASR time-outs or UI caps are orthogonal concerns and will get their own config later (likely in `speech` and `ui` modules).

---

## 3  Gradle `BuildConfig` fields

Add once in **`app/build.gradle.kts`** so Android code can read constants before DI spins up:

```kotlin
android {
    ...
    defaultConfig {
        ...
        buildConfigField("int", "REQUIRED_USAGES", "3")
        buildConfigField("int", "REPEAT_MODE_THRESHOLD", "2")
        buildConfigField("int", "MAX_REPEAT_LOOPS", "3")
    }
}
```

*No debug/release overrides*—we intentionally keep pedagogy stable. Unit tests will bypass `BuildConfig` entirely by constructing custom `SessionConfig`s.

---

## 4  Hilt wiring

```kotlin
// app/src/main/java/com/example/lango_coach_android/di/ConfigModule.kt
@Module
@InstallIn(SingletonComponent::class)
object ConfigModule {

    @Provides @Singleton
    fun provideSessionConfig(): SessionConfig = SessionConfig(
        requiredUsages      = PositiveInt(BuildConfig.REQUIRED_USAGES),
        repeatModeThreshold = PositiveInt(BuildConfig.REPEAT_MODE_THRESHOLD),
        maxRepeatLoops      = PositiveInt(BuildConfig.MAX_REPEAT_LOOPS),
    )
}
```

*Scope* `@Singleton` – thresholds are immutable for the lifetime of the process.

*Qualifiers* None for now; if we ever introduce, say, `SpeechConfig`, use `@Named("speech")` at that time.

---

## 5  Consuming-code conventions

* **Domain layer** – `StateMachineRunner(spec, sessionConfig)` via constructor injection.
* **UI layer** – `MainViewModel` already injects `SessionConfig` in the upcoming refactor .
* **Tests** – create ad-hoc configs:

```kotlin
val cfg = SessionConfig(PositiveInt(1), PositiveInt(1), PositiveInt(1))
runner = StateMachineRunner(spec, cfg)
```

No extra `ConfigProvider` interface—the data class itself is the contract.

---

## 6  Versioning & evolution

* **Queue/persistence JSON** already contains `schemaVersion` .
* **Runtime thresholds** don’t require their own version; when a new field lands we’ll:

  1. Add an optional property with a sensible default in `SessionConfig`.
  2. Surface a matching `buildConfigField`.
  3. Update `ConfigModule` injectors.

---

### Implementation checklist (copy into PR description)

1. **Add file** `domain/.../SessionConfig.kt` with defaults & `PositiveInt` guard.
2. **Gradle** – declare three `buildConfigField` ints in `app/build.gradle.kts`.
3. **Create DI module** `ConfigModule.kt` in `app` as shown.
4. **Refactor usages** – replace magic numbers with `sessionConfig.*` in FSM guard logic (spec §2.3 forbids literals ).
5. **Unit test** – golden-path test injects default config (spec §4.1 ).
6. **CI** – ensure `./gradlew :domain:test` and `:app:compileKotlin` pass.

Follow this and you’ll hit **first-compile green** for PR #1 without surprises.

---
## IMPACT CONTEXT

Below is a **point-by-point “ready-to-implement” clarification** for **Step 1 – Define & Expose Core Settings**. I have cross-checked every answer against the current codebase (**MY\_CB**), the refactor spec (**TDD\_F / SM\_REFACTOR\_SPEC**), the implementation plan (**SM\_REFACTOR\_PLAN**), and the state-machine spec (**ARCH\_TL**).

---

## 1  Kotlin @JvmInline value classes

| Item                 | Status in our tool-chain                                                | Action                                   |
| -------------------- | ----------------------------------------------------------------------- | ---------------------------------------- |
| Kotlin plugin        | **2.2.0** (root `build.gradle.kts`)                                     | Fully supports `@JvmInline value class`. |
| JVM target           | **17** everywhere (`kotlin { compilerOptions { jvmTarget = JVM_17 } }`) | OK.                                      |
| Extra compiler flags | **None required** since Kotlin 1.5                                      | Nothing to change.                       |

**Bottom-line:** the `PositiveInt` inline wrapper from *SM\_REFACTOR\_CURRENT\_TASK* will compile out-of-the-box; no fallback to a data class is necessary.

---

## 2  Domain-module package & source-set

*Current layout*

```groovy
domain/
  src/main/java/…       // configured as both java & kotlin root
  src/main/kotlin/…     // NOT in sourceSets
```

*Proposed file*

```
domain/src/main/kotlin/com/example/domain/config/SessionConfig.kt
```

*Fix*: **add** the `kotlin` dir to the existing sourceSet to avoid moving every package:

```kotlin
android {
    // …
    sourceSets["main"].kotlin.srcDirs(
        "src/main/java",
        "src/main/kotlin"      // ← add this line once
    )
}
```

Alternatively, drop the new file under `src/main/java/.../config/`.  Either approach is fine, but **updating the sourceSet once** is cleaner and unblocks future Kotlin-only files.

---

## 3  BuildConfig fields

*Add in `app/build.gradle.kts` (already has `buildFeatures { buildConfig = true }`):*

```kotlin
defaultConfig {
    // …
    buildConfigField("int", "REQUIRED_USAGES",        "3")
    buildConfigField("int", "REPEAT_MODE_THRESHOLD",  "2")
    buildConfigField("int", "MAX_REPEAT_LOOPS",       "3")
}
```

*Behaviour*

* They land in `com.example.lango_mvp_android.BuildConfig` exactly once per variant.
* The **domain module never touches them directly**; we read them in the new `ConfigModule` and map into `SessionConfig`.
* No flavour logic today → zero CI cache churn.

---

## 4  Hilt discovery for `ConfigModule`

*Directory choice:* follow existing pattern under `app/.../di`.  Example:

```kotlin
package com.example.lango_mvp_android.di

@Module
@InstallIn(SingletonComponent::class)
object ConfigModule {
    @Provides @Singleton
    fun provideSessionConfig(): SessionConfig = SessionConfig(
        requiredUsages      = PositiveInt(BuildConfig.REQUIRED_USAGES),
        repeatModeThreshold = PositiveInt(BuildConfig.REPEAT_MODE_THRESHOLD),
        maxRepeatLoops      = PositiveInt(BuildConfig.MAX_REPEAT_LOOPS),
    )
}
```

*Hilt will see this automatically*—no extra qualifiers or component tweaks are needed.

---

## 5  “Don’t touch legacy orchestrator” guard-rails

* Code freeze: simply **do not import `SessionConfig` anywhere except the new module** in this PR.
* Optional CI safety net: add a Danger-rule or Gradle convention that fails the build if `domain/src/main/kotlin/com/example/domain/*UseCase.kt` changes while the `legacy‐lock` flag is set.  Over-kill for Step 1, but easy: a tiny Kotlin script under `buildSrc/` that scans changed paths.
* Soft reminder: mark `CoachOrchestrator*` classes `@Deprecated("Will be removed in Step 5 of SM_REFACTOR_PLAN")` to scare off IDE auto-refactors.

---

## 6  Injection impact on existing tests

*Step 1 adds but does not **use** `SessionConfig` yet.*
All unit/integration tests currently create classes via constructors, not via Hilt, so nothing breaks.

If you want 100 % safety, add a test helper:

```kotlin
object TestConfigs {
    fun default() = SessionConfig()  // uses baked-in defaults
}
```

…but you won’t need it until Step 2.

---

## 7  Shared-test fixtures for future FSM tests

Add once under `shared-test/src/main/kotlin/com/example/testing/TestFixtures.kt`:

```kotlin
fun sessionConfig(
    required: Int = 3, repeatThresh: Int = 2, maxLoops: Int = 3
) = SessionConfig(
    PositiveInt(required),
    PositiveInt(repeatThresh),
    PositiveInt(maxLoops)
)
```

That gives Step 2’s golden-path test (`runner = StateMachineRunner(spec, sessionConfig(3,2,3))`) a ready utility.

---

## 8  JSON persistence schemaVersion

The spec’s new `schemaVersion` key is **only enforced in Step 4 (“Persist mastered items”)**.
Adding `SessionConfig` now **does not touch queue JSON**—no migration or asset tweak needed this round.

---

## 9  CI tasks to watch after the config PR

| Stage       | CI task                           | Why it matters                                                               |
| ----------- | --------------------------------- | ---------------------------------------------------------------------------- |
| Compilation | `./gradlew :domain:compileKotlin` | catches source-set mis-wires.                                                |
| Tests       | `./gradlew :domain:test`          | fails fast if we accidentally reference `SessionConfig` somewhere un-mocked. |
| App compile | `./gradlew :app:compileKotlin`    | validates BuildConfig field generation + Hilt graph.                         |

Set a lightweight GitHub-Actions workflow to run just those three jobs on the PR; they cover the 90 % compile / 80 % test “first-green” metric.

---

## 10  Documentation alignment

*For now* leave the YAML spec comments as-is; but add a **single inline NOTE**:

```yaml
context:
  required_usages: 3            # ← default, overridden via SessionConfig
```

…so future diff readers know the source of truth is `SessionConfig`.  We will fully parameterise the YAML when we wire the runner in Step 2.

---

### Quick implementation punch-list (copy into your PR)

1. **Add** `SessionConfig.kt` (with `PositiveInt`) under `domain/.../config/`.
2. **Update** domain sourceSet to include `src/main/kotlin`.
3. **Insert** three `buildConfigField` ints in `app/build.gradle.kts`.
4. **Create** `ConfigModule.kt` providing a singleton `SessionConfig`.
5. **(Optional)** annotate legacy orchestrator classes `@Deprecated`.
6. **Verify** CI (`:domain:compileKotlin`, `:domain:test`, `:app:compileKotlin`) is green.

Follow this and **PR #1 will meet the “first-compile green” target with zero surprise regressions**.
