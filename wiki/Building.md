# Building Signal Android

## Prerequisites

1. **Android Studio**: Latest stable version (Koala or newer recommended)
2. **JDK**: Java 17 or higher
3. **Android SDK**: API 34 (Android 14)
4. **NDK**: Version specified in `reproducible-builds/Dockerfile`
5. **CMake**: Version specified in `reproducible-builds/Dockerfile`

## Initial Setup

### 1. Clone the Repository

```bash
git clone https://github.com/signalapp/Signal-Android.git
cd Signal-Android
```

### 2. Open in Android Studio

1. Open Android Studio
2. Select "Open an existing project"
3. Navigate to the cloned directory and select it
4. Wait for Gradle sync to complete

### 3. Install Required SDK Components

Open SDK Manager in Android Studio and install:
- Android SDK Platform 34
- Android SDK Build-Tools
- NDK (version from Dockerfile)
- CMake (version from Dockerfile)

## Build Variants

Signal has multiple build variants:

| Variant | Description |
|---------|-------------|
| `playProdDebug` | Debug build with Play Services (development) |
| `playProdRelease` | Release build with Play Services |
| `playStagingDebug` | Debug build for staging server |
| `playStagingRelease` | Release build for staging server |
| `websiteProdRelease` | Release for website distribution |
| `githubProdRelease` | Release for GitHub distribution |

## Build Commands

### Using Android Studio

1. Select Build Variant from "Build Variants" tab
2. Click Build > Make Project (or Ctrl+F9)
3. Run > Run 'app' (or Shift+F10)

### Using Command Line

```bash
# Debug build
./gradlew assemblePlayProdDebug

# Release build (requires signing config)
./gradlew assemblePlayProdRelease

# Clean build
./gradlew clean assemblePlayProdDebug

# Run tests
./gradlew testPlayProdDebugUnitTest

# Run lint
./gradlew lintPlayProdRelease
```

## Code Formatting

Signal uses ktlint for code formatting:

```bash
# Check formatting
./gradlew ktlintCheck

# Auto-format code
./gradlew format
```

You can also set up a git hook using `lefthook.yml` configuration.

## Quality Assurance

Run the full QA suite before pushing:

```bash
./gradlew qa
```

This runs:
- Clean build
- ktlint checks
- Unit tests
- Lint checks
- Build logic tests

## Reproducible Builds

Signal supports reproducible builds for security verification. See `reproducible-builds/README.md` for details.

### Using Docker

```bash
cd reproducible-builds
docker build -t signal-android .
docker run -v $(pwd)/out:/out signal-android
```

## Build Configuration

### local.properties (Optional)

Create `local.properties` for local configuration:

```properties
# For quickstart builds
quickstart.credentials.dir=/path/to/credentials

# For benchmark testing
benchmark.backup.file=/path/to/backup
```

### Build Types

- **Debug**: Debuggable, unsigned, development builds
- **Release**: Optimized, signed, production builds
- **Spinner**: Performance testing variant
- **Perf**: Performance-optimized variant
- **Canary**: Canary release variant

## Troubleshooting

### Gradle Sync Fails

1. Ensure JDK 17+ is installed
2. Check Android SDK location in `local.properties`
3. Invalidate caches: File > Invalidate Caches / Restart

### NDK Issues

1. Install correct NDK version from SDK Manager
2. Check `ndk.dir` in `local.properties` if needed
3. Clean and rebuild: `./gradlew clean`

### Out of Memory

Add to `gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC
org.gradle.daemon=true
org.gradle.parallel=true
```

### Dependency Issues

```bash
# Refresh dependencies
./gradlew --refresh-dependencies build
```