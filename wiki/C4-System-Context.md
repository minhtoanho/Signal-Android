# C4 System Context Diagram

> **Level 1**: Shows the Signal Android system in its environment, with users and external systems it interacts with.

## Diagram

```mermaid
C4Context
    title System Context Diagram - Signal Android

    Person(user, "Signal User", "Android device user who sends/receives encrypted messages")
    Person(contact, "Signal Contact", "Another Signal user communicating with the primary user")

    System(signalAndroid, "Signal Android", "End-to-end encrypted messaging application for Android")

    System_Ext(signalServer, "Signal Server", "Signal's cloud infrastructure for message routing, user discovery, and account management")
    System_Ext(signalCDN, "Signal CDN", "Content Delivery Network for attachments and media")
    System_Ext(signalSVR, "Secure Value Recovery (SVR2)", "Encrypted backup of account keys using SGX enclaves")
    System_Ext(signalSFU, "Selective Forwarding Unit", "WebRTC infrastructure for group calls")
    System_Ext(fcm, "Firebase Cloud Messaging", "Push notification delivery service")
    System_Ext(googlePlay, "Google Play Services", "App distribution and in-app billing")
    System_Ext(mobileCoin, "MobileCoin Network", "Cryptocurrency network for payments")
    System_Ext(giphy, "Giphy API", "GIF search and sharing")
    System_Ext(signaliOS, "Signal iOS", "iOS client for cross-platform messaging")
    System_Ext(signalDesktop, "Signal Desktop", "Desktop client for cross-platform messaging")

    Rel(user, signalAndroid, "Uses")
    Rel(contact, signaliOS, "Uses", "iOS")
    Rel(contact, signalDesktop, "Uses", "Desktop")
    Rel(contact, signalAndroid, "Uses", "Android")
    
    Rel(signalAndroid, signalServer, "Message routing, registration, user discovery", "WebSocket/HTTPS")
    Rel(signalAndroid, signalCDN, "Upload/download attachments", "HTTPS")
    Rel(signalAndroid, signalSVR, "Backup/recover account keys", "HTTPS/SGX")
    Rel(signalAndroid, signalSFU, "Group call media routing", "WebRTC")
    Rel(signalAndroid, fcm, "Receive push notifications", "FCM Protocol")
    Rel(signalAndroid, googlePlay, "App updates, donations", "Google Play APIs")
    Rel(signalAndroid, mobileCoin, "Send/receive payments", "MobileCoin Protocol")
    Rel(signalAndroid, giphy, "Search and send GIFs", "HTTPS API")
    
    Rel(signaliOS, signalServer, "Message routing", "WebSocket/HTTPS")
    Rel(signalDesktop, signalServer, "Message routing", "WebSocket/HTTPS")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## System Description

### Signal Android (This System)

The Signal Android application is the official Android client for Signal Messenger. It provides:

- **End-to-end encrypted messaging** using the Signal Protocol
- **Voice and video calls** via WebRTC
- **Group messaging** with support for varying membership permissions
- **Media sharing** with automatic encryption
- **Stories** - ephemeral content sharing
- **Payments** - MobileCoin cryptocurrency transfers
- **Multi-device support** - link desktop/iOS devices

### External Systems

| System | Purpose | Protocol | Data Flow |
|--------|---------|----------|-----------|
| **Signal Server** | Message routing, user registration, account management, contact discovery | WebSocket, HTTPS REST API | Bidirectional encrypted messages, user presence |
| **Signal CDN** | Storage and delivery of encrypted attachments | HTTPS | Upload/download encrypted blobs |
| **SVR2** | Secure backup of account keys using Intel SGX enclaves | HTTPS with SGX attestation | Encrypted key storage/retrieval |
| **Signal SFU** | Media routing for group video calls | WebRTC | Encrypted media streams |
| **Firebase Cloud Messaging** | Push notification delivery when app is backgrounded | FCM Protocol | Wake-up notifications only (no message content) |
| **Google Play Services** | App distribution, in-app billing for donations | Google Play APIs | App updates, donation processing |
| **MobileCoin Network** | Cryptocurrency transactions for payments | MobileCoin Protocol | Payment transactions |
| **Giphy API** | GIF search and sharing | HTTPS REST API | GIF metadata and thumbnails |
| **Signal iOS/Desktop** | Cross-platform communication | Via Signal Server | End-to-end encrypted messages |

### Users

| Actor | Description | Key Interactions |
|-------|-------------|------------------|
| **Signal User** | Primary Android device user | Register account, send messages, make calls, manage settings |
| **Signal Contact** | Other Signal users (on any platform) | Exchange messages and calls with the primary user |

## Trust Boundaries

```
┌─────────────────────────────────────────────────────────────────┐
│                      TRUST BOUNDARY                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Signal Android App                      │    │
│  │  - All cryptographic operations                          │    │
│  │  - Message encryption/decryption                         │    │
│  │  - Key management                                        │    │
│  │  - Local database (SQLCipher encrypted)                  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                    Untrusted Network
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SYSTEMS                             │
│  - Signal Server (never sees plaintext)                         │
│  - CDN (stores encrypted blobs only)                            │
│  - FCM (notification triggers only)                             │
│  - MobileCoin (public blockchain)                               │
└─────────────────────────────────────────────────────────────────┘
```

## Key Architectural Decisions

### End-to-End Encryption
All messages are encrypted on the sender's device and can only be decrypted by the intended recipients. Signal servers never have access to plaintext content.

### Minimal Server Trust
The architecture is designed to minimize trust in Signal's infrastructure:
- Servers cannot read message content
- Push notifications contain no message data
- Attachments are encrypted before upload

### Cross-Platform Synchronization
Users can link multiple devices (Android, iOS, Desktop) to a single account. Each device has its own cryptographic keys, and messages are encrypted separately for each device.

## Related Documentation

- [Container Diagram](C4-Container-Diagram.md) - Detailed view of Signal Android's internal containers
- [Business Domain](Business-Domain.md) - Core domain concepts
- [Security & Cryptography](Security-Cryptography.md) - Cryptographic implementation details