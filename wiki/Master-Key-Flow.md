# Master Key Flow

> **This document explains Signal's key management architecture, including both the legacy MasterSecret (database encryption) and the modern MasterKey (SVR2 backup).**

## Overview

Signal uses two distinct "master key" concepts:

| Concept | Purpose | Location |
|---------|---------|----------|
| **MasterSecret** | Local database encryption | `app/.../crypto/` |
| **MasterKey** | SVR2 backup/recovery of account keys | `core/models-jvm/`, `lib/libsignal-service/` |

---

## 1. MasterSecret (Legacy Database Encryption)

The `MasterSecret` is used to encrypt local data at rest. It consists of two keys:

- **Encryption Key**: 128-bit AES key for encrypting local data
- **MAC Key**: 160-bit HMAC-SHA1 key for integrity verification

### Key Generation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    MasterSecret Generation                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Generate Random Keys                                        │
│     ├── encryptionSecret = KeyGenerator("AES").init(128)        │
│     └── macSecret = KeyGenerator("HmacSHA1").generate()         │
│                                                                 │
│  2. Combine and Encrypt with Passphrase                         │
│     ├── combinedSecrets = combine(encryptionSecret, macSecret)  │
│     ├── encryptionSalt = generateSalt()        // 16 bytes     │
│     ├── iterations = calculateIterations()     // adaptive      │
│     └── encryptedSecret = PBE.encrypt(combinedSecrets, pin)     │
│                                                                 │
│  3. MAC the Encrypted Data                                      │
│     ├── macSalt = generateSalt()               // 16 bytes     │
│     └── maccedData = HMAC(encryptedSecret, pin, macSalt)        │
│                                                                 │
│  4. Store Securely                                              │
│     ├── SharedPreferences: encryption_salt                      │
│     ├── SharedPreferences: mac_salt                             │
│     ├── SharedPreferences: passphrase_iterations                │
│     └── SharedPreferences: master_secret (encrypted + MAC)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Code Implementation

#### MasterSecret.java

```java
public class MasterSecret implements Parcelable {
    private final SecretKeySpec encryptionKey;  // 128-bit AES
    private final SecretKeySpec macKey;         // 160-bit HMAC-SHA1

    public MasterSecret(SecretKeySpec encryptionKey, SecretKeySpec macKey) {
        this.encryptionKey = encryptionKey;
        this.macKey = macKey;
    }

    public SecretKeySpec getEncryptionKey() { return this.encryptionKey; }
    public SecretKeySpec getMacKey() { return this.macKey; }
}
```

#### MasterSecretUtil.java - Key Generation

```java
public static MasterSecret generateMasterSecret(Context context, String passphrase) {
    // 1. Generate random secrets
    byte[] encryptionSecret = generateEncryptionSecret();  // 128-bit AES
    byte[] macSecret = generateMacSecret();                 // 160-bit HMAC
    
    // 2. Combine secrets
    byte[] masterSecret = Util.combine(encryptionSecret, macSecret);
    
    // 3. Generate salts and calculate iterations
    byte[] encryptionSalt = generateSalt();  // 16 random bytes
    int iterations = generateIterationCount(passphrase, encryptionSalt);
    
    // 4. Encrypt with passphrase using PBE
    byte[] encryptedMasterSecret = encryptWithPassphrase(
        encryptionSalt, iterations, masterSecret, passphrase
    );
    
    // 5. MAC the encrypted data
    byte[] macSalt = generateSalt();
    byte[] encryptedAndMacdMasterSecret = macWithPassphrase(
        macSalt, iterations, encryptedMasterSecret, passphrase
    );
    
    // 6. Store in SharedPreferences (Base64 encoded)
    save(context, "encryption_salt", encryptionSalt);
    save(context, "mac_salt", macSalt);
    save(context, "passphrase_iterations", iterations);
    save(context, "master_secret", encryptedAndMacdMasterSecret);
    
    return new MasterSecret(
        new SecretKeySpec(encryptionSecret, "AES"),
        new SecretKeySpec(macSecret, "HmacSHA1")
    );
}
```

