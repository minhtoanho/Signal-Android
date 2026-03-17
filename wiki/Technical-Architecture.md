# Technical Architecture

> Detailed technology stack, libraries, and architectural decisions for Signal Android.

## Technology Stack

### Languages

| Language | Usage | Percentage |
|----------|-------|------------|
| **Kotlin** | Primary development language | ~85% |
| **Java** | Legacy code, performance-critical sections | ~10% |
| **C/C++ (JNI)** | Native crypto, WebRTC, media processing | ~5% |

### Build System

| Tool | Version | Purpose |
|------|---------|---------|
| **Gradle** | 8.x | Build automation |
| **Kotlin DSL** | - | Build scripts |
| **KSP** | - | Annotation processing |
| **Wire** | 4.4.3 | Protocol buffer code generation |

### Android SDK

| Component | Version |
|-----------|---------|
| **minSdk** | 23 (Android 6.0) |
| **targetSdk** | 35 (Android 15) |
| **compileSdk** | 35 |
| **Build Tools** | 35.0.0 |
| **NDK** | r27 |

---

## Core Dependencies

### Cryptography

| Library | Purpose | License |
|---------|---------|---------|
| **libsignal** | Signal Protocol implementation | AGPLv3 |
| **SQLCipher** | Encrypted SQLite database | Apache 2.0 |
| **Conscrypt** | TLS implementation | Apache 2.0 |
| **AES-GCM Provider** | Hardware-accelerated AES | Apache 2.0 |

```kotlin
implementation("org.signal:libsignal-android:0.60.0")
implementation("net.zetetic:sqlcipher-android:4.5.7")
implementation("org.conscrypt:conscrypt-android:2.5.3")
```

### Networking

| Library | Purpose | License |
|---------|---------|---------|
| **OkHttp** | HTTP client | Apache 2.0 |
| **Okio** | I/O utilities | Apache 2.0 |
| **RxJava** | Reactive streams | Apache 2.0 |
| **Kotlinx Coroutines** | Async programming | Apache 2.0 |

```kotlin
implementation("com.squareup.okhttp3:okhttp:4.12.0")
implementation("io.reactivex.rxjava3:rxjava:3.1.8")
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
```

### Media Processing

| Library | Purpose | License |
|---------|---------|---------|
| **ExoPlayer / Media3** | Video playback | Apache 2.0 |
| **Glide** | Image loading | BSD |
| **CameraX** | Camera access | Apache 2.0 |
| **RingRTC** | WebRTC for Signal | GPL |

```kotlin
implementation("androidx.media3:media3-exoplayer:1.4.1")
implementation("com.github.bumptech.glide:glide:4.16.0")
implementation("androidx.camera:camera-camera2:1.3.4")
implementation("org.signal:ringrtc:2.41.3")
```

### UI Framework

| Library | Purpose | License |
|---------|---------|---------|
| **Jetpack Compose** | Modern declarative UI | Apache 2.0 |
| **Material Design** | UI components | Apache 2.0 |
| **AndroidX** | Support libraries | Apache 2.0 |
| **Lottie** | Animations | Apache 2.0 |

```kotlin
implementation(platform("androidx.compose:compose-bom:2024.10.01"))
implementation("androidx.compose.ui:ui")
implementation("androidx.compose.material3:material3")
implementation("com.airbnb.android:lottie:6.5.2")
```

### Payments

| Library | Purpose | License |
|---------|---------|---------|
| **MobileCoin SDK** | Cryptocurrency transactions | Apache 2.0 |

```kotlin
implementation("com.mobilecoin:android-sdk:7.1.1")
```

### Google Services

| Service | Purpose |
|---------|---------|
| **Firebase Cloud Messaging** | Push notifications |
| **Google Play Billing** | Donations |
| **Google Play Services Auth** | Account linking |
| **Maps SDK** | Location sharing |

---

## Architecture Patterns

### MVVM (Model-View-ViewModel)

