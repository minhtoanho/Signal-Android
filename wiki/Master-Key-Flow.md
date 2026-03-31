# Master Key Flow

> **This document explains Signal's key management architecture, including both the legacy MasterSecret (database encryption) and the modern MasterKey (SVR2 backup).**

## Overview

Signal uses two distinct "master key" concepts:

| Concept | Purpose | Location | Code File |
|---------|---------|----------|-----------|
| **MasterSecret** | Local database encryption | `app/.../crypto/` | [`MasterSecret.java:42`](app/src/main/java/org/thoughtcrime/securesms/crypto/MasterSecret.java#L42) |
| **MasterKey** | SVR2 backup/recovery of account keys | `core/models-jvm/` | [`MasterKey.kt:13`](core/models-jvm/src/main/java/org/signal/core/models/MasterKey.kt#L13) |

---

## 1. MasterSecret Flow (Legacy Database Encryption)

### Code Locations

| Component | File | Key Lines |
|-----------|------|-----------|
| MasterSecret class | `app/.../crypto/MasterSecret.java` | 42-120 |
| Key Generation | `app/.../crypto/MasterSecretUtil.java` | 170-193 |
| Key Retrieval | `app/.../crypto/MasterSecretUtil.java` | 106-128 |
| Encryption/Decryption | `app/.../crypto/MasterCipher.java` | 55-224 |
| In-Memory Cache | `app/.../service/KeyCachingService.java` | 62-330 |

### MasterSecret Structure

```
MasterSecret (128-bit AES + 160-bit HMAC)
├── encryptionKey: SecretKeySpec (128-bit AES)
└── macKey: SecretKeySpec (160-bit HMAC-SHA1)
```

**Reference:** [`MasterSecret.java:44-45`](app/src/main/java/org/thoughtcrime/securesms/crypto/MasterSecret.java#L44-L45)

### Flow 1.1: Key Generation

```
┌────────────────────────────────────────────────────────────────────┐
│                MASTERSECRET GENERATION FLOW                         │
│                MasterSecretUtil.generateMasterSecret():170-193      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  STEP 1: Generate Random Keys                                      │
│  ├── generateEncryptionSecret():248-259 → 128-bit AES key         │
│  └── generateMacSecret():261-269 → 160-bit HMAC key               │
│                                                                    │
│  STEP 2: Combine Secrets                                           │
│  └── Util.combine(encryptionSecret, macSecret):174                │
│                                                                    │
│  STEP 3: Generate Salt & Calculate Iterations                      │
│  ├── generateSalt():271-277 → 16 random bytes                     │
│  └── generateIterationCount():279-303 → adaptive count            │
│                                                                    │
│  STEP 4: Encrypt with Passphrase (PBE)                             │
│  └── encryptWithPassphrase():323-328                              │
│      └── PBEWITHSHA1AND128BITAES-CBC-BC                           │
│                                                                    │
│  STEP 5: MAC the Encrypted Data                                    │
│  └── macWithPassphrase():364-373                                  │
│                                                                    │
│  STEP 6: Store in SharedPreferences                                │
│  ├── "encryption_salt" → :181                                      │
│  ├── "mac_salt" → :182                                             │
│  ├── "passphrase_iterations" → :183                               │
│  ├── "master_secret" (encrypted + MAC) → :184                     │
│  └── "passphrase_initialized" = true → :185                       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 1.2: Key Retrieval (Unlock)

```
┌────────────────────────────────────────────────────────────────────┐
│                MASTERSECRET RETRIEVAL FLOW                          │
│                MasterSecretUtil.getMasterSecret():106-128           │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  STEP 1: Retrieve Stored Data                                      │
│  ├── retrieve("master_secret"):110                                 │
│  ├── retrieve("mac_salt"):111                                      │
│  ├── retrieve("passphrase_iterations"):112                         │
│  └── retrieve("encryption_salt"):114                               │
│                                                                    │
│  STEP 2: Verify MAC (Authentication)                               │
│  └── verifyMac():349-362                                           │
│      └── throws InvalidPassphraseException if wrong PIN           │
│                                                                    │
│  STEP 3: Decrypt with Passphrase                                   │
│  └── decryptWithPassphrase():330-335                               │
│                                                                    │
│  STEP 4: Split Combined Secrets                                    │
│  ├── encryptionSecret = split[0]:116 (16 bytes)                   │
│  └── macSecret = split[1]:117 (20 bytes)                          │
│                                                                    │
│  STEP 5: Create MasterSecret Object                                │
│  └── new MasterSecret(encryptionSecret, macSecret):119-120        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 1.3: Data Encryption/Decryption

```
┌────────────────────────────────────────────────────────────────────┐
│                MASTERCIPHER ENCRYPTION FLOW                         │
│                MasterCipher.encryptBytes():111-125                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ENCRYPT (encryptBytes:111-125):                                   │
│  ┌─────────────────────────────────────────────────┐              │
│  │ Input: plaintext bytes                          │              │
│  │                                                 │              │
│  │ Step 1: getEncryptingCipher():217-222           │              │
│  │         AES/CBC/PKCS5Padding, random IV         │              │
│  │                                                 │              │
│  │ Step 2: getEncryptedBody():181-190              │              │
│  │         Output: [16-byte IV][AES-CBC ciphertext]│              │
│  │                                                 │              │
│  │ Step 3: getMacBody():199-207                    │              │
│  │         HMAC-SHA1 over IV + ciphertext          │              │
│  │                                                 │              │
│  │ Output: [IV][ciphertext][20-byte HMAC]          │              │
│  └─────────────────────────────────────────────────┘              │
│                                                                    │
│  DECRYPT (decryptBytes:97-109):                                    │
│  ┌─────────────────────────────────────────────────┐              │
│  │ Input: [IV][ciphertext][HMAC]                   │              │
│  │                                                 │              │
│  │ Step 1: verifyMacBody():158-175                 │              │
│  │         Extract and verify HMAC                 │              │
│  │         throws InvalidMessageException if bad   │              │
│  │                                                 │              │
│  │ Step 2: getDecryptingCipher():209-215           │              │
│  │         Extract IV (first 16 bytes)             │              │
│  │                                                 │              │
│  │ Step 3: getDecryptedBody():177-179              │              │
│  │         AES-CBC decrypt remaining bytes         │              │
│  │                                                 │              │
│  │ Output: plaintext bytes                         │              │
│  └─────────────────────────────────────────────────┘              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 1.4: In-Memory Key Cache

```
┌────────────────────────────────────────────────────────────────────┐
│                KEYCACHINGSERVICE LIFECYCLE                          │
│                KeyCachingService.java:62-330                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  STATE MANAGEMENT:                                                 │
│  └── static masterSecret: MasterSecret = null:81                  │
│                                                                    │
│  CHECK LOCK STATE:                                                 │
│  └── isLocked():85-93                                              │
│      Returns true if masterSecret == null AND                      │
│      (passphraseEnabled OR screenLockEnabled)                      │
│                                                                    │
│  GET MASTER SECRET:                                                │
│  └── getMasterSecret():95-105                                      │
│      If disabled passphrase → auto-unlock with UNENCRYPTED_PASSPHRASE│
│                                                                    │
│  SET MASTER SECRET (after unlock):                                 │
│  └── setMasterSecret():118-132                                     │
│      ├── Store in static field:120                                 │
│      ├── foregroundService():122 (show lock notification)         │
│      ├── broadcastNewSecret():123 (notify app)                    │
│      └── startTimeoutIfAppropriate():124 (auto-lock timer)        │
│                                                                    │
│  CLEAR KEY (lock/manual):                                          │
│  └── handleClearKey():186-199                                      │
│      ├── masterSecret = null:188                                   │
│      ├── stopForeground(true):189                                  │
│      └── sendBroadcast(CLEAR_KEY_EVENT):194                       │
│                                                                    │
│  TIMEOUT MANAGEMENT:                                               │
│  └── startTimeoutIfAppropriate():225-265                           │
│      Uses AlarmManager to schedule auto-lock                       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 2. MasterKey Flow (SVR2 - Secure Value Recovery)

### Code Locations

| Component | File | Key Lines |
|-----------|------|-----------|
| MasterKey class | `core/models-jvm/.../MasterKey.kt` | 13-68 |
| Key Derivation | `core/models-jvm/.../MasterKey.kt` | 31-49 |
| SVR2 Client | `lib/libsignal-service/.../SecureValueRecoveryV2.kt` | 41-297 |
| PIN Hash Utilities | `lib/libsignal-service/.../PinHashUtil.kt` | 15-82 |

### MasterKey Structure

```
MasterKey (256-bit single key)
└── masterKey: ByteArray (32 bytes)

Derives multiple keys via HMAC-SHA256:
├── deriveRegistrationLock() → "Registration Lock"
├── deriveRegistrationRecoveryPassword() → "Registration Recovery"
├── deriveStorageServiceKey() → "Storage Service Encryption"
└── deriveLoggingKey() → "Logging Key"
```

**Reference:** [`MasterKey.kt:13-49`](core/models-jvm/src/main/java/org/signal/core/models/MasterKey.kt#L13-L49)

### Flow 2.1: SVR2 Backup (Set PIN)

```
┌────────────────────────────────────────────────────────────────────┐
│                SVR2 BACKUP FLOW                                     │
│                SecureValueRecoveryV2.Svr2PinChangeSession:182-283   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ENTRY: setPin(userPin, masterKey):53-55                           │
│         Returns Svr2PinChangeSession                               │
│                                                                    │
│  EXECUTE SESSION (execute:198-236):                                │
│                                                                    │
│  STEP 1: Normalize PIN                                             │
│  └── PinHashUtil.normalize(userPin):199                            │
│      → normalizeToString():65-73 → UTF-8 bytes                    │
│                                                                    │
│  STEP 2: Get Authorization                                         │
│  └── authorization():202 → GET /v2/svr/auth                        │
│      Returns AuthCredentials (username, password)                  │
│                                                                    │
│  STEP 3: Create Backup Request (getBackupResponse:242-256)         │
│  ├── PinHash.svr2(normalizedPin, username, mrEnclave):243         │
│  │   → Creates PIN-derived access key                             │
│  ├── PinHashUtil.createNewKbsData(pinHash, masterKey):244         │
│  │   → HmacSIV.encrypt(pinHash.encryptionKey(), masterKey):47-48 │
│  │   → Returns KbsData(kbsAccessKey, cipherText)                 │
│  └── POST /v2/svr/backup:245-251                                   │
│      {pin: kbsAccessKey, data: cipherText, maxTries: 10}          │
│                                                                    │
│  STEP 4: Expose Data (getExposeResponse:258-282)                   │
│  └── POST /v2/svr/expose:261-265                                   │
│      {data: cipherText}                                            │
│      Makes data eligible for restore                               │
│                                                                    │
│  RESULT: BackupResponse.Success(masterKey, authorization, SVR2)   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 2.2: SVR2 Restore (Recover PIN)

```
┌────────────────────────────────────────────────────────────────────┐
│                SVR2 RESTORE FLOW                                    │
│                SecureValueRecoveryV2.restoreData():111-169          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  STEP 1: Normalize PIN                                             │
│  └── PinHashUtil.normalize(userPin):112                            │
│                                                                    │
│  STEP 2: Get Authorization                                         │
│  └── fetchAuth():115 → GET /v2/svr/auth                            │
│      Returns AuthCredentials                                       │
│                                                                    │
│  STEP 3: Derive PIN Access Key                                     │
│  └── PinHash.svr2(normalizedPin, username, mrEnclave):121          │
│      .accessKey() → used as authentication token                   │
│                                                                    │
│  STEP 4: Send Restore Request                                      │
│  └── POST /v2/svr/restore:117-124                                  │
│      {pin: accessKey}                                              │
│                                                                    │
│  STEP 5: Process Response (126-154):                               │
│                                                                    │
│      CASE OK:                                                      │
│      ├── Extract ciphertext:128                                    │
│      ├── Recreate pinHash:130                                      │
│      ├── PinHashUtil.decryptSvrDataIVCipherText():131              │
│      │   → HmacSIV.decrypt(pinHash.encryptionKey(), ivc):57-58   │
│      └── Return Success(masterKey, authorization):132             │
│                                                                    │
│      CASE PIN_MISMATCH:                                            │
│      └── Return PinMismatch(tries):144                             │
│          Wrong PIN, decrement guess count                          │
│                                                                    │
│      CASE MISSING:                                                 │
│      └── Return Missing:140                                        │
│          No backup data found                                      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 2.3: Key Derivation from MasterKey

```
┌────────────────────────────────────────────────────────────────────┐
│                MASTERKEY DERIVATION                                 │
│                MasterKey.derive():47-49                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  INPUT: masterKey (32 bytes)                                       │
│                                                                    │
│  DERIVE KEY FOR PURPOSE:                                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ derive(keyName: String): ByteArray?                          │  │
│  │                                                              │  │
│  │ return CryptoUtil.hmacSha256(                               │  │
│  │     masterKey,                                              │  │
│  │     keyName.toByteArray(UTF_8)                              │  │
│  │ )                                                           │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  DERIVED KEYS:                                                     │
│  ├── deriveRegistrationLock():31-33                                │
│  │   → HMAC-SHA256(masterKey, "Registration Lock")               │
│  │   → Returns hex string                                        │
│  │                                                                │
│  ├── deriveRegistrationRecoveryPassword():35-37                   │
│  │   → HMAC-SHA256(masterKey, "Registration Recovery")           │
│  │   → Returns Base64 string                                     │
│  │                                                                │
│  ├── deriveStorageServiceKey():39-41                              │
│  │   → HMAC-SHA256(masterKey, "Storage Service Encryption")      │
│  │   → Returns StorageKey                                        │
│  │                                                                │
│  └── deriveLoggingKey():43-45                                     │
│      → HMAC-SHA256(masterKey, "Logging Key")                      │
│      → Returns ByteArray                                          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. Complete Flow Diagrams

### Flow 3.1: App Startup with Passphrase

```
┌─────────────────────────────────────────────────────────────────────┐
│                     APP STARTUP FLOW                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. KeyCachingService.onCreate():153-165                            │
│     └── if (passphraseDisabled && !screenLockEnabled)               │
│         └── Auto-unlock with UNENCRYPTED_PASSPHRASE                │
│                                                                     │
│  2. User Enters Passphrase                                          │
│     └── MasterSecretUtil.getMasterSecret(context, pin):106-128     │
│         ├── verifyMac() → authenticate user                        │
│         ├── decryptWithPassphrase() → recover keys                 │
│         └── return MasterSecret                                    │
│                                                                     │
│  3. KeyCachingService.setMasterSecret():118-132                     │
│     ├── Store in static field                                      │
│     ├── Show lock notification                                     │
│     └── Broadcast NEW_KEY_EVENT                                    │
│                                                                     │
│  4. App Components Receive Key                                      │
│     └── Can now encrypt/decrypt database and preferences           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Flow 3.2: Registration with PIN/SVR2

```
┌─────────────────────────────────────────────────────────────────────┐
│                     REGISTRATION SVR2 FLOW                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Generate MasterKey                                              │
│     └── MasterKey.createNew(secureRandom):19-24                     │
│         → 32 random bytes                                           │
│                                                                     │
│  2. User Sets PIN                                                   │
│     └── SecureValueRecoveryV2.setPin(userPin, masterKey):53-55     │
│         → Creates Svr2PinChangeSession                             │
│                                                                     │
│  3. Execute PIN Change                                              │
│     └── session.execute():198-236                                   │
│         ├── POST /v2/svr/backup (encrypted masterKey)              │
│         └── POST /v2/svr/expose (make restorable)                  │
│                                                                     │
│  4. Store Registration Lock Locally                                 │
│     └── masterKey.deriveRegistrationLock():31-33                    │
│         → Used for re-registration verification                    │
│                                                                     │
│  5. Derive Storage Service Key                                      │
│     └── masterKey.deriveStorageServiceKey():39-41                   │
│         → Used for cloud backup sync                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Flow 3.3: Account Recovery

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ACCOUNT RECOVERY FLOW                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. User Enters PIN                                                 │
│     └── normalize(pin):79-81                                        │
│                                                                     │
│  2. Request Auth Credentials                                        │
│     └── GET /v2/svr/auth:102-105                                    │
│         → Returns username for PIN hashing                         │
│                                                                     │
│  3. Derive Access Key                                               │
│     └── PinHash.svr2(normalizedPin, username, mrEnclave)           │
│         → Uses SGX enclave identifier                              │
│                                                                     │
│  4. Restore from SVR2                                               │
│     └── POST /v2/svr/restore {pin: accessKey}                       │
│         ├── OK: Returns encrypted masterKey                        │
│         ├── PIN_MISMATCH: Wrong PIN, tries remaining              │
│         └── MISSING: No backup exists                              │
│                                                                     │
│  5. Decrypt MasterKey                                               │
│     └── PinHashUtil.decryptSvrDataIVCipherText():56-59             │
│         → HmacSIV.decrypt(encryptionKey, ciphertext)               │
│         → Returns MasterKey                                        │
│                                                                     │
│  6. Derive Account Keys                                             │
│     ├── deriveRegistrationLock() → verify identity                 │
│     └── deriveStorageServiceKey() → restore backup                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Key Differences Summary

| Aspect | MasterSecret | MasterKey |
|--------|--------------|-----------|
| **Size** | 128-bit AES + 160-bit MAC | 256-bit single key |
| **Purpose** | Local database encryption | Deriving other keys + cloud backup |
| **Storage** | SharedPreferences (encrypted) | SVR2 server (encrypted) |
| **Protection** | User passphrase | User PIN + Intel SGX enclave |
| **Recovery** | Local only | Cloud-based with PIN |
| **Key Code** | [`MasterSecret.java:42`](app/src/main/java/org/thoughtcrime/securesms/crypto/MasterSecret.java#L42) | [`MasterKey.kt:13`](core/models-jvm/src/main/java/org/signal/core/models/MasterKey.kt#L13) |

## Related Documentation

- [Security & Cryptography](Security-Cryptography.md) - Overall security architecture
- [Database](Database.md) - SQLCipher database encryption
- [Business Domain - Identity Context](Business-Domain.md#identity-context) - Key management domain concepts