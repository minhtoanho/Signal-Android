# Security & Cryptography

Signal's security is built on the Signal Protocol, providing end-to-end encryption for all communications.

## Signal Protocol

### Core Components

1. **X3DH Key Agreement** - Initial session establishment
2. **Double Ratchet** - Ongoing message encryption
3. **Curve25519** - Elliptic curve cryptography
4. **AES-256** - Symmetric encryption
5. **HMAC-SHA256** - Message authentication

### Key Types

| Key Type | Purpose |
|----------|---------|
| Identity Key | Long-term identity |
| Signed Pre-Key | Medium-term key, signed by identity |
| One-Time Pre-Key | Single-use keys for new sessions |
| Sender Key | Group messaging encryption |

## Key Management

### Key Storage Location

```
org.thoughtcrime.securesms.crypto/
├── IdentityKeyParcelable.java    # Parcelable wrapper
├── KeyStoreHelper.java          # Android Keystore wrapper
├── MasterSecret.java            # Master secret for encryption
├── MasterSecretUtil.java        # Secret utilities
├── PreKeyUtil.java              # Pre-key management
├── ProfileKeyUtil.java          # Profile key utilities
└── SealedSenderAccessUtil.java  # Sealed sender access
```

### Android Keystore

Signal uses Android Keystore for:
- Key generation
- Key storage
- Hardware-backed encryption

```kotlin
object KeyStoreHelper {
    // Store keys in Android Keystore
    // Hardware-backed when available
}
```

### Master Secret

The master secret protects sensitive data:

```kotlin
class MasterSecret(val encoded: ByteArray) {
    // Used to encrypt database and attachments
}
```

## Database Encryption

### SQLCipher

All user data is encrypted at rest using SQLCipher:

```kotlin
class DatabaseSecret(val encoded: String) {
    // SQLCipher key
}
```

### Key Derivation

1. User passphrase → KDF → derived key
2. Derived key encrypts database key
3. Database key encrypts all data

## Attachment Encryption

### Encryption Process

1. Generate random attachment secret
2. Encrypt attachment with secret
3. Store encrypted attachment
4. Store secret in encrypted database

### Attachment Secret

```kotlin
class AttachmentSecret(val key: ByteArray, val iv: ByteArray) {
    // Used to encrypt/decrypt attachments
}
```

### Streaming Encryption

```kotlin
// Modern encrypting output stream
ModernEncryptingPartOutputStream(secret, outputStream)

// Modern decrypting input stream
ModernDecryptingPartInputStream(secret, inputStream)
```

## End-to-End Encryption

### Message Encryption

```kotlin
// Encrypt message for recipient
fun encrypt(message: Message, recipient: Recipient): EncryptedMessage {
    val session = getSession(recipient)
    val cipher = SessionCipher(session)
    return cipher.encrypt(message.body)
}
```

### Message Decryption

```kotlin
// Decrypt message from sender
fun decrypt(encrypted: EncryptedMessage, sender: Recipient): Message {
    val session = getSession(sender)
    val cipher = SessionCipher(session)
    return cipher.decrypt(encrypted)
}
```

## Sealed Sender

Anonymous message delivery without revealing sender:

```kotlin
object SealedSenderAccessUtil {
    // Create sealed sender certificate
    // Verify sealed sender messages
}
```

Benefits:
- Sender privacy
- Server cannot identify sender
- Recipient can verify authenticity

## Group Messaging

### Sender Keys

For efficient group encryption:

1. Group creator generates sender key
2. Sender key is encrypted for each member
3. All messages use single sender key
4. Reduces encryption overhead

### Group Security

- Each group has unique sender keys
- Keys rotate when members leave
- New keys distributed via individual sessions

## Identity Verification

### Safety Numbers

Unique identifier for conversation participants:

```kotlin
// Calculate safety number
fun computeSafetyNumber(local: Recipient, remote: Recipient): String {
    // Derived from identity keys
    // Can be compared out-of-band
}
```

### Verification States

| State | Description |
|-------|-------------|
| Unverified | Default state |
| Verified | User has verified |
| Default | Safety number changed |

## Key Rotation

### Automatic Rotation

- Pre-keys rotate automatically
- Sender keys rotate on group changes
- Signed pre-keys rotate periodically

### Manual Rotation

```kotlin
object PreKeyUtil {
    fun rotateSignedPreKey()
    fun rotateOneTimePreKeys()
}
```

## Secure Random

Signal uses cryptographically secure random:

```kotlin
object SecureRandomProvider {
    fun nextBytes(size: Int): ByteArray {
        // Cryptographically secure random bytes
    }
}
```

## Network Security

### Certificate Pinning

Signal pins certificates to prevent MITM:

```kotlin
// Certificate pinning for API calls
object SignalServiceTrustStore {
    // Trusted certificates
}
```

### TLS 1.3

All network communication uses TLS 1.3.

### Proxy Support

Signal supports HTTPS proxies:

```kotlin
// Proxy configuration
class SignalProxy(val host: String, val port: Int)
```

## Security Best Practices

### For Developers

1. Never log sensitive data
2. Use constant-time comparisons
3. Clear sensitive data from memory
4. Validate all inputs
5. Use secure random sources

### For Users

1. Verify safety numbers
2. Use strong device passcode
3. Enable screen lock
4. Keep app updated
5. Report suspicious activity

## Common Security Files

| File | Purpose |
|------|---------|
| `MasterSecret.java` | Master secret management |
| `DatabaseSecret.java` | Database encryption key |
| `AttachmentSecret.java` | Attachment encryption key |
| `IdentityTable.kt` | Identity key storage |
| `SessionTable.kt` | Signal protocol sessions |
| `PreKeyUtil.java` | Pre-key management |
| `KeyStoreHelper.java` | Android Keystore wrapper |

## Further Reading

- [Signal Protocol Documentation](https://signal.org/docs/)
- [X3DH Key Agreement](https://signal.org/docs/specifications/x3dh/)
- [Double Ratchet Algorithm](https://signal.org/docs/specifications/doubleratchet/)