All screens follow the MVVM pattern:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   View      │────▶│  ViewModel  │────▶│  Repository │
│  (Compose)  │◀────│ (RxJava/    │◀────│  (Database) │
│             │     │  LiveData)  │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
```

### Repository Pattern

Data access is abstracted through repositories:

| Repository | Data Source |
|------------|-------------|
| `ConversationRepository` | `MessageTable`, `ThreadTable` |
| `RecipientRepository` | `RecipientTable` |
| `GroupRepository` | `GroupTable` |
| `PaymentRepository` | `PaymentTable` |

### Job Queue Pattern

Background work uses a custom job queue:

```kotlin
abstract class Job(
    val parameters: Parameters = Parameters()
) {
    abstract fun run(): Result
    abstract fun serialize(): Data
    abstract fun getFactoryKey(): String
}

// Usage
jobManager.add(MessageSendJob(messageId))
```

### Event Bus

Cross-component communication uses EventBus:

```kotlin
@Subscribe(threadMode = ThreadMode.MAIN)
fun onEvent(event: MessageSentEvent) {
    // Handle event
}
```

---

## Data Flow Architecture

### Message Sending Flow

```
User Input
    │
    ▼
┌─────────────────┐
│   ViewModel     │
│  (UI State)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  MessageSender  │
│  (Orchestrator) │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│ Crypto│ │  DB   │
│(Encrypt)│(Store)│
└───┬───┘ └───┬───┘
    │         │
    └────┬────┘
         ▼
┌─────────────────┐
│ SignalService   │
│   (Network)     │
└─────────────────┘
```

### Message Receiving Flow

```
FCM Push
    │
    ▼
┌─────────────────┐
│ PushReceiver    │
│ (Wake App)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ MessageReceiver │
│ (Fetch+Decrypt) │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│ Crypto│ │  DB   │
│(Decrypt)│(Store)│
└───────┘ └───┬───┘
              │
              ▼
┌─────────────────┐
│  UI Observer    │
│  (Display)      │
└─────────────────┘
```

---

## Security Architecture

### Key Hierarchy

```
┌─────────────────────────────────────────┐
│           Android Keystore              │
│  ┌─────────────────────────────────┐    │
│  │   Master Key (Hardware-backed)  │    │
│  └─────────────────────────────────┘    │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│         Database Key (Encrypted)        │
│  - Stored in EncryptedSharedPreferences │
│  - Protected by Master Key              │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│            SQLCipher Database           │
│  - All tables encrypted at rest         │
│  - Transparent encryption/decryption    │
└─────────────────────────────────────────┘
```

### Signal Protocol Keys

| Key Type | Purpose | Lifetime |
|----------|---------|----------|
| **Identity Key** | User identity | Long-term |
| **Signed PreKey** | New session establishment | Weekly rotation |
| **One-Time PreKey** | First message | Single use |
| **Kyber PreKey** | Post-quantum sessions | Weekly rotation |
| **Sender Key** | Group encryption | Per-group |

### Network Security

| Layer | Protection |
|-------|------------|
| **Transport** | TLS 1.3 |
| **Application** | Signal Protocol encryption |
| **Certificate Pinning** | Signal server certificates |
| **Certificate Transparency** | Log verification |

---

## Performance Optimizations

### Database

- **WAL Mode**: Write-ahead logging for concurrent reads/writes
- **Indexes**: Strategic indexing on query columns
- **Connection Pooling**: Multiple database connections
- **Batch Operations**: Transactional bulk inserts

### UI

- **Compose**: Efficient recomposition
- **RecyclerView**: View recycling for lists
- **DiffUtil**: Minimal updates on data changes
- **ViewBinding**: Null-safe view access

### Memory

- **Glide**: Image memory caching
- **Weak References**: Avoid memory leaks
- **Lifecycle Awareness**: Release resources when not visible

### Network

- **WebSocket**: Persistent connection for real-time
- **Request Deduplication**: Avoid duplicate requests
- **Retry with Backoff**: Graceful error handling

---

## Build Variants

| Variant | Environment | Minification | Use Case |
|---------|-------------|--------------|----------|
| **playProdDebug** | Production | No | Development |
| **playProdRelease** | Production | Yes | Play Store release |
| **websiteProdRelease** | Production | Yes | APK direct distribution |
| **playStagingDebug** | Staging | No | Testing with staging servers |
| **nightlyProdRelease** | Production | Yes | Automated nightly builds |

---

## Related Documentation

- [C4 Container Diagram](C4-Container-Diagram.md) - High-level architecture
- [Database](Database.md) - Database layer details
- [Security & Cryptography](Security-Cryptography.md) - Security implementation
- [Module Structure](Module-Structure.md) - Module organization