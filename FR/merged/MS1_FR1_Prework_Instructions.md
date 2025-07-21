
# MS1_FR1 – Pre‑Work Implementation Guide (Code‑First)

**Source of truth:** current `lango‑mvp.zip` codebase  
**Target state:** repository ready to start Feature Request **MS1_FR1 “Bootstrap & Persist Queues”**

---

## 1  Learning Repository

| Gap ID | Current code (before) | Spec goal (after) | Required action |
|--------|----------------------|-------------------|-----------------|
| **G‑1** | `LearningRepositoryImpl.kt` constructor is truncated & doesn’t compile | Injectable constructor taking `AssetManager`, `File` and `Json` | Replace entire file |
| **G‑2** | `loadQueues()` reads only bundled assets | Must first read `filesDir/queues`; bootstrap if missing | Implement `ensureBootstrap()` |
| **G‑3** | No mutex / atomic writes in `saveQueues()` | Guard with `Mutex`; write via temp + `ATOMIC_MOVE` | Implement `ioMutex` + `atomicWrite()` |
| **G‑4** | Returns raw model, swallows errors | Return `Result<Queues>`, surface `SerializationException` | Change signatures |
| **G‑5** | Two overloads leak `InputStream`s | Provide just: `suspend fun loadQueues()` & `saveQueues()` | Delete overloads |

### 1.1  File replacements

```kotlin
// ─── domain/src/main/kotlin/.../LearningRepository.kt
interface LearningRepository {
    suspend fun loadQueues(): Result<Queues>
    suspend fun saveQueues(queues: Queues): Result<Unit>
}
```

```kotlin
// ─── data/src/main/kotlin/.../LearningRepositoryImpl.kt
@Singleton
class LearningRepositoryImpl @Inject constructor(
    private val assetManager: AssetManager,
    private val baseDir: File,
    private val json: Json
) : LearningRepository {

    private val ioMutex = Mutex()
    private val newQueueFile get() = File(baseDir, "queues/new_queue.json")
    private val learnedQueueFile get() = File(baseDir, "queues/learned_queue.json")

    override suspend fun loadQueues(): Result<Queues> = withContext(Dispatchers.IO) {
        ioMutex.withLock {
            try {
                ensureBootstrap()
                val newItems: List<LearningItem> =
                    json.decodeFromString(newQueueFile.readText())
                val learnedItems: List<LearningItem> =
                    json.decodeFromString(learnedQueueFile.readText())
                Result.success(Queues(newItems.toMutableList(), learnedItems.toMutableList()))
            } catch (e: SerializationException) {
                Result.failure(e)
            } catch (e: IOException) {
                Result.failure(e)
            }
        }
    }

    override suspend fun saveQueues(queues: Queues): Result<Unit> = withContext(Dispatchers.IO) {
        ioMutex.withLock {
            try {
                atomicWrite(newQueueFile, json.encodeToString(queues.newQueue))
                atomicWrite(learnedQueueFile, json.encodeToString(queues.learnedPool))
                Result.success(Unit)
            } catch (e: IOException) {
                Result.failure(e)
            }
        }
    }

    private fun ensureBootstrap() {
        if (newQueueFile.exists() && learnedQueueFile.exists()) return
        copyFromAssets()
    }

    private fun copyFromAssets() {
        newQueueFile.parentFile?.mkdirs()
        assetManager.open("queues/new_queue.json").use { it.copyTo(newQueueFile.outputStream()) }
        assetManager.open("queues/learned_queue.json").use { it.copyTo(learnedQueueFile.outputStream()) }
    }

    private fun atomicWrite(target: File, text: String) {
        val tmp = File.createTempFile(target.nameWithoutExtension, ".tmp", target.parentFile)
        tmp.writeText(text)
        Files.move(tmp.toPath(), target.toPath(),
                   StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING)
    }
}
```

**Build‑Point BP‑1**

```bash
./gradlew :data:compileDebugKotlin
```

---

## 2  DI and Coroutines

Create `app/src/main/java/.../di/RepositoryModule.kt`:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    @Provides @Singleton
    fun provideLearningRepository(
        assetManager: AssetManager,
        @ApplicationContext ctx: Context,
        json: Json
    ): LearningRepository =
        LearningRepositoryImpl(assetManager, ctx.filesDir, json)
}
```

**Build‑Point BP‑2**

```bash
./gradlew :app:assembleDebug
```

---

## 3  Domain & Use‑Case Updates

* Replace obsolete overload calls:

```kotlin
// before
val queues = repo.loadQueues(inputStream).getOrThrow()

// after
val queues = repo.loadQueues().getOrThrow()
```

* Wrap in `try/catch` or `.onFailure` as needed.  
* Mark suspending and call from `viewModelScope`.

---

## 4  Tests

### 4.1  Update existing repository tests

```kotlin
@Test
fun bootstrapCopiesAssets() = runBlocking {
    tempDir.deleteRecursively()
    val result = repo.loadQueues()
    assertTrue(result.isSuccess)
    assertTrue(File(tempDir,"queues/new_queue.json").exists())
}
```

### 4.2  Add concurrent read/write test

Use `coroutineScope { launch { repo.saveQueues(q) }; launch { repo.loadQueues() } }`.

**Build‑Point BP‑3**

```bash
./gradlew :data:test
```

---

## 5  Asset Cleanup (optional)

Remove duplicate root‑level JSON assets; keep only:

```
app/src/main/assets/queues/new_queue.json
app/src/main/assets/queues/learned_queue.json
```

---

## 7  Summary Checklist

| BP | Command | Expectation |
|----|---------|-------------|
| 1 | `./gradlew :data:compileDebugKotlin` | Compiles |
| 2 | `./gradlew :app:assembleDebug` | App builds |
| 3 | `./gradlew :data:test` | All repository tests green |
