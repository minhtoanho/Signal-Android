# Architecture

Signal Android is built on a modular architecture with clear separation of concerns.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        App Module                           │
│  (UI, Activities, Fragments, ViewModels, Services)          │
├─────────────────────────────────────────────────────────────┤
│                     Feature Modules                         │
│  (registration, camera, media-send)                        │
├─────────────────────────────────────────────────────────────┤
│                      Lib Modules                            │
│  (libsignal-service, glide, paging, qr, video, etc.)        │
├─────────────────────────────────────────────────────────────┤
│                      Core Modules                           │
│  (util, models, ui)                                         │
└─────────────────────────────────────────────────────────────┘
```

## Key Components

### UI Layer
- **Activities**: Main entry points for screens
- **Fragments**: Reusable UI components
- **ViewModels**: Manage UI state and business logic
- **Compose**: Modern UI using Jetpack Compose

### Data Layer
- **Database**: SQLite with SQLCipher encryption
- **Key-Value Store**: Simple key-value persistence
- **Preferences**: User settings storage
- **Storage Service**: Encrypted cloud backup sync

### Network Layer
- **libsignal-service**: Communication with Signal servers
- **WebSocket**: Real-time message delivery
- **Push**: FCM for notifications
- **REST API**: Server communication

### Security Layer
- **Crypto**: End-to-end encryption using Signal Protocol
- **Key Management**: Secure key storage and handling
- **Database Encryption**: SQLCipher for data at rest

## Package Structure (app module)

```
org.thoughtcrime.securesms/
├── attachments/      # Attachment handling
├── audio/            # Audio playback/recording
├── backup/           # Backup/restore functionality
├── calls/            # Voice/video calls (WebRTC)
├── components/       # Shared UI components
├── compose/          # Jetpack Compose components
├── contacts/         # Contact management
├── conversation/     # Chat UI and logic
├── conversationlist/ # Conversation list UI
├── crypto/           # Cryptographic operations
├── database/         # Database tables and helpers
├── groups/           # Group chat functionality
├── jobs/             # Background job processing
├── messages/         # Message processing
├── net/              # Networking utilities
├── notifications/    # Push notifications
├── payments/         # MobileCoin payments
├── push/             # Push notification handling
├── recipients/       # Recipient management
├── registration/     # User registration flow
├── service/          # Background services
├── storage/          # Storage service sync
├── stickers/         # Sticker packs
├── stories/          # Stories feature
└── webrtc/           # WebRTC call handling
```

## Job System

Signal uses a robust job queue system for background operations:

- **JobManager**: Manages job execution
- **Jobs**: Individual units of work
- **Constraints**: Conditions for job execution
- **Retry Logic**: Automatic retry with backoff

## Threading Model

- **Main Thread**: UI updates only
- **Background Threads**: Database, network, crypto operations
- **Coroutines**: Kotlin coroutines for async operations
- **WorkManager**: System-level background work

## Dependency Injection

The app uses a service locator pattern with:
- **ApplicationContext**: Central dependency container
- **Lazy initialization**: Dependencies created on demand
- **Singletons**: Shared instances across the app