#### MasterSecretUtil.java - Key Retrieval

```java
public static MasterSecret getMasterSecret(Context context, String passphrase)
    throws InvalidPassphraseException {
    
    // 1. Retrieve stored data
    byte[] encryptedAndMacdMasterSecret = retrieve(context, "master_secret");
    byte[] macSalt = retrieve(context, "mac_salt");
    int iterations = retrieve(context, "passphrase_iterations", 100);
    byte[] encryptionSalt = retrieve(context, "encryption_salt");
    
    // 2. Verify MAC (fails if passphrase is wrong)
    byte[] encryptedMasterSecret = verifyMac(
        macSalt, iterations, encryptedAndMacdMasterSecret, passphrase
    );
    
    // 3. Decrypt with passphrase
    byte[] combinedSecrets = decryptWithPassphrase(
        encryptionSalt, iterations, encryptedMasterSecret, passphrase
    );
    
    // 4. Split back into component keys
    byte[][] parts = Util.split(combinedSecrets, 16, 20);
    byte[] encryptionSecret = parts[0];
    byte[] macSecret = parts[1];
    
    return new MasterSecret(
        new SecretKeySpec(encryptionSecret, "AES"),
        new SecretKeySpec(macSecret, "HmacSHA1")
    );
}
```

### Encryption/Decryption with MasterCipher

The `MasterCipher` class provides encrypt/decrypt operations using the MasterSecret:

```java
public class MasterCipher {
    private final MasterSecret masterSecret;
    private final Cipher encryptingCipher;  // AES/CBC/PKCS5Padding
    private final Mac hmac;                 // HmacSHA1

    /**
     * Encrypt format:
     * [16-byte IV][AES-CBC encrypted data][20-byte HMAC]
     */
    public byte[] encryptBytes(byte[] body) {
        // 1. Encrypt with AES-CBC
        Cipher cipher = getEncryptingCipher(masterSecret.getEncryptionKey());
        byte[] encrypted = cipher.doFinal(body);
        byte[] iv = cipher.getIV();  // 16 bytes
        
        // 2. Combine IV + encrypted
        byte[] ivAndBody = new byte[iv.length + encrypted.length];
        System.arraycopy(iv, 0, ivAndBody, 0, iv.length);
        System.arraycopy(encrypted, 0, ivAndBody, iv.length, encrypted.length);
        
        // 3. Calculate HMAC
        Mac mac = getMac(masterSecret.getMacKey());
        byte[] mac = mac.doFinal(ivAndBody);
        
        // 4. Return: IV + encrypted + MAC
        byte[] result = new byte[ivAndBody.length + mac.length];
        System.arraycopy(ivAndBody, 0, result, 0, ivAndBody.length);
        System.arraycopy(mac, 0, result, ivAndBody.length, mac.length);
        
        return result;
    }

    public byte[] decryptBytes(byte[] decodedBody) throws InvalidMessageException {
        // 1. Extract and verify MAC
        Mac mac = getMac(masterSecret.getMacKey());
        byte[] encryptedBody = verifyMacBody(mac, decodedBody);
        
        // 2. Extract IV (first 16 bytes)
        byte[] iv = Arrays.copyOf(encryptedBody, 16);
        byte[] encrypted = Arrays.copyOfRange(encryptedBody, 16, encryptedBody.length);
        
        // 3. Decrypt with AES-CBC
        Cipher cipher = getDecryptingCipher(masterSecret.getEncryptionKey(), iv);
        return cipher.doFinal(encrypted);
    }
}
```

### KeyCachingService (In-Memory Key Cache)

The `KeyCachingService` keeps the MasterSecret in memory while the app is running:

