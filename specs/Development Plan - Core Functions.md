\# Test-Driven Build Plan for Lango MVP Android App \- Core Functionality Phase

\#\# Version 1.0  
\*\*Date:\*\* July 15, 2025    
\*\*Author:\*\* Grok (based on TDD Section 7, architecture spec, and user responses)    
\*\*Purpose:\*\* This document outlines a detailed Test-Driven Development (TDD) build plan focused on achieving the Core Functionality Tests from the TDD (Unit Tests 1-4 for file read, API send, API receive, audio play; plus the End-to-End Integration Test for the full sequence). This is the first viable build milestone, validating the foundational pipeline (JSON read → LLM prompt send/receive → TTS play/display) using a sample prompt like "Generate a simple greeting using the new\_target 'Hallo'". Once complete, we iterate to add full Phase 0 MVP features (e.g., session loop, mastery tracking per PROMPT).

The plan follows TDD principles: Write failing tests first, implement minimal code to pass, refactor. No UI tests; UI elements (e.g., a temporary "Test Core Flow" button) may trigger behaviors but are not assessed. The end-to-end (E2E) integration test uses predetermined artifacts (provided sample JSON, fixed prompt) and runs on an emulator (Pixel 6, Android 14\) with real OpenAI APIs; user manually verifies audio playback and text display. Unit tests use mocks (simple fixed strings) for isolation.

