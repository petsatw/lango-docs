
# Pre-Build Check: FR “Bootstrap & Persist Queues”

## Summary
The current build is **architecturally ready**, but two critical gaps block implementation of **FR “Bootstrap & Persist Queues”**:

| Gap | Impact | Resolution (high-level) |
|-----|--------|-------------------------|
| **No bootstrap logic** – `LearningRepositoryImpl.loadQueues()` assumes the two queue files already exist. | Crash on first run. | Copy bundled JSON from `assets/queues` into `context.filesDir` the very first time the app starts, then parse. |
| **No atomic, thread-safe persistence** – `saveQueues()` is a straight overwrite and is not protected from concurrent `loadQueues()` calls. | Possible data-loss & race conditions. | Guard all disk I/O with a shared `Mutex`; write to a temp file and `Files.move(…, ATOMIC_MOVE)`. |

---

## 1. Pre-Work (do **before** tackling FR tasks)

| # | Change | Exact Location | Details |
|---|--------|----------------|---------|
| **PW-1** | **Add bundled assets** | `:app/src/main/assets/queues/new_queue.json` & `learned_queue.json` | Copy the JSON files supplied in the repo (see `/assets/uploads`). Keep snake-case property names. |
| **PW-2** | **Ensure Kotlinx-Serialization is configured** | `build.gradle(:data)` | ```groovy
plugins {
    id("org.jetbrains.kotlin.plugin.serialization") version "2.0.0"
}
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.0")
}
``` |
| **PW-3** | **Expose Application Context in DI** | `:di/DataModule.kt` | ```kotlin
@Provides
fun provideLearningRepository(ctx: Application): LearningRepository =
    LearningRepositoryImpl(ctx)
``` |
| **PW-4** | **Introduce queue data models mapped for serialization** | `:domain/model/Queues.kt` | Annotate all snake-case JSON fields with `@SerialName` so parsing survives naming mismatches. |

---

## 2. FR Implementation Tasks

### 2.1 Repository refactor (`LearningRepositoryImpl` – :data module)

| # | Task | Code Snippet / Algorithm |
|---|------|--------------------------|
| **T-1** | **Add class-level members** | ```kotlin
private val ioMutex = Mutex()
private val json = Json {
    ignoreUnknownKeys = true
    namingStrategy = JsonNamingStrategy.SnakeCase
}
private val newQueueFile = File(filesDir, "new_queue.json")
private val learnedQueueFile = File(filesDir, "learned_queue.json")
``` |
| **T-2** | **Bootstrap helper** | ```kotlin
private suspend fun ensureQueuesExist() = withContext(Dispatchers.IO) {
    if (!newQueueFile.exists() || !learnedQueueFile.exists()) {
        copyAsset("queues/new_queue.json", newQueueFile)
        copyAsset("queues/learned_queue.json", learnedQueueFile)
    }
}
``` |
| **T-3** | **`loadQueues()`** | ```kotlin
return runCatching {
    ioMutex.withLock {
        ensureQueuesExist()
        val newQ: List<LearningItem> = json.decodeFromString(newQueueFile.readText())
        val learned: List<LearningItem> = json.decodeFromString(learnedQueueFile.readText())
        Queues(newQ.toMutableList(), learned.toMutableList())
    }
}.recoverCatching {
    // malformed → restore defaults
    restoreFromAssets()
}
``` |
| **T-4** | **`saveQueues()` (atomic)** | ```kotlin
suspend fun save(queues: Queues): Result<Unit> = runCatching {
    ioMutex.withLock {
        atomicWrite(newQueueFile) {
            it.writeText(json.encodeToString(queues.newQueue))
        }
        atomicWrite(learnedQueueFile) {
            it.writeText(json.encodeToString(queues.learnedPool))
        }
    }
}
``` |
| **T-5** | **Unit-test race-safety** | Implement tests FS-1‒FS-5 from FR doc using `runBlocking` & `TestCoroutineDispatcher`. |

---

## 3. Ordering of Key Changes

1. **KEY_CHANGE_1** – *Add Kotlinx-Serialization + snake-case data models* (PW-2, PW-4).  
2. **KEY_CHANGE_2** – *Introduce `Mutex` + atomic file utilities* (T-1, T-4).

> Implement KEY_CHANGE_1 **first** to ensure serialization logic compiles and integrates cleanly. KEY_CHANGE_2 then builds safe concurrency atop that.

---

## 4. Fundamental Gaps Check

| Required Fundamental | Present? | Action |
|----------------------|----------|--------|
| Bundled default JSON in assets | **No** | Add via **PW-1** |
| Serialization library | Partial (models exist but plugin missing) | **PW-2** |
| Thread-safe file I/O helpers | **No** | Implement `atomicWrite`, `ioMutex` (T-1, T-4) |

---

## 5. Documentation Updates

| Doc | Section | Update |
|-----|---------|--------|
| **Architecture.md** | Data Layer description | Note bootstrap copy & atomic persist logic. |
| **BUILD SPEC.md** | Dependencies list | Add `kotlinx-serialization-json:1.7.0`; mention `java.nio.file` ATOMIC_MOVE requirement (API 26+ already satisfied). |
| **TDD.md** | Unit-Test list | Append FR test IDs FS-1 through FS-5. |
| **README** | “First Launch” steps | Note that default queues are now auto-copied on install. |

---

## 6. Dependency Risk

None detected: `kotlinx-serialization-json:1.7.0` is **new** and does not conflict with existing dependencies.

---

## Build Points

### 🔹 After PW-1: Assets in place
- ✅ Confirm `new_queue.json` and `learned_queue.json` exist in `:app/src/main/assets/queues/`
- ✅ Verify file format matches expected `LearningItem` schema with snake_case

### 🔹 After PW-2: Serialization plugin and dependency
- ✅ Ensure `kotlinx-serialization-json` 1.7.0 compiles and doesn’t conflict with existing dependencies
- ✅ Verify plugin `kotlin("plugin.serialization")` active and effective in `:data` module

### 🔹 After PW-4: Domain models
- ✅ Confirm all fields use `@SerialName("snake_case_name")`
- ✅ Round-trip test (encode → decode → encode) maintains data integrity

### 🔹 After T-1: Repository-level file and mutex declarations
- ✅ Static analysis: check visibility, mutability, thread-safety
- ✅ Validate `json` configuration includes SnakeCase naming strategy

### 🔹 After T-2: Bootstrap logic
- ✅ Validate fallback only triggers once
- ✅ Confirm restored files are byte-equal to assets on first launch

### 🔹 After T-3: loadQueues() logic
- ✅ Confirm returned `Queues` has populated `.newQueue` and `.learnedPool`
- ✅ Simulate missing/corrupt JSON and ensure recovery with `restoreFromAssets()`

### 🔹 After T-4: saveQueues()
- ✅ Ensure atomic file write creates no partials
- ✅ File contents must match round-tripped encoded JSON from in-memory `Queues`

### 🔹 After T-5: Test coverage
- ✅ Implement unit test `FS-1` through `FS-5` as per `MS1_FR1.md`
- ✅ Run load/save concurrently (e.g. `FS-3`) to validate `Mutex` effectiveness

### 🔹 After Documentation Updates
- ✅ Confirm all touched files have updated dependency info, architecture notes, and test plans