```java
public class KeyCachingService extends Service {
    private static MasterSecret masterSecret;  // Static in-memory cache

    public static synchronized MasterSecret getMasterSecret(Context context) {
        if (masterSecret == null && passphraseDisabled) {
            // Auto-unlock if no passphrase set
            return MasterSecretUtil.getMasterSecret(context, UNENCRYPTED_PASSPHRASE);
        }
        return masterSecret;
    }

    public void setMasterSecret(MasterSecret masterSecret) {
        synchronized (KeyCachingService.class) {
            KeyCachingService.masterSecret = masterSecret;
            foregroundService();       // Show lock notification
            broadcastNewSecret();      // Notify app components
            startTimeoutIfAppropriate(); // Auto-lock timeout
        }
    }

    private void handleClearKey() {
        KeyCachingService.masterSecret = null;  // Clear from memory
        stopForeground(true);
        sendBroadcast(new Intent(CLEAR_KEY_EVENT));
    }
}
```

---

## 2. MasterKey (SVR2 - Secure Value Recovery)

The `MasterKey` is a 32-byte key used to derive other cryptographic keys and is backed up to Signal's SVR2 servers using Intel SGX enclaves.

### MasterKey Structure

```kotlin
class MasterKey(masterKey: ByteArray) {
    companion object {
        private const val LENGTH = 32  // 256 bits

        fun createNew(secureRandom: SecureRandom): MasterKey {
            val key = ByteArray(LENGTH)
            secureRandom.nextBytes(key)
            return MasterKey(key)
        }
    }

    // Derive different keys for different purposes
    fun deriveRegistrationLock(): String {
        return Hex.toStringCondensed(derive("Registration Lock"))
    }

    fun deriveRegistrationRecoveryPassword(): String {
        return Base64.encodeWithPadding(derive("Registration Recovery")!!)
    }

    fun deriveStorageServiceKey(): StorageKey {
        return StorageKey(derive("Storage Service Encryption")!!)
    }

    fun deriveLoggingKey(): ByteArray? {
        return derive("Logging Key")
    }

    private fun derive(keyName: String): ByteArray? {
        return CryptoUtil.hmacSha256(masterKey, keyName.toByteArray(Charsets.UTF_8))
    }
}
```

### SVR2 Backup Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    SVR2 Backup (Set PIN)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User enters PIN                                             │
│                                                                 │
│  2. Normalize PIN                                               │
│     normalizedPin = normalize(userPin)  // UTF-8 bytes          │
│                                                                 │
│  3. Derive PIN hash with SGX enclave info                       │
│     pinHash = PinHash.svr2(                                     │
│         normalizedPin,                                          │
│         username,              // from auth credentials         │
│         mrEnclave              // SGX enclave identifier        │
│     )                                                           │
│                                                                 │
│  4. Encrypt MasterKey with PIN hash                             │
│     kbsData = PinHashUtil.createNewKbsData(pinHash, masterKey)  │
│     // Returns: kbsAccessKey, cipherText                        │
│                                                                 │
│  5. Send Backup Request to SVR2                                 │
│     POST /v2/svr/backup {                                       │
│         pin: kbsAccessKey,                                      │
│         data: cipherText,                                       │
│         maxTries: 10                                            │
│     }                                                           │
│                                                                 │
│  6. Expose the data (make it restorable)                        │
│     POST /v2/svr/expose { data: cipherText }                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### SVR2 Restore Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    SVR2 Restore (Recover PIN)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User enters PIN                                             │
│                                                                 │
│  2. Get auth credentials from server                            │
│     auth = GET /v2/svr/auth                                     │
│                                                                 │
│  3. Derive PIN access key                                       │
│     pinHash = PinHash.svr2(normalizedPin, username, mrEnclave)  │
│     accessKey = pinHash.accessKey()                             │
│                                                                 │
│  4. Send Restore Request to SVR2                                │
│     POST /v2/svr/restore { pin: accessKey }                     │
│                                                                 │
│  5. Server Response:                                            │
│     ├── OK: { data: encryptedData, tries: remaining }           │
│     ├── PIN_MISMATCH: { tries: remaining }  // Wrong PIN        │
│     └── MISSING: No data found                                  │
│                                                                 │
│  6. If OK, decrypt the MasterKey                                │
│     masterKey = PinHashUtil.decryptSvrDataIVCipherText(         │
│         pinHash,                                                │
│         encryptedData                                           │
│     ).masterKey                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### SecureValueRecoveryV2 Implementation

