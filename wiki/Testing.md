# Testing

Signal Android has comprehensive testing across multiple levels.

## Test Types

### Unit Tests

Located in `src/test/` directories. Test individual classes and functions.

```kotlin
class ExampleTest {
    @Test
    fun `test something`() {
        assertEquals(4, 2 + 2)
    }
}
```

### Instrumentation Tests

Located in `src/androidTest/` directories. Test UI and Android components.

```kotlin
@RunWith(AndroidJUnit4::class)
class ExampleInstrumentedTest {
    @Test
    fun useAppContext() {
        val appContext = InstrumentationRegistry.getInstrumentation().targetContext
        assertEquals("org.thoughtcrime.securesms", appContext.packageName)
    }
}
```

### Benchmark Tests

Located in `benchmark/` and `microbenchmark/` modules. Test performance.

## Running Tests

### All Unit Tests

```bash
./gradlew testPlayProdDebugUnitTest
```

### Specific Module Tests

```bash
./gradlew :app:testPlayProdDebugUnitTest
./gradlew :lib:signal-service:testDebugUnitTest
```

### Specific Test Class

```bash
./gradlew test --tests "org.thoughtcrime.securesms.database.MessageTableTest"
```

### Instrumentation Tests

```bash
# Requires connected device or emulator
./gradlew connectedPlayProdDebugAndroidTest
```

### Full QA Suite

```bash
./gradlew qa
```

This runs:
- Clean build
- ktlint checks
- Unit tests
- Lint checks
- Build logic tests

## Test Structure

### App Module Tests

```
app/src/
├── test/
│   └── java/org/thoughtcrime/securesms/
│       ├── database/          # Database tests
│       ├── crypto/            # Crypto tests
│       ├── util/              # Utility tests
│       └── jobs/              # Job tests
└── androidTest/
    └── java/org/thoughtcrime/securesms/
        ├── database/          # Database instrumentation tests
        ├── ui/                # UI tests
        └── espresso/          # Espresso tests
```

### Library Module Tests

Each library module has its own test directory:

```
lib/libsignal-service/src/
├── test/
│   └── java/                  # Unit tests
└── androidTest/
    └── java/                  # Instrumentation tests
```

## Testing Patterns

### ViewModel Testing

```kotlin
class ConversationViewModelTest {
    @get:Rule
    val instantTaskExecutorRule = InstantTaskExecutorRule()
    
    private lateinit var viewModel: ConversationViewModel
    
    @Before
    fun setUp() {
        viewModel = ConversationViewModel(/* test dependencies */)
    }
    
    @Test
    fun `given message sent when viewModel processes then updates state`() = runTest {
        // Given
        val message = Message(content = "Test")
        
        // When
        viewModel.sendMessage(message)
        
        // Then
        assertEquals(MessageState.SENT, viewModel.state.value)
    }
}
```

### Database Testing

```kotlin
@RunWith(AndroidJUnit4::class)
class MessageTableTest {
    private lateinit var database: SignalDatabase
    
    @Before
    fun setUp() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = SignalDatabase.getInMemoryDatabase(context)
    }
    
    @Test
    fun `insert message returns correct id`() = runTest {
        val message = TestFixtures.message()
        
        val id = database.messages.insert(message)
        
        assertTrue(id > 0)
    }
}
```

### Job Testing

```kotlin
class SendJobTest {
    @Test
    fun `given network available when job runs then sends message`() = runTest {
        // Given
        val job = SendJob(message)
        
        // When
        job.run()
        
        // Then
        verify(messageSender).send(message)
    }
    
    @Test
    fun `given no network when job runs then retries`() = runTest {
        // Given
        val job = SendJob(message)
        whenever(network.isAvailable).thenReturn(false)
        
        // When
        job.run()
        
        // Then
        assertEquals(JobResult.RETRY, job.result)
    }
}
```

## Test Fixtures

### Creating Test Data