\*\*Key Guidelines:\*\*  
\- \*\*Tools/Frameworks:\*\* JUnit 5 for tests; Mockito for mocks; Robolectric for Android context in unit tests (e.g., asset loading); OkHttp MockWebServer as fallback for API mocking if SDK issues arise.  
\- \*\*Test Structure:\*\* Follow convention: \`ClassUnderTest\_MethodUnderTest\_ExpectedBehavior\` (e.g., \`LearningRepository\_LoadQueuesFromAssets\_ReturnsParsedEntities\`). Use \`@BeforeEach\` for setup; assertions via AssertJ for readability.  
\- \*\*Mocks:\*\* Simple fixed strings (e.g., LLM returns "Hallo\! Das ist ein Gruß"; TTS "plays" via callback).  
\- \*\*API Keys:\*\* Real keys via secrets-gradle-plugin (OPENAI\_API\_KEY) for E2E; mocks for units.  
\- \*\*Verification Process:\*\* At each stage:  
  1\. Run unit test; confirm pass (green in Android Studio/Gradle).  
  2\. If passed, integrate function into E2E test (replace placeholder).  
  3\. Run E2E; observe progressive passes.  
\- \*\*E2E Setup:\*\* Initial placeholders fail with \`fail("Not implemented")\`. Run on emulator; user verifies via logs/audio/text. Not headless.  
\- \*\*CI/CD:\*\* Use GitHub Actions for pipeline (lint, unit tests, build). Require green pipeline \+ 1 code-owner approval before merging to dev branch. Branches per stage (e.g., \`feature/core-unit1\`).  
\- \*\*Code Coverage:\*\* Target 80%+ per stage (via JaCoCo); fail pipeline if \<70%.  
\- \*\*Iteration:\*\* Commit after each stage; rapid cycles (1-2 hours per stage assuming no blockers).  
\- \*\*Dependencies:\*\* Add to build.gradle (app module unless modular):  
  \`\`\`groovy  
  // Test deps  
  testImplementation("junit:junit:5.10.3")  
  testImplementation("org.mockito:mockito-core:5.12.0")  
  testImplementation("org.robolectric:robolectric:4.13")  
  testImplementation("org.assertj:assertj-core:3.26.3")  
  testImplementation("com.squareup.okhttp3:mockwebserver:5.0.0-alpha.12") // Fallback

  // OpenAI SDK (e.g., com.knuddels:jtokkit or official; assume Retrofit for simplicity)  
  implementation("com.squareup.retrofit2:retrofit:2.11.0")  
  implementation("com.squareup.retrofit2:converter-gson:2.11.0")  
  implementation("com.google.code.gson:gson:2.11.0")

  // Coroutines for async  
  implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")  
  \`\`\`  
\- \*\*Project Structure:\*\* Modular (:app, :domain, :data, :speech). Tests in src/test.

\#\# End-to-End Integration Test Setup  
Create \`CoreSequenceIntegrationTest\` in :app/src/test (runs on emulator via @RunWith(AndroidJUnit4::class)).  
\- Use Robolectric for context; real APIs.  
\- Predetermined Artifacts: Use provided sample JSON (hardcode or copy to assets for test).  
\- Temporary Trigger: Add a "Test Core Flow" button in MainActivity to invoke the sequence (removed later).  
\- Initial Code (all placeholders fail):  
  \`\`\`kotlin  
  @RunWith(AndroidJUnit4::class)  
  class CoreSequenceIntegrationTest {

      @get:Rule val activityRule \= ActivityScenarioRule(MainActivity::class.java)

      @Test  
      fun fullCoreSequence\_completesWithoutErrors\_playsAudioAndDisplaysText() {  
          // Step 1: Load files (placeholder)  
          onView(withId(R.id.test\_core\_flow\_button)).perform(click()) // Trigger  
          fail("File load not implemented") // Placeholder fail

          // Step 2: Construct and send prompt (placeholder)  
          fail("API send not implemented")

          // Step 3: Receive and process response (placeholder)  
          fail("API receive not implemented")

          // Step 4: TTS play and display (placeholder)  
          fail("Audio play not implemented")

          // Full Verification: Check log/text/audio (user manual)  
          // e.g., assertThat(log).contains("Hallo\! Das ist ein Gruß")  
          // User verifies audio on emulator  
      }  
  }  
  \`\`\`  
\- Run initially: All steps fail. As stages complete, replace placeholders with actual assertions (e.g., Espresso checks for text display; log assertions for sequence).

\#\# Build Stages for Core Functionality

\#\#\# Stage 0: Project Bootstrap  
\- Setup modules, build.gradle with deps, Hilt, assets folder with sample JSON files (copy provided new\_queue/learned\_pool as core\_blocks.json/learned\_queue.json).  
\- Create entities (LearningItem, Queues).  
\- Create empty LearningRepository, LlmService, TtsService.  
\- Commit to \`feature/core-bootstrap\`; CI green \+ approval → merge to dev.

\#\#\# Stage 1: Unit Test 1 \- File Read (Data Module)  
\- \*\*Unit Test:\*\* In :data/src/test/LearningRepositoryTest.kt  
  \`\`\`kotlin  
  class LearningRepositoryTest {  
      private lateinit var repository: LearningRepositoryImpl  
      private lateinit var context: Context // Robolectric

      @BeforeEach fun setup() {  
          Robolectric.setupActivity(Activity::class.java)  
          context \= RuntimeEnvironment.systemContext  
          repository \= LearningRepositoryImpl(Gson(), context)  
      }

      @Test fun loadQueues\_fromAssets\_returnsParsedEntities() {  
          val result \= repository.loadQueues(null) // Null paths → assets

          assertThat(result.newQueue).hasSize(2)  
          assertThat(result.newQueue\[0\].text).isEqualTo("Hallo")  
          assertThat(result.learnedPool).hasSize(5)  
          assertThat(result.learnedPool\[0\].text).isEqualTo("Wie sagt man \_\_\_?")  
      }

      @Test fun loadQueues\_invalidJson\_throwsException() {  
          // Simulate bad file  
          assertThrows\<Exception\> { repository.loadQueues(Pair("bad\_path", "bad\_path")) }  
      }  
  }  
  \`\`\`  
\- \*\*Implementation:\*\* In LearningRepositoryImpl:  
  \- Use context.assets.open() for null paths; FileInputStream for custom paths.  
  \- Gson.fromJson(reader, Array\<LearningItem\>::class.java).toList()  
  \- Handle exceptions (e.g., JsonSyntaxException → throw).  
\- \*\*Verification:\*\* Run test; green → add to CI.  
\- \*\*Integrate to E2E:\*\* Replace Step 1 placeholder:  
  \`\`\`kotlin  
  // In E2E test  
  onView(withId(R.id.test\_core\_flow\_button)).perform(click())  
  // Assert load success via log or exposed state (e.g., Espresso check text if UI updated)  
  \`\`\`  
\- \*\*Branch:\*\* \`feature/core-unit1\`; CI green \+ approval → merge.

\#\#\# Stage 2: Unit Test 2 \- API Send (Speech/LLM Module)  
\- \*\*Unit Test:\*\* In :speech/src/test/LlmServiceTest.kt  
  \`\`\`kotlin  
  class LlmServiceTest {  
      @Mock private lateinit var retrofit: Retrofit // Or OpenAI SDK mock  
      private lateinit var service: LlmServiceImpl

      @BeforeEach fun setup() {  
          MockitoAnnotations.openMocks(this)  
          service \= LlmServiceImpl(retrofit)  
      }

      @Test fun generate\_withSamplePrompt\_sendsCorrectRequest() {  
          val prompt \= "Generate a simple greeting using the new\_target 'Hallo'"  
          val mockCall \= mock\<Call\<String\>\>()  
          whenever(retrofit.create(Any::class.java).generate(any())).thenReturn(mockCall)

          service.generate(prompt)

          verify(mockCall).enqueue(any()) // Verify send  
          // Assert prompt format (log or capture arg)  
      }  
  }  
  \`\`\`  
\- \*\*Implementation:\*\* Use Retrofit to build OpenAI endpoint; add OPENAI\_API\_KEY header.  
  \- Prompt construction: Include newTarget from dequeued item.  
\- \*\*Verification:\*\* Green test.  
\- \*\*Integrate to E2E:\*\* Replace Step 2 placeholder:  
  \`\`\`kotlin  
  // Assume trigger sends prompt  
  // Assert via log: contains(sent prompt)  
  \`\`\`  
\- \*\*Branch:\*\* \`feature/core-unit2\`; CI \+ approval.

\#\#\# Stage 3: Unit Test 3 \- API Receive (Speech/LLM Module)  
\- \*\*Unit Test:\*\*  
  \`\`\`kotlin  
  @Test fun receive\_processesMockedResponse\_returnsText() {  
      val mockedResponse \= "Hallo\! Das ist ein Gruß" // Simple fixed  
      // Mock enqueue callback success  
      val text \= service.generate("prompt") // Wait for response

      assertThat(text).isEqualTo(mockedResponse)  
      // Handle error: empty → fallback "Error"  
  }  
  \`\`\`  
\- \*\*Implementation:\*\* Enqueue callback; extract text from JSON (e.g., choices\[0\].message.content).  
\- \*\*Verification:\*\* Green.  
\- \*\*Integrate to E2E:\*\* Replace Step 3:  
  \`\`\`kotlin  
  // Assert received text in log/state  
  \`\`\`  
\- \*\*Branch:\*\* \`feature/core-unit3\`; CI \+ approval.

\#\#\# Stage 4: Unit Test 4 \- Audio Play (TTS Module)  
\- \*\*Unit Test:\*\* In :speech/src/test/TtsServiceTest.kt  
  \`\`\`kotlin  
  class TtsServiceTest {  
      @Mock private lateinit var mediaPlayer: MediaPlayer  
      private lateinit var service: TtsServiceImpl

      @BeforeEach fun setup() {  
          MockitoAnnotations.openMocks(this)  
          service \= TtsServiceImpl(retrofit, mediaPlayer)  
      }

      @Test fun speak\_withText\_callsTtsApiAndPlaysAudio() {  
          val text \= "Hallo\! Das ist ein Gruß"  
          val mockAudioStream \= ByteArrayInputStream(byteArrayOf()) // Mock API response  
          // Mock Retrofit to return audio bytes

          service.speak(text)

          verify(mediaPlayer).setDataSource(any())  
          verify(mediaPlayer).prepare()  
          verify(mediaPlayer).start()  
          // Callback on complete  
      }  
  }  
  \`\`\`  
\- \*\*Implementation:\*\* Retrofit for TTS API (audio/mp3 response); MediaPlayer.setDataSource from bytes; play; display text via intent/broadcast to UI (for E2E verify).  
\- \*\*Verification:\*\* Green.  
\- \*\*Integrate to E2E:\*\* Replace Step 4:  
  \`\`\`kotlin  
  // User verifies audio on emulator; assert text displayed via Espresso onView(withText("Hallo\! ...")).check(matches(isDisplayed()))  
  \`\`\`  
\- \*\*Branch:\*\* \`feature/core-unit4\`; CI \+ approval.

\#\# Post-Core: Adding Phase 0 MVP Features  
After E2E passes (full sequence works):  
\- New branches for features (e.g., \`feature/session-loop\`): Add TDD cycles for mastery, turns, etc.  
\- Extend E2E to full session simulation (mock user input via predetermined STT text).  
\- Remove temp button; integrate into session flow.  
\- Final CI: All tests green; coverage \>80%; approval → dev.

This plan ensures rapid iteration (stages build on each other); if blockers, refactor or ask for clarification.