```kotlin
class SecureValueRecoveryV2(
    private val serviceConfiguration: SignalServiceConfiguration,
    private val mrEnclave: String,
    private val authWebSocket: SignalWebSocket.AuthenticatedWebSocket
) : SecureValueRecovery {

    // Set PIN and backup MasterKey
    override fun setPin(userPin: String, masterKey: MasterKey): PinChangeSession {
        return Svr2PinChangeSession(userPin, masterKey)
    }

    // Restore MasterKey with PIN
    private fun restoreData(fetchAuth: () -> AuthCredentials, userPin: String): RestoreResponse {
        val normalizedPin = PinHashUtil.normalize(userPin)
        val authorization = fetchAuth()

        // Create restore request with PIN-derived access key
        val pinHash = PinHash.svr2(normalizedPin, authorization.username(), Hex.fromStringCondensed(mrEnclave))
        
        val response = Svr2Socket(serviceConfiguration, mrEnclave).makeRequest(
            authorization = authorization,
            clientRequest = Request(
                restore = RestoreRequest(
                    pin = pinHash.accessKey().toByteString()
                )
            )
        )

        return when (response.restore?.status) {
            ProtoRestoreResponse.Status.OK -> {
                val ciphertext = response.restore.data_.toByteArray()
                val masterKey = PinHashUtil.decryptSvrDataIVCipherText(pinHash, ciphertext).masterKey
                RestoreResponse.Success(masterKey, authorization)
            }
            ProtoRestoreResponse.Status.PIN_MISMATCH -> {
                RestoreResponse.PinMismatch(response.restore.tries)
            }
            ProtoRestoreResponse.Status.MISSING -> {
                RestoreResponse.Missing
            }
            else -> RestoreResponse.ApplicationError(...)
        }
    }

    inner class Svr2PinChangeSession(
        val userPin: String,
        val masterKey: MasterKey,
        private var setupComplete: Boolean = false
    ) : PinChangeSession {

        override fun execute(): BackupResponse {
            val normalizedPin = PinHashUtil.normalize(userPin)
            val authorization = authorization()

            // Step 1: Backup (creates encrypted blob, resets guess count)
            val pinHash = PinHash.svr2(normalizedPin, authorization.username(), Hex.fromStringCondensed(mrEnclave))
            val data = PinHashUtil.createNewKbsData(pinHash, masterKey)

            val backupResponse = Svr2Socket(serviceConfiguration, mrEnclave).makeRequest(
                authorization,
                Request(backup = BackupRequest(
                    pin = data.kbsAccessKey.toByteString(),
                    data_ = data.cipherText.toByteString(),
                    maxTries = 10
                ))
            )

            // Step 2: Expose (make data restorable)
            val exposeResponse = Svr2Socket(serviceConfiguration, mrEnclave).makeRequest(
                authorization,
                Request(expose = ExposeRequest(data_ = data.cipherText.toByteString()))
            )

            return BackupResponse.Success(masterKey, authorization, SvrVersion.SVR2)
        }
    }
}
```

---

## 3. Porting to Your Project

### Minimum Implementation for MasterSecret (Local Encryption)

If you only need local encrypted storage:

