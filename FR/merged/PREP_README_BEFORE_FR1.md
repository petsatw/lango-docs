# Lango MVP — **Prep Guide before Feature‑1 (Bootstrap & Persist Queues)**  
*Full migration to **kotlinx.serialization** and **Hilt** with built‑in safety checkpoints.*

---

## 1  Why this document exists  
The current `dev` branch still relies on **Gson**, offers **no Hilt setup**, and stores queue assets under legacy names/paths.  
Feature‑1 (FR‑1) requires:  

1. A single JSON engine (`kotlinx.serialization`).  
2. Dependency Injection via **Hilt** (for `Json`, repository paths, etc.).  
3. Asset rename & relocation.  
4. A smoke‑test scaffold **disabled** until FR‑1 completes.  

This README is the *only* script you need to execute before handing the codebase to the FR‑1 team.

---

## 2  Migration road‑map at a glance  

| Phase | Goal | Risk level | Build Checkpoint |
|-------|------|-----------|------------------|
| **0** | Tag a baseline snapshot | none | `./gradlew build` green |
| **1** | Introduce version‑catalog, serialization & Hilt plugins (no prod code) | ★☆☆ | **BC‑1** |
| **2** | Swap to `kotlinx.serialization` **and** stand‑up Hilt graph | ★★☆ | **BC‑2** |
| **3** | Rename/move assets, extend repository ctor, add disabled smoke test | ★★★ | **BC‑3** |

Each Build Checkpoint (BC) must pass before merging the phase PR.

---

> **BC‑0**  `./gradlew build` must succeed exactly as on `dev`.

---
## 4  Phase 1 — Build & dependency groundwork *(no production code touched)*  

> **Goal:** make the tool-chain ready for Hilt **without** yet wiring it into modules, and ensure we *reuse* every version that already exists in `gradle/libs.versions.toml`.

---

### 4.1  Extend the *version-catalog*  

Open **`gradle/libs.versions.toml`** and **append** (do *not* duplicate) the following blocks:

```toml
[versions]                 # --- additions ---
hilt = "2.56.1"            # latest stable as of July 2025

[libraries]                # --- additions ---
hilt-android  = { group = "com.google.dagger", name = "hilt-android",  version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }
```

4.2 Register the Hilt Gradle plugin
In build.gradle.kts at the project root (the file that already lists AGP, Kotlin, Secrets, …) add the plugin in the same style:

```kotlin
plugins {
    id("com.android.application")                                   version "8.6.0" apply false
    id("org.jetbrains.kotlin.android")                              version "2.2.0" apply false
    id("org.jetbrains.kotlin.plugin.serialization")                 version "2.2.0" apply false
    id("com.google.android.libraries.mapsplatform.secrets-gradle-plugin") version "2.0.1" apply false

    /* NEW ↓ — Hilt only registered, not yet applied in any module */
    id("com.google.dagger.hilt.android")                            version "2.56.1" apply false
}
```
No module applies the plugin yet; that happens in Phase 2 when we start injecting the Json singleton.

4.3 No module changes (yet)
Because the plugin is only registered, IDE sync remains green and the build has zero code churn at this stage.

Build-checkpoint BC-1:
./gradlew help finishes without new warnings; IDE import succeeds.

## 5  Phase 2 — Hilt foundation + JSON engine swap  

### 5.1  Bootstrap Hilt  

