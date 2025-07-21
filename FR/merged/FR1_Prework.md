# FR1 Pre‑work — Blocking & Drift Fix Tasks

> Scope: Items from analysis Sections 1 & 2 plus the Robolectric → TemporaryFolder change (Section 4).  
> Complete these in order **before** any remaining FR‑1 feature work.

---

## 0 – Ground Rules
1. Work **top‑to‑bottom**; each task unblocks the next.  
2. After every **Build Check** run the indicated Gradle command on a **clean** workspace  
   (`./gradlew clean` first).  
3. Push an **atomic PR** per task (or small group) so reviewers can verify each fix in isolation.  

---

## ⛔ Section 1 — Blocking Build / Test Failures

|  № | Task | Files / Classes | Steps | Build Check |
|---|------|-----------------|-------|-------------|
| 1 |Replace *AssetManager* injection with `Context.assets`|`app/src/main/java/.../di/RepositoryModule.kt`<br>`LearningRepositoryImpl`|1. Change provider signature:<br>```kotlin
@Provides fun provideLearningRepository(@ApplicationContext ctx: Context, …): LearningRepository =
    LearningRepositoryImpl(ctx.assets, ctx.filesDir, json)
```<br>2. Remove stale `AssetManager` imports.|`./gradlew :app:assembleDebug` must compile (no KAPT binding errors).|
| 2 |Stabilise repository unit tests|`data/src/test/.../LearningRepositoryImplTest.kt` (+ siblings)|A. Add **optional ctor flag** to `LearningRepositoryImpl`:<br>`bootstrap: Boolean = true`<br>B. In tests instantiate with `bootstrap = false` **or** delete both queue files in `@Before`.<br>C. Adjust expectations (malformed JSON should surface).|`./gradlew :data:test --tests "*LearningRepositoryImplTest"` all green.|
| 3 |Align Kotlin version across build|`gradle/libs.versions.toml`<br>`build.gradle.kts` (root)|Set **one** Kotlin version (`2.2.0`) everywhere.<br>➜ in *versions.toml*: `kotlin = "2.2.0"`<br>➜ in root plugin block: `kotlin("android") version libs.versions.kotlin`.|`./gradlew clean build` — no `NoSuchMethodError` at runtime tests.|
| 4 |Add DI bindings for Speech module|`:speech/src/main/.../LlmServiceImpl.kt`<br>`TtsServiceImpl.kt`<br>(new) `SpeechModule.kt`|A. Annotate impl ctors: `@Inject constructor(…)` & `@Singleton`.<br>B. Alternative: create `@Module` with `@Binds` for each interface→impl.|`./gradlew :app:assembleDebug` — Hilt generates bindings without errors.|

---

## ⚠️ Section 2 — Pre‑work Drift (Fix before feature work)

|  № | Task | Files / Classes | Steps | Build Check |
|---|------|-----------------|-------|-------------|
| 5 |Make `ensureBootstrap()` **non‑destructive**|`LearningRepositoryImpl`|Change copy condition to **both** files missing:<br>`if (!newQueueFile.exists() && !learnedQueueFile.exists()) { copyFromAssets() }`|`./gradlew :data:test` — new *IntegrationBootstrap* test passes; manual progress files survive app restart.|
| 6 |Tidy `atomicWrite()` temp files|`FileUtils.kt` (or where `atomicWrite` lives)|After successful `Files.move`, add `tmp.delete()` (inside `try`).|Run new unit test *atomicWriteDeletesTemp* in `:data:test`.|
| 7 |Propagate JSON load failures to UI|`StartSessionUseCase.kt`|Return the `Result` from repository **unchanged**; remove blanket `Result.success(Unit)`.<br>Update ViewModel to surface error state.|`./gradlew :domain:test` — *StartSessionUseCaseFailure* test green.|
| 8 |Remove duplicate Hilt bindings|`LearningRepositoryImpl` annotation|Delete class‑level `@Singleton` annotation (module already provides).|`./gradlew :app:assembleDebug` — no `DuplicateBindingsException`.|

---

## 🧪 Section 4 — Test Infrastructure Cleanup

|  № | Task | Files / Classes | Steps | Build Check |
|---|------|-----------------|-------|-------------|
| 9 |Replace **Robolectric** with JUnit `TemporaryFolder`|`data/build.gradle.kts` (test deps)<br>`data/src/test/.../*.kt`|A. Remove `robolectric` dependency.<br>B. Add JUnit 5 (*junit‑jupiter*) & `org.junit.jupiter.api.io.TempDir`.<br>C. Replace `RobolectricTestRunner` with `JUnitPlatform`. <br>D. Use `@TempDir Path tempDir` in tests instead of `RuntimeEnvironment.getApplication()`. |`./gradlew :data:test` should pass in <3 s (tests run on plain JVM).|

---

### ✅ Completion Criteria

* All nine **Build Checks** succeed on CI.  
* No regressions in existing feature tests.  
* Team can continue FR‑1 feature work on a stable base.

---

**Last updated:** 21 Jul 2025 (Europe/Vienna)
