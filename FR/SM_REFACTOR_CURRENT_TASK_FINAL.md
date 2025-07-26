
### Step 1: Define & Expose Core Settings
**Goal:** Centralize all thresholds in a single config.

---

# CONTEXT

## 1. Threshold values & defaults

| Setting               | Authoritative default | Rationale                                                                               | Variant overrides                                                                 |
| --------------------- | --------------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `requiredUsages`      | **3**                 | Matches the learning invariant “use every new target three times before it’s mastered”. | Tests can inject lower numbers via constructor injection.                          |
| `repeatModeThreshold` | **2**                 | FSM enters *Repeat‑after‑me* after two consecutive failures.                             | Same across build types.                                                          |
| `maxRepeatLoops`      | **3**                 | Safety‑net that caps the remediation loop.                                               | Same across build types.                                                          |

> **Note:** These values encode core pedagogy. Environment‑specific tweaks (e.g., for demos or E2E tests) should be provided by injecting an alternative `SessionConfig` in test harnesses.

### Impact Context Integrated
- **Kotlin inline support:** Project uses Kotlin plugin **2.2.0** and targets JVM **17**, which fully supports `@JvmInline value class` without extra compiler flags.  
- **Documentation alignment:** The YAML spec (`session.yaml`) should retain its `required_usages: 3` comment, annotated with a NOTE that the true source of defaults is `SessionConfig`.

---

## 2. `SessionConfig` API shape

```kotlin
package com.example.domain.config

/**
 * Immutable runtime thresholds for a coaching session.
 */
@JvmInline value class PositiveInt(val value: Int) {
  init { require(value > 0) }
}

data class SessionConfig(
    val requiredUsages:      PositiveInt = PositiveInt(3),
    val repeatModeThreshold: PositiveInt = PositiveInt(2),
    val maxRepeatLoops:      PositiveInt = PositiveInt(3),
)
```

- **File location:** `domain/src/main/kotlin/com/example/domain/config/SessionConfig.kt`
- **SourceSet update:** Ensure `domain` module’s `sourceSets["main"].kotlin.srcDirs` includes `"src/main/kotlin"` so this Kotlin file is compiled.

---

## 3. Gradle `BuildConfig` fields

Add to **`app/build.gradle.kts`** under `defaultConfig`:

```kotlin
buildConfigField("int", "REQUIRED_USAGES",        "3")
buildConfigField("int", "REPEAT_MODE_THRESHOLD",  "2")
buildConfigField("int", "MAX_REPEAT_LOOPS",       "3")
```

- **Prerequisite:** `buildFeatures { buildConfig = true }` is already enabled.  
- **Behavior:** Exposes constants in `BuildConfig` for `ConfigModule` to read. No debug/release overrides are needed to maintain pedagogical stability.

---

## 4. Hilt wiring

```kotlin
// app/src/main/java/com/example/lango_coach_android/di/ConfigModule.kt
package com.example.lango_coach_android.di

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

- **Discovery:** Hilt auto‑detects modules under `app/.../di`.  
- **Scope:** `@Singleton` suffices for immutable thresholds.  
- **Qualifiers:** None needed now; named qualifiers can be added later (e.g., for a `SpeechConfig`).

---

## 5. Consuming‑code conventions & tests

- **Domain code:** Inject `SessionConfig` (e.g., in `StateMachineRunner(spec, sessionConfig)`).  
- **UI layer:** `MainViewModel` will inject via Hilt.  
- **Legacy guard‑rails:** Avoid importing `SessionConfig` into old orchestrator/use‑case classes until refactor is complete; consider marking those classes `@Deprecated` to discourage accidental use.  
- **Tests impact:** Existing unit tests construct `SessionConfig` manually (no Hilt), so none will break. Optionally add a test helper:

  ```kotlin
  object TestConfigs {
    fun default() = SessionConfig()
  }
  ```

- **Shared‑test fixtures:** Future FSM tests can use:

  ```kotlin
  fun sessionConfig(
    required: Int = 3, repeatThresh: Int = 2, maxLoops: Int = 3
  ) = SessionConfig(
      PositiveInt(required),
      PositiveInt(repeatThresh),
      PositiveInt(maxLoops)
  )
  ```

---

## 6. Versioning & evolution; CI checklist

- **JSON schemaVersion:** Unaffected by adding `SessionConfig`.  
- **Future fields:** Add optional properties with defaults, matching `buildConfigField` and `ConfigModule` updates.  
- **CI tasks (PR step 1):**
  1. `./gradlew :domain:compileKotlin`  – catches sourceSet issues.  
  2. `./gradlew :domain:test`           – ensures no unexpected references.  
  3. `./gradlew :app:compileKotlin`     – validates BuildConfig and Hilt graph.  

---

# Implementation Checklist (copy into PR description)

1. **Add** `SessionConfig.kt` with `PositiveInt` guard under `domain/.../config/`.  
2. **Update** domain `sourceSets` to include `src/main/kotlin`.  
3. **Insert** three `buildConfigField` ints in `app/build.gradle.kts`.  
4. **Create** `ConfigModule.kt` providing singleton `SessionConfig`.  
5. **Deprecate** legacy orchestrator classes to prevent accidental usage.  
6. **Verify** CI (`:domain:compileKotlin`, `:domain:test`, `:app:compileKotlin`) is green.
