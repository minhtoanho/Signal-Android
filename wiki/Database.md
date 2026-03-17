# Database

Signal uses SQLite with SQLCipher encryption for data persistence.

## Overview

The database layer is located in `app/src/main/java/org/thoughtcrime/securesms/database/`.

### Key Characteristics

- **Encrypted**: All data at rest is encrypted using SQLCipher
- **Migrations**: Schema changes managed through migrations
- **Observers**: Real-time change notifications
- **Thread-safe**: All database operations are thread-safe

## Core Database Tables

### Message Tables

| Table | Purpose |
|-------|---------|
| `MessageTable` | SMS and MMS messages |
| `MmsTable` | MMS-specific message data |
| `SmsTable` | SMS-specific message data |
| `AttachmentTable` | Message attachments |

### Thread Tables

| Table | Purpose |
|-------|---------|
| `ThreadTable` | Conversation threads |
| `GroupTable` | Group information |
| `DistributionListTables` | Story distribution lists |

### Identity Tables

| Table | Purpose |
|-------|---------|
| `IdentityTable` | Identity keys for users |
| `RecipientTable` | Recipient information |
| `SessionTable` | Signal protocol sessions |

### Media Tables

| Table | Purpose |
|-------|---------|
| `AttachmentTable` | Attachment metadata |
| `EmojiSearchTable` | Emoji search index |
| `StickerTable` | Sticker packs |

### System Tables

| Table | Purpose |
|-------|---------|
| `KeyValueDatabase` | Key-value storage |
| `JobDatabase` | Job queue persistence |
| `ChatFolderTables` | Chat folder configuration |

## Database Access

### DatabaseFactory

The `SignalDatabase` (formerly DatabaseFactory) provides access to all database tables:

```kotlin
// Get message database
SignalDatabase.messages

// Get thread database  
SignalDatabase.threads

// Get attachment database
SignalDatabase.attachments
```

### DatabaseTable

All database tables extend `DatabaseTable`:

```kotlin
abstract class DatabaseTable(context: Context) {
    protected val context: Context = context.applicationContext
    protected abstract val tableName: String
    
    // Common database operations
}
```

## Database Encryption

### SQLCipher Setup

```kotlin
// Database is encrypted using SQLCipher
// Key is derived from user's passphrase
class DatabaseSecret(val encoded: String) {
    // Encrypted database key
}
```

### Key Derivation

1. User enters passphrase
2. Passphrase is stretched using KDF
3. Derived key encrypts the actual database key
4. Database key is stored encrypted

## Database Observers

### DatabaseObserver

Real-time change notifications:

```kotlin
val observer = DatabaseObserver(context)

// Observe message changes
observer.registerMessageObserver { threadId ->
    // Handle message change
}

// Observe thread changes
observer.registerThreadObserver {
    // Handle thread change
}
```

### Observable Updates

Tables notify observers after changes:

```kotlin
// After inserting message
notifyConversationListeners(threadId)
notifyConversationListListeners()
```

## Migrations

### Migration System

Migrations are versioned and applied sequentially:

```kotlin
object Migrations {
    fun migrate(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
        // Apply migrations from oldVersion to newVersion
    }
}
```

### Migration Best Practices

1. Always migrate forward
2. Handle exceptions gracefully
3. Log migration steps
4. Test with real data

## Content Providers

Signal uses content providers for:
- External app integration (limited)
- Search suggestions
- Data sharing

## Query Patterns

### Cursor-Based Queries

```kotlin
fun getMessages(threadId: Long): Cursor {
    readableDatabase.query(
        TABLE_NAME,
        null,
        "$THREAD_ID = ?",
        arrayOf(threadId.toString()),
        null,
        null,
        "$DATE_SENT DESC"
    ).use { cursor ->
        // Process cursor
    }
}
```

### Kotlin Extensions

```kotlin
// Extension for cursor to list
fun <T> Cursor.mapToList(mapper: (Cursor) -> T): List<T>

// Read single value
fun <T> Cursor.readSingle(mapper: (Cursor) -> T): T?
```

## Performance Considerations

### Indexes

Key columns are indexed:
- Thread IDs
- Message timestamps
- Recipient IDs

### Batch Operations

```kotlin
// Use transactions for batch operations
writableDatabase.beginTransaction()
try {
    // Multiple operations
    writableDatabase.setTransactionSuccessful()
} finally {
    writableDatabase.endTransaction()
}
```

### WAL Mode

Write-Ahead Logging is enabled for better concurrent access.

## Backup

### Backup Format

- Encrypted backup file
- Contains all messages and attachments
- Can be restored on new device

### Backup Tables

```kotlin
object BackupTable {
    // Tracks backup state
    // Media snapshots
    // Backup metadata
}
```

## Testing

### In-Memory Databases

Tests use in-memory databases:

```kotlin
@Before
fun setUp() {
    // Create in-memory database for testing
    testDatabase = SignalDatabase.getInMemoryDatabase()
}
```

### Test Fixtures

Common test data patterns:
- Test messages
- Test recipients
- Test threads