```kotlin
object TestFixtures {
    fun message(
        id: Long = 1,
        content: String = "Test message",
        timestamp: Long = System.currentTimeMillis()
    ) = Message(id, content, timestamp)
    
    fun recipient(
        id: Long = 1,
        phoneNumber: String = "+15551234567"
    ) = Recipient(id, phoneNumber)
}
```

### In-Memory Database

```kotlin
@Before
fun setUp() {
    val context = ApplicationProvider.getApplicationContext<Context>()
    database = SignalDatabase.getInMemoryDatabase(context)
}
```

## Mocking

### Using Mockito

```kotlin
@Mock
private lateinit var mockRepository: MessageRepository

@InjectMocks
private lateinit var viewModel: MessageViewModel

@Before
fun setUp() {
    MockitoAnnotations.openMocks(this)
}

@Test
fun test() {
    whenever(mockRepository.getMessages()).thenReturn(listOf(message))
    
    viewModel.loadMessages()
    
    verify(mockRepository).getMessages()
}
```

### Using MockK (Kotlin)

```kotlin
private val mockRepository: MessageRepository = mockk()

@Before
fun setUp() {
    every { mockRepository.getMessages() } returns listOf(message)
}

@Test
fun test() {
    val result = mockRepository.getMessages()
    
    verify { mockRepository.getMessages() }
}
```

## UI Testing with Espresso

```kotlin
@RunWith(AndroidJUnit4::class)
class ConversationActivityTest {
    @get:Rule
    val activityRule = ActivityScenarioRule(ConversationActivity::class.java)
    
    @Test
    fun typingMessage_updatesEditText() {
        onView(withId(R.id.message_input))
            .perform(typeText("Hello"))
        
        onView(withId(R.id.message_input))
            .check(matches(withText("Hello")))
    }
    
    @Test
    fun sendMessage_clearsInput() {
        onView(withId(R.id.message_input))
            .perform(typeText("Hello"))
        
        onView(withId(R.id.send_button))
            .perform(click())
        
        onView(withId(R.id.message_input))
            .check(matches(withText("")))
    }
}
```

## Coroutines Testing

```kotlin
class CoroutineTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    @Test
    fun `test coroutine function`() = runTest {
        val result = suspendFunction()
        
        assertEquals(expected, result)
    }
}

class MainDispatcherRule : TestWatcher() {
    override fun starting(description: Description?) {
        Dispatchers.setMain(StandardTestDispatcher())
    }
    
    override fun finished(description: Description?) {
        Dispatchers.resetMain()
    }
}
```

## Benchmarking

### Macrobenchmark

```kotlin
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule
    val rule = MacrobenchmarkRule()
    
    @Test
    fun startupNoCompilation() = rule.measureRepeated(
        packageName = "org.thoughtcrime.securesms",
        metrics = listOf(StartupTimingMetric()),
        iterations = 10,
        compilationMode = CompilationMode.None()
    ) {
        pressHome()
        startActivityAndWait()
    }
}
```

### Microbenchmark

```kotlin
@RunWith(AndroidJUnit4::class)
class CryptoBenchmark {
    @get:Rule
    val benchmarkRule = BenchmarkRule()
    
    @Test
    fun encryptMessage() {
        val message = "Test message".toByteArray()
        benchmarkRule.measureRepeated {
            Crypto.encrypt(message)
        }
    }
}
```

## Continuous Integration

Signal uses CI for:
- Unit tests on every PR
- Instrumentation tests on merges
- Performance benchmarks
- Lint checks
- Security scans

## Test Coverage

Check coverage with:

```bash
./gradlew testPlayProdDebugUnitTestCoverage
```

Coverage reports are generated in `app/build/reports/coverage/`.

## Best Practices

1. **Write tests first** when possible (TDD)
2. **Test behavior, not implementation**
3. **Use descriptive test names** with backticks
4. **One assertion per test** when reasonable
5. **Use test fixtures** for common data
6. **Mock external dependencies**
7. **Test edge cases and errors**
8. **Keep tests fast and isolated**