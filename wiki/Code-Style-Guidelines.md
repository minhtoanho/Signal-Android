# Code Style Guidelines

Signal follows specific code conventions to maintain consistency across the codebase.

## Formatting

### Automatic Formatting

Use ktlint for automatic formatting:

```bash
# Check formatting
./gradlew ktlintCheck

# Auto-format code
./gradlew format
```

### Git Hook

Set up automatic formatting on commit using lefthook:

```bash
# See lefthook.yml for configuration
```

## Language Conventions

### Kotlin (Preferred)

Kotlin is the preferred language for new code:

```kotlin
// Use data classes for models
data class User(
    val id: Long,
    val name: String,
    val phoneNumber: String?
)

// Use sealed classes for state
sealed class Result {
    data class Success(val data: Data) : Result()
    data class Error(val message: String) : Result()
}

// Use extension functions
fun String.toPhoneNumber(): PhoneNumber {
    // Extension function
}
```

### Java

Java is used for legacy code and interoperability:

```java
// Use final for immutability
public final class User {
    private final long id;
    private final String name;
    
    public User(long id, String name) {
        this.id = id;
        this.name = name;
    }
}
```

## Naming Conventions

### Classes

- PascalCase for class names
- Descriptive names, avoid abbreviations

```kotlin
class ConversationActivity : AppCompatActivity()
class MessageAdapter : RecyclerView.Adapter<ViewHolder>()
class SendJob : BaseJob()
```

### Functions

- camelCase for function names
- Verb-noun format for actions

```kotlin
fun sendMessage(message: Message)
fun getUserById(id: Long): User?
fun calculateUnreadCount(): Int
```

### Variables

- camelCase for variables
- Descriptive names

```kotlin
// Good
val unreadMessageCount = 0
val currentUser = getCurrentUser()

// Avoid
val count = 0
val u = getCurrentUser()
```

### Constants

- UPPER_SNAKE_CASE for constants
- Use companion object or @JvmField

```kotlin
companion object {
    const val MAX_MESSAGE_LENGTH = 65536
    const val DEFAULT_PAGE_SIZE = 50
}
```

## Code Organization

### File Structure

```kotlin
// 1. Copyright notice (if required)
// 2. Package declaration
package org.thoughtcrime.securesms.example

// 3. Import statements (alphabetized)
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import org.thoughtcrime.securesms.R

// 4. Class declaration
class ExampleActivity : AppCompatActivity() {
    
    // 5. Companion object
    companion object {
        private const val TAG = "ExampleActivity"
    }
    
    // 6. Properties
    private lateinit var binding: ActivityExampleBinding
    
    // 7. Lifecycle methods
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
    }
    
    // 8. Public methods
    fun publicMethod() {}
    
    // 9. Private methods
    private fun privateMethod() {}
    
    // 10. Inner classes
    inner class InnerClass
}
```

### Import Order

1. Android imports
2. AndroidX imports
3. Third-party imports
4. Signal imports
5. Java imports

## Null Safety

### Null Checks

```kotlin
// Use safe calls
user?.name?.let { name ->
    // Use name
}

// Use elvis operator
val name = user?.name ?: "Unknown"

// Use require/check for preconditions
requireNotNull(value) { "Value cannot be null" }
```

### Nullable Types

```kotlin
// Prefer nullable types over null objects
fun findUser(id: Long): User? {
    // Return null if not found
}

// Use @Nullable/@NonNull for Java interop
@Nullable
fun findUser(id: Long): User?
```

## Threading

### Coroutines

```kotlin
// Use lifecycleScope in activities/fragments
lifecycleScope.launch {
    val result = withContext(Dispatchers.IO) {
        // Background work
    }
    // Update UI
}

// Use viewModelScope in ViewModels
viewModelScope.launch {
    // Coroutine work
}
```

### Background Threads

```kotlin
// Use SignalExecutors for thread pools
SignalExecutors.BOUNDED.execute {
    // Background work
}

// Use ThreadUtil for common operations
ThreadUtil.runOnMain {
    // UI update
}
```

## Logging

### Log Levels

```kotlin
import org.signal.core.util.logging.Log

// Use appropriate log levels
Log.d(TAG, "Debug message")
Log.i(TAG, "Info message")
Log.w(TAG, "Warning message")
Log.e(TAG, "Error message", exception)
```

### Sensitive Data

Never log sensitive data:
- Passwords
- Keys
- Phone numbers
- Message contents
- Personal information

```kotlin
// BAD - Don't do this
Log.d(TAG, "User password: $password")

// GOOD - Log safely
Log.d(TAG, "User authenticated")
```

## Resources

### Strings

```xml
<!-- strings.xml -->
<string name="message_sent">Message sent</string>
<string name="unread_count">Unread: %d</string>
```

```kotlin
// Use resources
getString(R.string.message_sent)
getString(R.string.unread_count, count)
```

### Dimensions

```xml
<!-- dimens.xml -->
<dimen name="margin_small">8dp</dimen>
<dimen name="margin_medium">16dp</dimen>
```

### Colors

```xml
<!-- colors.xml -->
<color name="signal_primary">#3A76F0</color>
```

## Comments

### When to Comment

- Complex algorithms
- Non-obvious decisions
- Workarounds
- TODOs (with ticket number)

```kotlin
// Calculate HMAC using SHA256 for message integrity
// See: https://github.com/signalapp/Signal-Android/issues/1234
private fun calculateHmac(data: ByteArray): ByteArray {
    // Implementation
}
```

### When NOT to Comment

- Obvious code
- Redundant explanations
- Commented-out code (delete it)

## Error Handling

### Exceptions

```kotlin
// Throw appropriate exceptions
throw IllegalArgumentException("Invalid message type")

// Catch and handle appropriately
try {
    sendOperation()
} catch (e: IOException) {
    Log.w(TAG, "Send failed", e)
    Result.Error(e)
}
```

### Result Types

```kotlin
// Prefer Result type over exceptions
sealed class OperationResult {
    data class Success(val data: Data) : OperationResult()
    data class Error(val exception: Exception) : OperationResult()
}
```

## Testing

### Unit Tests

```kotlin
class MessageTest {
    @Test
    fun `test message creation`() {
        val message = Message(content = "Hello")
        assertEquals("Hello", message.content)
    }
}
```

### Test Naming

```kotlin
// Use descriptive test names with backticks
@Test
fun `given user is null when loading then return empty`()

@Test
fun `given valid input when sending message then success`()
```

## Dependency Injection

Signal uses a service locator pattern:

```kotlin
// Access dependencies through ApplicationContext
val database = ApplicationContext.getInstance(context).messageDatabase
```

## Checklist Before Commit

- [ ] Code formatted with `./gradlew format`
- [ ] No STOPSHIP comments
- [ ] Tests pass
- [ ] Lint passes
- [ ] No sensitive data in logs
- [ ] Strings externalized
- [ ] Proper null handling