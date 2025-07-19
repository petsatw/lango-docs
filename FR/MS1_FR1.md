# Feature Request: Bootstrap & Persist Queues

### Title  
Bootstrap & Persist Queues

---

### Problem  
On a fresh install the app crashes because `LearningRepositoryImpl.loadQueues()` finds no queue files in `filesDir`.  
The app must instead copy bundled JSON into private storage, parse it into memory, and later persist changes safely.

---

### Scope  
* **First-run bootstrap** – copy `assets/queues/new_queue.json` and `assets/queues/learned_queue.json` to **`context.filesDir`** the very first time the app runs.  
* **Parsing** – read both files into in-memory `Queue<LearningItem>` objects using Gson / Kotlinx with camel-case fields.  
* **Save & mutex** – `saveQueues()` serialises queues to a temp file, then atomically renames; guarded by a `Mutex` shared with `loadQueues()`.  
* **Robustness** – if a file is missing or JSON is malformed, return `Result.failure` and fall back to asset defaults.

### Non-Goals  
* UI feedback or progress bars.  
* Database migrations, cloud backup, or encryption.

---

### Acceptance Criteria  

| Given | When | Then |
|-------|------|------|
| Fresh install (no queue files) | `LearningRepository.loadQueues()` is called | Files `new_queue.json` & `learned_queue.json` now exist in `filesDir`, and the returned `Result` is **success** with queues equal to bundled JSON. |
| Queues are modified in memory | `saveQueues()` is invoked | JSON on disk reflects the modified counts; call returns success. |
| Another thread simultaneously calls `loadQueues()` | Both operations complete | No data races or exceptions occur. |
| A corrupted JSON file exists | `loadQueues()` is called | `Result.failure(JsonSyntaxException)` is returned **and** queues fall back to asset defaults. |
| Path points to a non‑existent directory | `loadQueues()` is called | Operation succeeds by copying assets into that directory. |

---

### Affected Modules  
* **data** – `LearningRepositoryImpl`  
* **di** – provider now passes `Context` into the repository

---

### Data Contract  

```jsonc
// new_queue.json  (identical shape for learned_queue.json)
[
  {
    "id": "CP017",
    "phrase": "Wie viel kostet das?",
    "presentationCount": 0,
    "usageCount": 0
  }
  // …
]
```
* File names use **snake_case**; JSON property names are **camelCase** to match the Kotlin data class.

---

### UX Assets  
None.

---

### Test Guidance  

| Test ID | Level | Scenario & Steps | Expected Assertions |
|---------|-------|------------------|---------------------|
| **FS‑1 – Cold‑start load** | Instrumented (Robolectric ok) | 1. Delete all files in `context.filesDir`.<br>2. `repo.loadQueues()` | *a)* Call returns `Result.success`.<br>*b)* `new_queue.json` and `learned_queue.json` now exist.<br>*c)* `newQueue.size` equals bundled JSON count.<br>*d)* First item ID matches first JSON element. |
| **FS‑2 – Persist & reload** | Instrumented | 1. Load queues.<br>2. Increment `presentationCount` on first item.<br>3. `repo.saveQueues()`.<br>4. New repo → `loadQueues()` | Incremented count is present; call returns success. |
| **FS‑3 – Concurrent read/write** | JVM (temp dir) | 1. Repo points at `tempDir`.<br>2. Coroutine A: `loadQueues()`.<br>3. Coroutine B: `saveQueues()` | Both complete without exceptions; JSON intact. |
| **FS‑4 – Malformed JSON** | JVM | 1. Write `{ bad json,}` to `tempDir/new_queue.json`.<br>2. `loadQueues(path)` | Returns `Result.failure(JsonSyntaxException)` **and** fallback queues match asset defaults. |
| **FS‑5 – Missing file** | JVM | 1. Point repo at `emptyDir` (no JSON files).<br>2. `loadQueues()` | Returns `Result.success`; queues equal bundled defaults; files created in `emptyDir`. |

