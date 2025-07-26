1. Add Concrete Fake Service Classes
1.1 Create FakeLlmService.kt in speech/src/main/kotlin/.../

Implements LlmService

Reads speech/src/test/resources/sample_turn_1.json to return DialogueResponse

1.2 Create FakeTtsService.kt in speech/src/main/kotlin/.../

Implements TtsService

speak(text: String) immediately returns Result.success(Unit)

Build point: Both classes compile and implement their interfaces.

2. Add Production-Side Hilt Module (SpeechModule)
2.1 Create SpeechModule.kt under app/src/main/di/ with provideLlmService(...) and provideTtsService() that switch on BuildConfig.UAT_MODE.

2.2 Ensure any required constructor parameters (e.g. Json) are provided in the graph.

Build point: Launch the app with and without UAT_MODE and verify via logs which implementation is injected.

3. Update Test-Only Hilt Module (TestAppModule)
3.1 In app/src/di-test/TestAppModule.kt, replace the existing mockk providers with real fakes:


@Provides @Singleton fun provideLlmService(json: Json): LlmService =
  FakeLlmService(json)

@Provides @Singleton fun provideTtsService(): TtsService =
  FakeTtsService()
Build point: Add a quick smoke test to confirm instrumentation tests now inject your fakes.

4. Introduce a Test-Only SpeechRunner
(so you can smoke-test without UI)

4.1 Define SpeechRunner in production DI (speech/src/main/kotlin/.../SpeechRunner.kt):

interface SpeechRunner {
  suspend fun generateAndReturn(): DialogueResponse
  suspend fun speakText(text: String): Result<Unit>
}
4.2 In SpeechModule, bind it:

kotlin
Copy
Edit
@Provides @Singleton fun provideSpeechRunner(
  llm: LlmService,
  tts: TtsService
): SpeechRunner = object : SpeechRunner {
  override suspend fun generateAndReturn() = llm.generateDialogue()
  override suspend fun speakText(text: String) = tts.speak(text)
}
Build point: Verify you can inject SpeechRunner and call its methods in isolation.

5. Surface Test Fixture in “Test Data” Section
5.1 Add to MS1_F3_TDD.md under Test Data:

## Test Data
- **sample_turn_1.json**  
  Location: `speech/src/test/resources/sample_turn_1.json`  
  Contains a single turn of canned LLM output for `FakeLlmService`.
Build point: Confirm developers know exactly where to place/update the fixture.

6. Implement & Validate Unit Tests for Fakes
6.1 (C-1) In speech/src/test/FakeLlmServiceTest.kt, assert generateDialogue() returns the exact JSON from speech/src/test/resources/sample_turn_1.json.

6.2 (C-2) In speech/src/test/FakeTtsServiceTest.kt, assert speak("any text") returns Result.success immediately.

Build point: All unit tests in :speech pass green.

7. Instrumentation Tests for DI Toggle + Runner Smoke Test
7.1 (DI-1) In app/src/androidTest/HiltSwitchInstrumentationTest.kt:

Launch with UAT_MODE=true, retrieve LlmService/TtsService, assert concrete types.

Invoke generateDialogue() & speak("test") on the injected services.

Repeat with UAT_MODE=false, expecting the real implementations.

7.2 (Smoke via Runner) In the same test (or a new one):


@Test fun smoke_speechRunner_flow_when_UAT() = runBlocking {
  val runner = EntryPointAccessors.fromApplication(
    ApplicationProvider.getApplicationContext(),
    TestSpeechRunnerEntryPoint::class.java
  ).speechRunner()

  // Fake flow:
  val dialogue = runner.generateAndReturn()
  assertThat(dialogue).isEqualTo(
    File("speech/src/test/resources/sample_turn_1.json")
      .readText().toDialogueResponse()
  )
  assertThat(runner.speakText("hello").isSuccess).isTrue()
}
Build point: Instrumentation suite passes, covering both DI binding and end-to-end smoke flow without any UI.