```kotlin
// 1. Create MasterSecret on first launch
class KeyManager(context: Context) {
    private val prefs = context.getSharedPreferences("secrets", Context.MODE_PRIVATE)
    private var masterSecret: MasterSecret? = null

    fun initialize(passphrase: String): MasterSecret {
        val encryptionKey = generateAesKey(128)
        val macKey = generateHmacKey(160)
        
        // Encrypt and store
        val salt = generateSalt(16)
        val iterations = calculateIterations(passphrase, salt)
        val encrypted = encryptWithPbe(encryptionKey + macKey, passphrase, salt, iterations)
        
        prefs.edit()
            .putString("salt", Base64.encode(salt))
            .putInt("iterations", iterations)
            .putString("encrypted", Base64.encode(encrypted))
            .apply()
            
        masterSecret = MasterSecret(encryptionKey, macKey)
        return masterSecret!!
    }

    fun unlock(passphrase: String): MasterSecret {
        val salt = Base64.decode(prefs.getString("salt", "")!!)
        val iterations = prefs.getInt("iterations", 100)
        val encrypted = Base64.decode(prefs.getString("encrypted", "")!!)
        
        val decrypted = decryptWithPbe(encrypted, passphrase, salt, iterations)
        val encryptionKey = decrypted.sliceArray(0 until 16)
        val macKey = decrypted.sliceArray(16 until 36)
        
        masterSecret = MasterSecret(encryptionKey, macKey)
        return masterSecret!!
    }

    fun encrypt(data: ByteArray): ByteArray {
        val secret = masterSecret ?: throw IllegalStateException("Not unlocked")
        return AesCbc.encrypt(data, secret.encryptionKey, secret.macKey)
    }

    fun decrypt(data: ByteArray): ByteArray {
        val secret = masterSecret ?: throw IllegalStateException("Not unlocked")
        return AesCbc.decrypt(data, secret.encryptionKey, secret.macKey)
    }
}
```

### Minimum Implementation for MasterKey (Cloud Backup)

For secure backup to your own servers:

```kotlin
// 1. MasterKey class
class MasterKey(private val key: ByteArray) {
    init { require(key.size == 32) { "MasterKey must be 32 bytes" } }
    
    fun derive(purpose: String): ByteArray {
        return HmacSha256(key, purpose.toByteArray())
    }
    
    fun serialize(): ByteArray = key.clone()
    
    companion object {
        fun createNew(): MasterKey {
            val key = ByteArray(32)
            SecureRandom().nextBytes(key)
            return MasterKey(key)
        }
    }
}

// 2. Backup to server
class SecureBackupClient(
    private val apiClient: ApiClient
) {
    suspend fun backup(userPin: String, masterKey: MasterKey): Result<Unit> {
        // Derive key from PIN
        val pinHash = Argon2.hash(userPin, salt = getRandomBytes(16))
        
        // Encrypt master key
        val iv = getRandomBytes(16)
        val encryptedKey = AesGcm.encrypt(masterKey.serialize(), pinHash, iv)
        
        // Upload to server
        return apiClient.uploadBackup(encryptedKey, iv)
    }
    
    suspend fun restore(userPin: String): Result<MasterKey> {
        // Download encrypted backup
        val (encryptedKey, iv) = apiClient.downloadBackup()
        
        // Derive key from PIN
        val pinHash = Argon2.hash(userPin, salt = iv)
        
        // Decrypt master key
        val decrypted = AesGcm.decrypt(encryptedKey, pinHash, iv)
        return Result.success(MasterKey(decrypted))
    }
}
```

---

## Key Differences Summary

| Aspect | MasterSecret | MasterKey |
|--------|--------------|-----------|
| **Size** | 128-bit AES + 160-bit MAC | 256-bit single key |
| **Purpose** | Local database encryption | Deriving other keys + cloud backup |
| **Storage** | SharedPreferences (encrypted) | SVR2 server (encrypted) |
| **Protection** | User passphrase | User PIN + Intel SGX enclave |
| **Recovery** | Local only | Cloud-based with PIN |

## Related Documentation

- [Security & Cryptography](Security-Cryptography.md) - Overall security architecture
- [Database](Database.md) - SQLCipher database encryption
- [Business Domain - Identity Context](Business-Domain.md#identity-context) - Key management domain concepts