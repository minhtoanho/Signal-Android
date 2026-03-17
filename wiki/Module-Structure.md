# Module Structure

Signal Android uses a modular architecture to separate concerns and enable better build times and code organization.

## Module Overview

```
Signal-Android/
├── app/                    # Main application
├── core/                   # Core utilities
│   ├── util/              # Android utilities
│   ├── util-jvm/          # JVM-only utilities
│   ├── models/            # Android models
│   ├── models-jvm/        # JVM-only models
│   └── ui/                # Shared UI components
├── lib/                    # Library modules
│   ├── libsignal-service/ # Signal service communication
│   ├── billing/           # Google Play billing
│   ├── blurhash/          # BlurHash image encoding
│   ├── contacts/          # Contact utilities
│   ├── debuglogs-viewer/  # Debug log viewer
│   ├── device-transfer/   # Device transfer functionality
│   ├── donations/         # Donation handling
│   ├── glide/             # Glide image loading extensions
│   ├── image-editor/      # Image editing
│   ├── paging/            # Paging library extensions
│   ├── photoview/         # Photo viewing
│   ├── qr/                # QR code handling
│   ├── spinner/           # Loading spinner
│   ├── sticky-header-grid/# Sticky header grid
│   └── video/             # Video processing
├── feature/               # Feature modules
│   ├── camera/            # Camera functionality
│   ├── media-send/        # Media sending
│   └── registration/      # Registration flow
├── demo/                  # Demo applications
├── build-logic/           # Build plugins
├── lintchecks/            # Custom lint rules
├── benchmark/             # Benchmarking
├── baseline-profile/      # Baseline profiles
├── microbenchmark/        # Microbenchmarks
└── reproducible-builds/   # Reproducible builds
```

## Core Modules

### core:util
Android-specific utility classes:
- Extension functions
- Helper classes
- Android-specific utilities

### core:util-jvm
JVM-only utilities (no Android dependencies):
- Pure Kotlin/Java utilities
- Can be used in unit tests

### core:models
Android-specific model classes:
- Parcelable models
- Android-specific data classes

### core:models-jvm
JVM-only model classes:
- Platform-independent models
- Serializable data classes

### core:ui
Shared UI components:
- Compose components
- Theme definitions
- Common UI utilities

## Library Modules

### lib:libsignal-service
Core Signal service communication:
- API clients
- WebSocket handling
- Protocol implementation
- Message sending/receiving

### lib:billing
Google Play Billing integration:
- Subscription handling
- In-app purchases
- Donation badges

### lib:glide
Glide image loading extensions:
- Custom model loaders
- Decoders
- Transformations

### lib:image-editor
Image editing functionality:
- Crop, rotate, draw
- Text overlay
- Filters

### lib:paging
Paging 3 library extensions:
- Custom paging sources
- Paging helpers

### lib:qr
QR code handling:
- QR generation
- QR scanning

### lib:video
Video processing:
- Video compression
- Thumbnail generation
- Playback utilities

## Feature Modules

### feature:registration
User registration flow:
- Phone number verification
- PIN setup
- Account creation

### feature:camera
Camera functionality:
- Photo capture
- Video recording
- Camera preview

### feature:media-send
Media sending flow:
- Media selection
- Media editing
- Multi-media sending

## Demo Modules

Each demo module tests a specific library:
- `demo:paging` - Tests lib:paging
- `demo:camera` - Tests feature:camera
- `demo:qr` - Tests lib:qr
- `demo:video` - Tests lib:video
- `demo:image-editor` - Tests lib:image-editor

## Build Logic

### build-logic:plugins
Custom Gradle plugins:
- Android application configuration
- Library configuration
- Convention plugins

### build-logic:tools
Build tools and utilities:
- Custom tasks
- Build utilities

## Testing Modules

### lintchecks
Custom lint rules:
- Project-specific checks
- Code quality rules

### benchmark
Macrobenchmark tests:
- Startup performance
- Runtime performance

### microbenchmark
Microbenchmark tests:
- Code-level performance
- Algorithm benchmarks

### baseline-profile
Baseline profiles for:
- Faster startup
- Improved runtime performance

## Module Dependencies

```
                    app
                     │
         ┌──────────┼──────────┐
         │          │          │
     feature       lib        core
         │          │          │
         └──────────┴──────────┘
                    │
                  core
```

Key principles:
1. **app** depends on all modules
2. **feature** modules depend on **lib** and **core**
3. **lib** modules depend only on **core**
4. **core** modules have no internal dependencies
5. **demo** modules are standalone test apps