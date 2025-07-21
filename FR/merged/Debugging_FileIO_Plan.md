# Debugging Plan – File‑Persistence Failures

*Target module: `data`*   |   *Author: ChatGPT*   |   *Date: 2025‑07‑21*

---

## 0 . Prerequisites

* **Branch:** work on a disposable feature branch.
* **Tooling:** Android Studio Flamingo or later, JDK 17.
* **Build:** `./gradlew clean assembleDebug`.

---

## 1 . Expose the Root Exception

1. Open `LearningRepositoryImplTest.kt`.
2. Locate every `saveResult.isSuccess` assertion.
3. Insert immediately **above** each assertion:

   ```kotlin
   saveResult.exceptionOrNull()?.printStackTrace()
   ```

4. Enable test‐output forwarding in *data* module:

   ```kotlin
   // data/build.gradle.kts
   tasks.withType<Test>().configureEach {
       testLogging { events("standard_out", "standard_error") }
   }
   ```

5. Re‑run the failing tests:

   ```bash
   ./gradlew :data:testDebugUnitTest --info
   ```

   *Observe* the printed stack‑trace.  
   ✔️ You now know **exactly** which exception is thrown.

---

## 2 . Micro‑Test #1 – Plain JVM

*Goal: prove whether `atomicWrite` works on a normal filesystem.*

```kotlin
class AtomicWriteJvmTest {
    private val json = Json { encodeDefaults = true }

    @Test
    fun atomicWrite_succeeds_on_JVM() {
        val tmpDir = createTempDir()
        val repo = LearningRepositoryImpl(FakeAssets(), tmpDir, json)

        val queues = Queues(mutableListOf(dummyItem()), mutableListOf())
        val result = runBlocking { repo.saveQueues(queues) }

        assertTrue(result.isSuccess)
        assertTrue(File(tmpDir, "queues/new_queue.json").exists())
    }
}
```

Run with:

```bash
./gradlew :data:test --tests AtomicWriteJvmTest
```

*Interpretation:*  
* **Green:** your code is fine on a real filesystem → problem is Robolectric.  
* **Red:** logic bug in `atomicWrite`.

---

## 3 . Micro‑Test #2 – Robolectric Fallback

*Goal: verify the fallback branch when `ATOMIC_MOVE` is unsupported.*

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [26])
class AtomicWriteRoboTest {

    private val json = Json { encodeDefaults = true }
    private val context = ApplicationProvider.getApplicationContext<Context>()
    private val repoDir = context.cacheDir

    @Test
    fun atomicWrite_fallback_succeeds_on_Robolectric() {
        val repo = LearningRepositoryImpl(context.assets, repoDir, json)

        val method = repo.javaClass.getDeclaredMethod(
            "atomicWrite", File::class.java, String::class.java
        ).apply { isAccessible = true }

        val target = File(repoDir, "queues/robo.json").apply { parentFile!!.mkdirs() }
        method.invoke(repo, target, ""ok"")

        assertTrue(target.exists())
    }
}
```

Run with:

```bash
./gradlew :data:testDebugUnitTest --tests AtomicWriteRoboTest
```

*Interpretation:*  
* **Green:** Robolectric can complete the fallback → failing tests must be elsewhere.  
* **Red:** fallback path is broken.

---

## 4 . Decision Table

| JVM Test | Robo Test | Production Risk | Action |
|----------|-----------|-----------------|--------|
| ✅ | ✅ | Low – tests themselves need fixing | Update FS‑2/3 assertions and keep. |
| ✅ | ❌ | Medium – add new fallback or mock | Patch `atomicWrite` fallback to use `Files.copy` under Robolectric. |
| ❌ | ❌ | High – logic bug affects users | Fix `atomicWrite` (tmp dir, options, paths). |

---

## 5 . Apply the Minimal Fix (if needed)

```kotlin
private fun atomicWrite(target: File, jsonText: String) {
    val tmp = File.createTempFile(target.name, ".tmp", target.parentFile)
    tmp.writeText(jsonText)

    try {
        Files.move(tmp.toPath(), target.toPath(),
                   StandardCopyOption.ATOMIC_MOVE,
                   StandardCopyOption.REPLACE_EXISTING)
    } catch (e: AtomicMoveNotSupportedException) {
        // Robolectric & exotic FS fallback
        Files.move(tmp.toPath(), target.toPath(),
                   StandardCopyOption.REPLACE_EXISTING)
    }
}
```

Remove any inner `try/catch` that previously swallowed exceptions and always returned `Result.success(Unit)`.

---

## 6 . Re‑Run Original FS‑2 / FS‑3 Tests

```bash
./gradlew :data:testDebugUnitTest --tests *FS2* --tests *FS3*
```

Both should now pass. If not, re‑examine the stack‑trace emitted in **Step 1**.

---

## 7 . Instrumentation Sanity Test (Real Device)

*Keep this in **androidTest**; run only once per CI.*

```kotlin
@LargeTest
@RunWith(AndroidJUnit4::class)
class QueuePersistenceInstrumentedTest {

    private val json = Json { encodeDefaults = true }

    @Test
    fun saveQueues_isSuccessful_on_device() = runBlocking {
        val context = InstrumentationRegistry.getInstrumentation().targetContext
        val repoDir = context.filesDir
        val repo = LearningRepositoryImpl(context.assets, repoDir, json)

        val queues = Queues(mutableListOf(dummyItem()), mutableListOf())
        val result = repo.saveQueues(queues)

        assertTrue(result.isSuccess)
        assertTrue(File(repoDir, "queues/new_queue.json").exists())
    }
}
```

Run:

```bash
./gradlew :data:connectedAndroidTest
```

**Pass = proof** that production devices can save queues.  
**Fail = critical**—fix before shipping.

---

*End of document*
