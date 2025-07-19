# Lango MVP – **Prep Guide before F‑1 (Bootstrap & Persist Queues)**

This is the *single* README your platform team should follow **before** they hand the codebase to the Feature‑1 developers.  
It removes hidden dependency pitfalls and seeds a disabled smoke test that FR‑1 will later flip green.

---

## 1  Why this prep is needed

* The current `dev` branch still references **Gson** as the JSON engine in multiple specs fileciteturn5file2, yet the Milestone‑1 context requires **kotlinx.serialization** for forward‑compatible, multiplatform parsing fileciteturn5file1.  
* No Gradle **Version Catalog** entries exist for Kotlinx libs (catalog file missing in repository snapshot).  
* A blocking smoke test should exist but remain disabled so CI stays green until FR‑1 merges.

---

## 2  Dependency housekeeping (Version Catalog + modules)

> **Action:** open `gradle/libs.versions.toml`.  
> If the file does **not** exist, create it exactly at that path; if it exists, **search first** for each alias below before adding to avoid duplicates.

```toml
[versions]                      # add keys only if they don't already exist
kotlinx-serialization = "1.6.3"
kotlinx-coroutines    = "1.9.0"
robolectric           = "4.11.1"

[libraries]                     # idem
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinx-serialization" }
kotlinx-coroutines-test  = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test",  version.ref = "kotlinx-coroutines" }
robolectric              = { group = "org.robolectric",      name = "robolectric",              version.ref = "robolectric" }

[plugins]                       # if kotlin-serialization already present, keep its existing version
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version = "1.9.23" }
```

### 2.1  Remove Gson (optional but recommended)

1. Search workspace for `implementation("com.google.code.gson`  
2. If found and **not** needed elsewhere, delete the line.  
3. If another module still requires Gson, leave it — Kotlinx and Gson can coexist; just avoid duplicate JSON engines in **data** module.

### 2.2  Apply the serialization compiler plugin

In every module that serialises JSON (`:data`, `:domain` if models live there):

```kotlin
plugins {
    id("kotlin-android")
    id("kotlin-serialization")   // <- add (one line)
}
```

### 2.3  Add library coordinates to `data/build.gradle.kts`

```kotlin
dependencies {
    implementation(libs.kotlinx.serialization.json)

    testImplementation(libs.kotlinx.coroutines.test)
    testImplementation(libs.robolectric)   // for FS‑1 and FS‑2
}
```

> **Safeguard:** if the alias already resolves (Gradle will warn about duplicates), remove duplicate lines instead of ignoring the warning.

---

## 3  Provide a singleton `Json` instance via Hilt

Create `di/JsonModule.kt`:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object JsonModule {
    @Provides
    fun provideJson(): Json = Json {
        ignoreUnknownKeys = true
        prettyPrint       = true
    }
}
```

No other modules are modified.

---

## 4  Introduce a **disabled** BootstrapSmokeTest

### 4.1  File path

```
data/src/test/java/com/example/data/BootstrapSmokeTest.kt
```

### 4.2  Template (JUnit 5)

```kotlin
@org.junit.jupiter.api.Disabled("Enable after FR‑1 merges")
class BootstrapSmokeTest {

    private val context = ApplicationProvider.getApplicationContext<Application>()
    private val json    = Json { ignoreUnknownKeys = true }

    @Test
    fun bootsAndSaves() = runTest {
        val repo = LearningRepositoryImpl(
            json          = json,
            assetManager  = context.assets,
            baseDir       = context.filesDir.toPath()     // constructor will be added in FR‑1
        )

        val q1 = repo.loadQueues().getOrThrow()
        q1.newQueue.first().presentationCount++

        repo.saveQueues(q1).getOrThrow()

        val q2 = repo.loadQueues().getOrThrow()
        assertEquals(1, q2.newQueue.first().presentationCount)
    }
}
```

*If the project still uses JUnit 4, replace the `@Disabled` import with `org.junit.Ignore`.*

### 4.3  Workflow guidance

| Phase | What to do with the annotation |
|-------|--------------------------------|
| **Prep PR** | This phase. Keep `@Disabled` so CI passes. |
| **FR‑1 branch** | Remove the annotation (or change to `@EnabledOnCi`) when repository code is complete. |
| **PR review gate** | CI must now execute and pass BootstrapSmokeTest; reviewers confirm the annotation is gone. |

---

## 5  Asset renaming & relocation

1. **Rename** `app/src/main/assets/core_blocks.json` → `new_queue.json`.  
2. **Move** both `new_queue.json` and `learned_queue.json` into `app/src/main/assets/queues/`.  
3. Search and replace any hard‑coded `"core_blocks.json"` or `"learned_queue_example.json"` strings; prefer the pathless names (`new_queue.json`, `learned_queue.json`) because `AssetManager.open()` already looks under assets root.

---

## 6  Constructor refactor sketch (for FR‑1 devs)

> Not implemented in prep stage, but smoke test constructor call shows the intended signature:

```kotlin
class LearningRepositoryImpl(
    private val json: Json,
    private val assetManager: AssetManager,
    private val baseDir: Path               // <— injectable for JVM tests
) : LearningRepository { ... }
```

*Prep team should **only** add the parameter to the primary constructor and update Hilt providers; the body can remain a stub. This avoids a second DI change during FR‑1.*

---

## 7  Verification checklist before handing to FR‑1

| ✔ | Item |
|---|------|
| libs.versions.toml contains **no duplicate** keys after edits |
| `data` module sees `kotlin-serialization` plugin in Gradle sync |
| `JsonModule` compiles and Hilt generates code |
| Assets renamed & moved under `assets/queues/` |
| BootstrapSmokeTest compiles and is **ignored** (CI green) |

Once all bullets are green, the repository is friction‑free for the feature team to implement **Bootstrap & Persist Queues**.

---

**End of Prep Guide**  
