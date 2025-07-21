
# FR1 – **LearningRepositoryImpl** test failures (`FS‑2 Persist & reload`, `FS‑3 Concurrent read‑write`)

---

## 1&nbsp;· Executive summary  
Both failing tests die **before their functional assertions run** — the save operation itself is returning `Result.failure`.  
The trouble is **inside `LearningRepositoryImpl.saveQueues`**, _not_ in the test logic that reloads the repository.  
Fix the repository’s atomic‑write helper and the original tests pass as written.

> Your plan to “keep the repository instance alive / reload after save” treats a symptom, not the disease.

---

## 2&nbsp;· What the crash tells us  

* `FS‑2` throws at `assertTrue(saveResult.isSuccess)` — the very first assertion after the call to `saveQueues` fileciteturn1file4.  
* The only code path that makes `saveQueues` return a failure is the `catch (IOException)` around the `atomicWrite` helper fileciteturn2file4.  

Therefore the test never reaches the *reload* step; persistence is failing at the filesystem layer.

---

## 3&nbsp;· Root cause  

`atomicWrite()` always attempts  
```kotlin
Files.move(tmp.toPath(), target.toPath(),
            StandardCopyOption.ATOMIC_MOVE,
            StandardCopyOption.REPLACE_EXISTING)
```  
but **`StandardCopyOption.ATOMIC_MOVE` is not supported by the Jimfs/Windows‑temp filesystem that Robolectric uses**.  
`java.nio.file.AtomicMoveNotSupportedException` is a subclass of `IOException`, so it is swallowed by the `catch (IOException)` block and converted into `Result.failure`, reproducing the stack‑trace observed.

---

## 4&nbsp;· Resolution design  

| Change | File | Details |
|--------|------|---------|
| **4‑A** | `LearningRepositoryImpl.kt` | Replace the one‑shot `Files.move(…, ATOMIC_MOVE)` with a *graceful fallback*: attempt atomic move, catch `AtomicMoveNotSupportedException`, and redo the move without the flag. |
| **4‑B** | `atomicWrite` signature | Add a `suspend` variant so we can stay on `Dispatchers.IO` and call it directly from the main coroutine instead of wrapping try/catch inside `saveQueues`. |
| **4‑C** | `saveQueues()` | Remove the inner `try/catch`; let all exceptions surface inside `Result.runCatching { … }` so tests fail loudly on unexpected issues. |
| **4‑D** | Unit tests | Keep original `FS‑2` & `FS‑3`; add **one tiny new test** `FS‑2b Unsupported‑atomic‑move fallback` that force‑throws `AtomicMoveNotSupportedException` with a temp Jimfs filesystem. |

### 4‑A. Suggested replacement  
```kotlin
private fun atomicWrite(target: File, text: String) {
    val tmp = File.createTempFile(target.name, ".tmp", target.parentFile)
    tmp.writeText(text)
    try {
        Files.move(
            tmp.toPath(), target.toPath(),
            StandardCopyOption.ATOMIC_MOVE,
            StandardCopyOption.REPLACE_EXISTING
        )
    } catch (e: AtomicMoveNotSupportedException) {
        // Falls back to a regular, still thread‑safe replace.
        Files.move(
            tmp.toPath(), target.toPath(),
            StandardCopyOption.REPLACE_EXISTING
        )
    }
}
```

---

## 5&nbsp;· Additional hardening (optional but low‑cost)

1. **fsync** after writing `tmp.writeText()` to guarantee durability before the move.  
2. Use **`FileChannel` + `.force(true)`** for larger payloads.  
3. Change the JSON codec to **`prettyPrint = false`** to minimise write size.  

---

## 6&nbsp;· Steps to implement  

1. **Patch the helper** (`4‑A`).  
2. **Delete the `try/catch` in `saveQueues`** and wrap the whole block in `return runCatching { … }`.  
3. **Rerun unit tests** – `FS‑2` and `FS‑3` now pass.  
4. Add new fallback test (`4‑D`) to pin the behaviour.  
5. Run a full instrumented test pass on macOS/Linux CI to confirm ATOMIC_MOVE still used where available.

---

## 7&nbsp;· Documentation / code‑review checklist  

* Update **`FR1_Build_Plan.md`** to list the fallback as **KEY_CHANGE_3**.  
* Mention Windows/Jimfs limitation in **Architecture.md → Data layer → Persistence**.  
* Link the new test case in **TDD.md**.  

---

## 8&nbsp;· Outcome  

With the atomic‑write fallback in place:

* `saveQueues()` succeeds in all filesystems Robolectric uses.  
* `FS‑2` properly persists and reloads modified state.  
* `FS‑3` can interleave read/write without data loss, validated by the existing mutex.  

No test modifications are required beyond the new guard case, so your CI remains aligned with the functional‑requirements document.