| File | New / Modified | Snippet |
|------|----------------|---------|
| `app/src/.../LangoApplication.kt` | **new** | `@HiltAndroidApp class LangoApplication : Application()` |
| `:app/build.gradle.kts` | **mod** | `apply plugin: "com.google.dagger.hilt.android"`<br>`kapt { correctErrorTypes = true }` |
| `di/JsonModule.kt` | **new** | see code block below |

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object JsonModule {
    @Provides
    fun provideJson(): Json = Json { ignoreUnknownKeys = true; prettyPrint = true }
}
```

### 5.2  Annotate DTOs  

Add `@Serializable` to every data class crossing a serialization boundary (usually in `:domain`).

### 5.3  Replace Gson usages  

```diff
-class LearningRepositoryImpl(...){
-    private val gson = Gson()
-    ...
-    gson.fromJson<Queues>(reader, typeToken)
+class LearningRepositoryImpl @Inject constructor(private val json: Json, ...){
+    json.decodeFromString<Queues>(text)
 }
```

Remove `implementation("com.google.code.gson:gson")` from all modules.

> **BC‑2**  `./gradlew :data:testDebugUnitTest` passes; app launches and basic flows still work.

---

## 6  Phase 3 — Asset migration, constructor refactor & smoke test  

### 6.1  Rename & move assets  

| Old path | New path |
|----------|----------|
| `app/src/main/assets/core_blocks.json` | `app/src/main/assets/queues/new_queue.json` |
| `app/src/main/assets/learned_queue.json` | `app/src/main/assets/queues/learned_queue.json` |

Update hard‑coded strings accordingly.

### 6.2  Extend repository constructor  

```kotlin
class LearningRepositoryImpl @Inject constructor(
    private val json: Json,
    private val assetManager: AssetManager,
    private val baseDir: Path        // prod: context.filesDir.toPath()
) : LearningRepository { /* body unchanged for prep */ }
```

Provide `baseDir` in a new `PathModule`.

### 6.3  Add **disabled** smoke test  

`data/src/test/java/.../BootstrapSmokeTest.kt`:

```kotlin
@Disabled("Enable after FR‑1 merges")
class BootstrapSmokeTest { /* template from README */ }
```

### 6.4  Workflow for the annotation  

| Phase | Annotation state |
|-------|------------------|
| Prep PR | `@Disabled` (CI green) |
| FR‑1 branch | Remove or flip to `@EnabledOnCi` |
| PR gate | Test must run & pass |

> **BC‑3**  Full `./gradlew build` green with smoke test ignored.

---

## 7  Dependency ledger  

| Action | Library | Gradle notation |
|--------|---------|-----------------|
| **ADDED** | Kotlinx Serialization | `libs.kotlinx.serialization.json` |
| **ADDED** | Hilt Runtime | `libs.hilt.android` |
| **ADDED** | Hilt Compiler (kapt) | `libs.hilt.compiler` |
| **REMOVED** | Gson | `com.google.code.gson:gson` |
| **UNCHANGED** | Kotlin Coroutines | `org.jetbrains.kotlinx:kotlinx-coroutines-core` |

---

## 8  New / modified classes overview  

| Module | Class | Status | Purpose |
|--------|-------|--------|---------|
| `:app` | `LangoApplication` | **new** | Boots Hilt |
| `:app` | `JsonModule` | **new** | Provides singleton `Json` |
| `:app` | `PathModule` | **new** | Provides `Path` for persistence |
| `:data` | `LearningRepositoryImpl` | **mod** | Swapped to `Json`, new ctor |
| `:data:test` | `BootstrapSmokeTest` | **new** | Disabled guard test |

---

## 9  Verification checklist (must be green before hand‑over)  

- [ ] BC‑1 passes (Gradle sync & build)  
- [ ] BC‑2 passes (unit tests, app launch)  
- [ ] Assets live under `assets/queues/`  
- [ ] Gson *completely* absent from dependency graph (`./gradlew :app:dependencies | grep gson` returns nothing)  
- [ ] BC‑3 passes (CI green, smoke test disabled)  

When all boxes are checked, tag `prep-complete‑$(date +%Y%m%d)` and merge to `main`. 🎉

---

### Appendix A  — Gradle commands cheat‑sheet  

```bash
# Clean & verify all tests
./gradlew clean build

# Check dependency tree for leftovers
./gradlew :app:dependencies --configuration releaseRuntimeClasspath
```

---

**End of Prep Guide**  
