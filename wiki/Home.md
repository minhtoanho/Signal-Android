# Signal Android

Signal is a simple, powerful, and secure messenger that uses your phone's data connection (WiFi/3G/4G/5G) to communicate securely.

## Overview

Signal's advanced privacy-preserving technology is always enabled, so you can focus on sharing the moments that matter with the people who matter to you.

| Attribute | Value |
|-----------|-------|
| **Platform** | Android (minSdk 23, targetSdk 35) |
| **Language** | Kotlin, Java |
| **License** | GNU AGPLv3 |
| **Current Version** | 8.3.2 |

## Architecture Documentation (C4 Model)

This wiki follows the **C4 Model** for architectural documentation, providing views at different levels of abstraction.

| Level | Document | Audience |
|-------|----------|----------|
| 1 | [System Context](C4-System-Context.md) | All stakeholders |
| 2 | [Container Diagram](C4-Container-Diagram.md) | Developers, Architects |
| 3 | [Component Diagram](C4-Component-Diagram.md) | Developers |

## Domain Documentation

| Document | Description |
|----------|-------------|
| [Business Domain](Business-Domain.md) | Core domain concepts, bounded contexts, and ubiquitous language |
| [Use Cases](Use-Cases.md) | Key user interactions and business workflows |

## Technical Documentation

| Document | Description |
|----------|-------------|
| [Architecture](Architecture.md) | Overview of the application architecture |
| [Module Structure](Module-Structure.md) | Understanding the modular architecture |
| [Database](Database.md) | Database layer documentation |
| [Job System](Job-System.md) | Background job management and scheduling |
| [Background Tasks](Background-Tasks.md) | Periodic tasks, foreground services, keep-alive mechanisms |
| [Media Lifecycle](Media-Lifecycle.md) | Attachment download, upload, retention, and storage |
| [Security & Cryptography](Security-Cryptography.md) | Security implementation details |
| [Signal Protocol Integration Guide](Signal-Protocol-Integration-Guide.md) | **How to integrate Signal Protocol into your app** |
| [Signal Protocol Messaging](Signal-Protocol-Messaging.md) | Session handshake, encryption, decryption, retry flows |
| [Signal Protocol Code Level](Signal-Protocol-Code-Level.md) | **C4 Level 4: Deep dive into classes, methods, algorithms** |
| [Master Key Flow](Master-Key-Flow.md) | Key management: MasterSecret & MasterKey (SVR2) |
| [Building](Building.md) | How to build the project |
| [Testing](Testing.md) | Testing guidelines |
| [Code Style Guidelines](Code-Style-Guidelines.md) | Coding conventions |
| [Contributing](Contributing.md) | How to contribute |
| [Features](Features.md) | Feature overview |

## Project Structure

```
Signal-Android/
├── app/                    # Main application module
├── core/                   # Core utilities and models
├── lib/                    # Library modules
├── feature/                # Feature modules
├── demo/                   # Demo applications
├── build-logic/            # Build logic and plugins
├── lintchecks/             # Custom lint rules
├── benchmark/              # Benchmarking modules
└── reproducible-builds/    # Reproducible build configuration
```

## Quick Navigation

### For New Developers
1. Start with [System Context](C4-System-Context.md) to understand the system's place in the world
2. Read [Business Domain](Business-Domain.md) to understand what Signal does
3. Review [Building](Building.md) to set up your environment

### For Architects
1. Review [Container Diagram](C4-Container-Diagram.md) for system structure
2. Dive into [Component Diagram](C4-Component-Diagram.md) for detailed architecture
3. Study [Architecture](Architecture.md) for technology decisions

### For Security Researchers
1. Read [Security & Cryptography](Security-Cryptography.md) for encryption details
2. Review [Database](Database.md) for data handling
3. Study the [Business Domain](Business-Domain.md) for trust boundaries

## Available Platforms

- **Android**: This repository
- **iOS**: [signal-ios](https://github.com/signalapp/signal-ios)
- **Desktop**: [signal-desktop](https://github.com/signalapp/signal-desktop)

## Download

- [Google Play Store](https://play.google.com/store/apps/details?id=org.thoughtcrime.securesms)
- [signal.org](https://signal.org/android/apk/)

## License

Copyright 2013-2025 Signal Messenger, LLC

Licensed under the GNU AGPLv3: https://www.gnu.org/licenses/agpl-3.0.html