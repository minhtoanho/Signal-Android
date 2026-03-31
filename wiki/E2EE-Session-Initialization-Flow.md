# E2EE Session Initialization Flow

> **This document explains how Signal establishes end-to-end encrypted sessions using the Signal Protocol (X3DH + Double Ratchet), including prekey exchange, session creation, and message encryption/decryption.**

## Overview

Signal uses the Signal Protocol for end-to-end encryption, which combines:
- **X3DH (Extended Triple Diffie-Hellman)**: Key agreement protocol for initial session establishment
- **Double Ratchet Algorithm**: Provides forward secrecy and break-in recovery
- **PreKey Bundles**: Server-stored keys for asynchronous session initiation
- **Sealed Sender**: Sender authentication without revealing identity to server

### Key Concepts

| Concept | Purpose | Protocol Layer |
|---------|---------|----------------|
| **PreKey Bundle** | Async session initiation | X3DH |
| **Session Record** | Stores ratchet state | Double Ratchet |
| **Signed PreKey** | Medium-term key signed by identity | X3DH |
| **One-Time PreKey** | Single-use key for forward secrecy | X3DH |
| **Kyber PreKey** | Post-quantum KEM key | PQXDH |
| **Sealed Sender** | Hide sender identity from server | Envelope |

---

## 1. Code Architecture

### Component Locations

| Component | File | Key Lines |
|-----------|------|-----------|
| **Message Sender** | `lib/.../SignalServiceMessageSender.java` | 166-3114 |
| **Message Decryptor** | `app/.../MessageDecryptor.kt` | 86-660 |
| **Session Cipher** | `lib/.../SignalServiceCipher.java` | 69-284 |
| **Sealed Session Cipher** | `lib/.../SignalSealedSessionCipher.java` | 34-83 |
| **Session Builder** | `lib/.../SignalSessionBuilder.java` | 12-27 |
| **Session Store** | `app/.../TextSecureSessionStore.java` | 24-203 |
| **Session Table** | `app/.../SessionTable.kt` | 22-240 |
| **Keys API** | `lib/.../KeysApi.kt` | 39-260 |

### Data Flow Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    E2EE SESSION LAYER ARCHITECTURE                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │  Application    │    │    Protocol     │    │    Storage      │ │
│  │    Layer        │    │     Layer       │    │     Layer       │ │
│  ├─────────────────┤    ├─────────────────┤    ├─────────────────┤ │
│  │ MessageSender   │───▶│ SessionBuilder  │───▶│ SessionStore    │ │
│  │ :2831-2870      │    │ :12-27          │    │ :24-203         │ │
│  │                 │    │                 │    │                 │ │
│  │ MessageDecryptor│───▶│ SessionCipher   │───▶│ SessionTable    │ │
│  │ :86-660         │    │ :69-284         │    │ :22-240         │ │
│  │                 │    │                 │    │                 │ │
│  │ KeysApi         │───▶│ PreKeyBundle    │───▶│ IdentityStore   │ │
│  │ :39-260         │    │                 │    │                 │ │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘ │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    CRYPTOGRAPHIC LAYER (libsignal)              ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         ││
│  │  │   X3DH       │  │ DoubleRatchet│  │   AES-CTR    │         ││
│  │  │  Protocol    │  │   Protocol   │  │  + HMAC      │         ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘         ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Session Initialization on Send

### Flow 2.1: Complete Send Flow with Session Check

```
┌────────────────────────────────────────────────────────────────────┐
│            MESSAGE SEND WITH SESSION INITIALIZATION                 │
│            SignalServiceMessageSender.getEncryptedMessage():2831-2870│
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ENTRY: send() or sendSyncMessage()                               │
│                                                                    │
│  STEP 1: Check for Existing Session                                │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ val address = SignalProtocolAddress(recipient, deviceId)   │   │
│  │ if (!aciStore.containsSession(address)) {                   │   │
│  │   // Need to establish session via PreKey bundle           │   │
│  │ }                                                           │   │
│  │ Reference: SignalServiceMessageSender:2841                 │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STEP 2: Fetch PreKey Bundle (if no session)                       │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ getPreKeys(recipient, sealedSenderAccess, deviceId):2947   │   │
│  │   └── keysApi.getPreKeys():148-154                         │   │
│  │       └── GET /v2/keys/{identifier}/{deviceSpecifier}:189  │   │
│  │                                                            │   │
│  │ Returns: List<PreKeyBundle> containing:                    │   │
│  │   - registrationId: Int                                    │   │
│  │   - deviceId: Int                                          │   │
│  │   - preKey: ECPublicKey (one-time)                         │   │
│  │   - signedPreKey: ECPublicKey + signature                  │   │
│  │   - identityKey: IdentityKey                               │   │
│  │   - kyberPreKey: KEMPublicKey + signature                  │   │
│  │                                                            │   │
│  │ Reference: KeysApi:188-259, PreKeyBundle construction:240-254 │ │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STEP 3: Process PreKey Bundle (X3DH Key Agreement)                │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ for (preKey in preKeys) {                                  │   │
│  │   val sessionBuilder = SignalSessionBuilder(               │   │
│  │     sessionLock,                                           │   │
│  │     SessionBuilder(aciStore, preKeyAddress)                │   │
│  │   )                                                        │   │
│  │   sessionBuilder.process(preKey)  // X3DH happens here     │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ Reference: SignalServiceMessageSender:2845-2855            │   │
│  │            SignalSessionBuilder.process():22-26            │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STEP 4: Encrypt Message                                           │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ val cipher = SignalServiceCipher(                          │   │
│  │   localAddress, localDeviceId, aciStore, sessionLock       │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ if (sealedSenderAccess != null) {                          │   │
│  │   // Sealed Sender (unidentified)                          │   │
│  │   cipher.encrypt(address, sealedSenderAccess, plaintext)   │   │
│  │ } else {                                                   │   │
│  │   // Normal (identified)                                   │   │
│  │   cipher.encrypt(address, null, plaintext)                 │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ Reference: SignalServiceMessageSender:2839, 2866           │   │
│  │            SignalServiceCipher.encrypt():115-133           │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  OUTPUT: OutgoingPushMessage (encrypted envelope)                 │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 2.2: PreKey Bundle Fetching

```
┌────────────────────────────────────────────────────────────────────┐
│                PREKEY BUNDLE FETCH                                  │
│                KeysApi.getPreKeys():148-259                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  NETWORK REQUEST:                                                  │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ GET /v2/keys/{destination}/{deviceSpecifier}               │   │
│  │                                                            │   │
│  │ Headers:                                                   │   │
│  │   - Authorization: Bearer {token} (if not sealed sender)   │   │
│  │   - Unidentified-Access-Key: {key} (if sealed sender)      │   │
│  │                                                            │   │
│  │ deviceSpecifier:                                           │   │
│  │   - "1" = Primary device only                              │   │
│  │   - "*" = All devices (when fetching for primary)          │   │
│  │   - "{N}" = Specific device ID                             │   │
│  │                                                            │   │
│  │ Reference: KeysApi:188-189                                 │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  RESPONSE PROCESSING:                                              │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ PreKeyResponse {                                           │   │
│  │   identityKey: IdentityKey                                 │   │
│  │   devices: List<PreKeyResponseItem>                        │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ For each device:                                           │   │
│  │ ├── Validate signedPreKey exists:217-224                   │   │
│  │ ├── Validate kyberPreKey exists:231-238                    │   │
│  │ └── Build PreKeyBundle:240-254                             │   │
│  │                                                            │   │
│  │ PreKeyBundle construction:                                 │   │
│  │   PreKeyBundle(                                            │   │
│  │     registrationId,                                        │   │
│  │     deviceId,                                              │   │
│  │     preKeyId, preKey,          // One-time EC prekey       │   │
│  │     signedPreKeyId, signedPreKey, signedPreKeySignature,   │   │
│  │     identityKey,               // Long-term identity       │   │
│  │     kyberPreKeyId, kyberPreKey, kyberPreKeySignature       │   │
│  │   )                                                        │   │
│  │                                                            │   │
│  │ Reference: KeysApi:240-254                                 │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ERROR HANDLING:                                                   │
│  ├── 401: Authentication failed → retry or fail                  │
│  ├── 404: User not found → UnregisteredUserException:200-202     │
│  └── 429: Rate limited → retry with backoff                       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 2.3: X3DH Session Establishment

```
┌────────────────────────────────────────────────────────────────────┐
│                X3DH KEY AGREEMENT                                   │
│                SignalSessionBuilder.process():22-26                │
│                (wraps libsignal SessionBuilder.process())          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  X3DH PROTOCOL (libsignal native):                                │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                                                            │   │
│  │  ALICE (sender)                 BOB (recipient)            │   │
│  │  ============                 ===============              │   │
│  │                                                            │   │
│  │  Has: identityKeyPair            Published:               │   │
│  │       signedPreKeyPair             - identityKey           │   │
│  │       (one-time) preKey?           - signedPreKey          │   │
│  │                                    - one-time preKey?      │   │
│  │                                    - kyberPreKey           │   │
│  │                                                            │   │
│  │  ─────────────────────────────────────────────────────    │   │
│  │                                                            │   │
│  │  DH1 = DH(IKa, SPKb)     // Identity ↔ Signed PreKey      │   │
│  │  DH2 = DH(EKa, IKb)      // Ephemeral ↔ Identity          │   │
│  │  DH3 = DH(EKa, SPKb)     // Ephemeral ↔ Signed PreKey     │   │
│  │  DH4 = DH(EKa, OPKb)     // Ephemeral ↔ One-Time PreKey   │   │
│  │          (if available)                                    │   │
│  │                                                            │   │
│  │  SK = KDF(DH1 || DH2 || DH3 || DH4)                       │   │
│  │                                                            │   │
│  │  AD = Info(IKa) || Info(IKb)                              │   │
│  │     // Associated Data for authentication                 │   │
│  │                                                            │   │
│  │  ─────────────────────────────────────────────────────    │   │
│  │                                                            │   │
│  │  SESSION INITIALIZATION:                                   │   │
│  │  - Root key = SK                                          │   │
│  │  - Chain keys derived from root key                       │   │
│  │  - Double ratchet starts with first message               │   │
│  │                                                            │   │
│  │  STORE SESSION:                                            │   │
│  │  sessionStore.storeSession(address, sessionRecord)        │   │
│  │  └── TextSecureSessionStore.storeSession():68-72          │   │
│  │      └── SignalDatabase.sessions().store():44-56          │   │
│  │                                                            │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  PQXDH EXTENSION (Post-Quantum):                                  │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ Additional step:                                           │   │
│  │   DH_PQ = Kyber.Encapsulate(kyberPreKey)                  │   │
│  │   SK = KDF(DH1 || DH2 || DH3 || DH4 || DH_PQ)             │   │
│  │                                                            │   │
│  │ This provides post-quantum confidentiality.               │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  THREAD SAFETY:                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ try (SignalSessionLock.Lock unused = lock.acquire()) {    │   │
│  │   builder.process(preKey);                                 │   │
│  │ }                                                          │   │
│  │ Reference: SignalSessionBuilder:23-25                      │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. Message Encryption

### Flow 3.1: Envelope Encryption

```
┌────────────────────────────────────────────────────────────────────┐
│                MESSAGE ENCRYPTION                                   │
│                SignalServiceCipher.encrypt():115-133                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  STEP 1: Initialize Cipher                                         │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ val sessionCipher = SignalSessionCipher(                   │   │
│  │   sessionLock,                                             │   │
│  │   SessionCipher(signalProtocolStore, destination)          │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ Reference: SignalServiceCipher:121                         │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STEP 2a: Sealed Sender Encryption (Preferred)                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ if (sealedSenderAccess != null) {                          │   │
│  │   val sealedCipher = SignalSealedSessionCipher(            │   │
│  │     sessionLock,                                           │   │
│  │     SealedSessionCipher(store, localUuid, localE164, ...)  │   │
│  │   )                                                        │   │
│  │                                                            │   │
│  │   return content.processSealedSender(                      │   │
│  │     sessionCipher,                                         │   │
│  │     sealedCipher,                                          │   │
│  │     destination,                                           │   │
│  │     sealedSenderAccess.senderCertificate                   │   │
│  │   )                                                        │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ Reference: SignalServiceCipher:122-126                     │   │
│  │            SignalSealedSessionCipher.encrypt():44-50       │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STEP 2b: Unsealed Sender Encryption (Fallback)                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ else {                                                     │   │
│  │   return content.processUnsealedSender(                    │   │
│  │     sessionCipher,                                         │   │
│  │     destination                                            │   │
│  │   )                                                        │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ Reference: SignalServiceCipher:127-128                     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  DOUBLE RATCHET ENCRYPTION:                                        │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                                                            │   │
│  │  1. Get or create sending chain key                       │   │
│  │  2. Derive message key from chain key                     │   │
│  │  3. Encrypt with AES-CTR                                  │   │
│  │  4. Calculate HMAC over ciphertext                        │   │
│  │  5. Increment chain key for next message                  │   │
│  │                                                            │   │
│  │  Output format:                                            │   │
│  │  [Version][RatchetHeader][EncryptedContent][HMAC]         │   │
│  │                                                            │   │
│  │  RatchetHeader = {                                         │   │
│  │    senderEphemeral: ECPublicKey,                          │   │
│  │    previousChainLength: Int,                               │   │
│  │    messageNumber: Int                                      │   │
│  │  }                                                         │   │
│  │                                                            │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  OUTPUT: OutgoingPushMessage                                       │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ {                                                          │   │
│  │   type: PREKEY_BUNDLE | CIPHERTEXT,                        │   │
│  │   destinationDeviceId: Int,                                │   │
│  │   destinationRegistrationId: Int,                          │   │
│  │   content: Base64(ciphertext),                             │   │
│  │   timestamp: Long                                          │   │
│  │ }                                                          │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 3.2: Sealed Sender Encryption

```
┌────────────────────────────────────────────────────────────────────┐
│                SEALED SENDER ENCRYPTION                             │
│                SignalSealedSessionCipher.encrypt():44-50            │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  PURPOSE: Hide sender identity from server                        │
│                                                                    │
│  ENVELOPE STRUCTURE:                                               │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                                                            │   │
│  │  UnidentifiedSenderMessage {                               │   │
│  │    ephemeralPublicKey: ECPublicKey,                       │   │
│  │    encryptedStatic: ByteArray,      // Encrypted sender ID │   │
│  │    encryptedMessage: ByteArray      // Encrypted content   │   │
│  │  }                                                         │   │
│  │                                                            │   │
│  │  UnidentifiedSenderMessageContent {                        │   │
│  │    type: CIPHERTEXT | PREKEY_TYPE,                        │   │
│  │    senderCertificate: SenderCertificate,                  │   │
│  │    content: ByteArray,            // Inner ciphertext     │   │
│  │    contentHint: Int,              // For retry receipts   │   │
│  │    groupId: ByteArray?            // Optional group ID    │   │
│  │  }                                                         │   │
│  │                                                            │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ENCRYPTION STEPS:                                                 │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ 1. Create inner ciphertext with SessionCipher             │   │
│  │ 2. Wrap in UnidentifiedSenderMessageContent               │   │
│  │ 3. Encrypt with sealed session cipher                     │   │
│  │    - Uses sender certificate for authentication           │   │
│  │    - Server cannot read sender identity                   │   │
│  │                                                            │   │
│  │ Reference: SealedSessionCipher.encrypt() (libsignal)      │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  SENDER CERTIFICATE:                                               │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ SenderCertificate {                                        │   │
│  │   sender: String,              // UUID or E164            │   │
│  │   senderDevice: Int,                                      │   │
│  │   expires: Long,              // Certificate expiry       │   │
│  │   identityKey: IdentityKey,     // Sender's identity      │   │
│  │   signer: ServerCertificate     // Signed by server       │   │
│  │ }                                                         │   │
│  │                                                            │   │
│  │ Obtained from: SealedSenderAccess.senderCertificate       │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 4. Session Initialization on Receive

### Flow 4.1: Message Decryption

```
┌────────────────────────────────────────────────────────────────────┐
│                MESSAGE DECRYPTION                                   │
│                MessageDecryptor.decrypt():98-299                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ENTRY: IncomingMessageObserver.onMessage():314                    │
│                                                                    │
│  STEP 1: Validate Envelope                                         │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ val destination = ServiceId.parseOrNull(                   │   │
│  │   envelope.destinationServiceId,                           │   │
│  │   envelope.destinationServiceIdBinary                      │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ // Must match our ACI or PNI                               │   │
│  │ if (destination != selfAci && destination != selfPni) {   │   │
│  │   return Result.Ignore(...)                                │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ Reference: MessageDecryptor:107-117                        │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STEP 2: Initialize Cipher                                         │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ val cipher = SignalServiceCipher(                          │   │
│  │   localAddress,                                            │   │
│  │   SignalStore.account.deviceId,                            │   │
│  │   bufferedProtocolStore.get(destination),                  │   │
│  │   ReentrantSessionLock.INSTANCE,                           │   │
│  │   SealedSenderAccessUtil.getCertificateValidator()         │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ Reference: MessageDecryptor:152-154                        │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STEP 3: Decrypt Envelope                                          │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ val cipherResult = cipher.decrypt(                         │   │
│  │   envelope,                                                │   │
│  │   serverDeliveredTimestamp                                 │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ Reference: MessageDecryptor:159, SignalServiceCipher:135-166│   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STEP 4: Handle PreKey Bundle Message                              │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ if (envelope.type == Envelope.Type.PREKEY_BUNDLE) {        │   │
│  │   // First message in session                              │   │
│  │   sessionCipher.decrypt(PreKeySignalMessage(content))      │   │
│  │                                                            │   │
│  │   // Schedule PreKey sync (our PreKey was consumed)        │   │
│  │   followUpOperations += PreKeysSyncJob.create().asChain() │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ Reference: SignalServiceCipher:190-198                     │   │
│  │            MessageDecryptor:145-150                        │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STEP 5: Handle Sender Key Distribution                            │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ if (cipherResult.content.senderKeyDistributionMessage != null) {│
│  │   handleSenderKeyDistributionMessage(                      │   │
│  │     sourceServiceId,                                       │   │
│  │     sourceDeviceId,                                        │   │
│  │     SenderKeyDistributionMessage(content)                  │   │
│  │   )                                                        │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ Reference: MessageDecryptor:203-211                        │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ERROR HANDLING:                                                   │
│  ├── ProtocolNoSessionException → Build retry receipt            │
│  ├── ProtocolInvalidKeyException → Reset session                 │
│  ├── ProtocolUntrustedIdentityException → Safety number change   │
│  └── ProtocolDuplicateMessageException → Ignore                  │
│                                                                    │
│  OUTPUT: Result.Success(envelope, content, metadata)              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 4.2: SignalServiceCipher.decrypt() Internal

```
┌────────────────────────────────────────────────────────────────────┐
│                SIGNAL SERVICE CIPHER DECRYPT                        │
│                SignalServiceCipher.decryptInternal():168-260        │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ENVELOPE TYPE HANDLING:                                           │
│                                                                    │
│  CASE 1: PREKEY_BUNDLE (First message)                             │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ // X3DH initialization message                             │   │
│  │                                                            │   │
│  │ val sourceAddress = SignalProtocolAddress(                │   │
│  │   sourceServiceId.toString(),                              │   │
│  │   envelope.sourceDevice                                    │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ val sessionCipher = SignalSessionCipher(                  │   │
│  │   sessionLock,                                             │   │
│  │   SessionCipher(signalProtocolStore, sourceAddress)        │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ paddedMessage = sessionCipher.decrypt(                     │   │
│  │   PreKeySignalMessage(envelope.content)                    │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ // Session is now established!                             │   │
│  │ signalProtocolStore.clearSenderKeySharedWith(sourceAddress)│   │
│  │                                                            │   │
│  │ Reference: SignalServiceCipher:190-198                     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  CASE 2: CIPHERTEXT (Normal message)                               │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ val sourceAddress = SignalProtocolAddress(                │   │
│  │   sourceServiceId.toString(),                              │   │
│  │   envelope.sourceDevice                                    │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ val sessionCipher = SignalSessionCipher(                  │   │
│  │   sessionLock,                                             │   │
│  │   SessionCipher(signalProtocolStore, sourceAddress)        │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ paddedMessage = sessionCipher.decrypt(                     │   │
│  │   SignalMessage(envelope.content)                          │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ Reference: SignalServiceCipher:199-207                     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  CASE 3: UNIDENTIFIED_SENDER (Sealed Sender)                       │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ val sealedCipher = SignalSealedSessionCipher(              │   │
│  │   sessionLock,                                             │   │
│  │   SealedSessionCipher(                                     │   │
│  │     signalProtocolStore,                                   │   │
│  │     localUuid,                                             │   │
│  │     localE164,                                             │   │
│  │     localDeviceId                                          │   │
│  │   )                                                        │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ val decryptionResult = sealedCipher.decrypt(               │   │
│  │   certificateValidator,                                    │   │
│  │   envelope.content,                                        │   │
│  │   serverDeliveredTimestamp                                 │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ // Decrypt reveals:                                        │   │
│  │ // - Sender ACI/E164                                       │   │
│  │ // - Sender device ID                                      │   │
│  │ // - Inner message type (PREKEY or CIPHERTEXT)             │   │
│  │ // - Decrypted content                                     │   │
│  │                                                            │   │
│  │ Reference: SignalServiceCipher:210-239                     │   │
│  │            SignalSealedSessionCipher.decrypt():66-70       │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  DOUBLE RATCHET DECRYPTION:                                        │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ 1. Parse ratchet header                                   │   │
│  │ 2. Perform DH ratchet step if needed                      │   │
│  │ 3. Derive message key from receiving chain                │   │
│  │ 4. Verify HMAC                                            │   │
│  │ 5. Decrypt with AES-CTR                                   │   │
│  │ 6. Remove padding                                         │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 5. Session Storage

### Flow 5.1: Session Store Operations

```
┌────────────────────────────────────────────────────────────────────┐
│                SESSION STORE                                        │
│                TextSecureSessionStore:24-203                        │
│                SessionTable:22-240                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  DATABASE SCHEMA:                                                  │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ CREATE TABLE sessions (                                    │   │
│  │   _id INTEGER PRIMARY KEY AUTOINCREMENT,                   │   │
│  │   account_id TEXT NOT NULL,      -- ACI or PNI            │   │
│  │   address TEXT NOT NULL,         -- Recipient service ID  │   │
│  │   device INTEGER NOT NULL,       -- Device ID             │   │
│  │   record BLOB NOT NULL,          -- Serialized SessionRecord│   │
│  │   UNIQUE(account_id, address, device)                      │   │
│  │ )                                                          │   │
│  │                                                            │   │
│  │ Reference: SessionTable:32-41                              │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  LOAD SESSION:                                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ fun loadSession(address: SignalProtocolAddress): SessionRecord {│
│  │   try (lock.acquire()) {                                   │   │
│  │     val record = SignalDatabase.sessions().load(           │   │
│  │       accountId, address                                   │   │
│  │     )                                                      │   │
│  │     return record ?: SessionRecord()  // Empty if new     │   │
│  │   }                                                        │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ Reference: TextSecureSessionStore:35-46                    │   │
│  │            SessionTable.load():58-76                       │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  STORE SESSION:                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ fun storeSession(address: SignalProtocolAddress,           │   │
│  │                   record: SessionRecord) {                 │   │
│  │   try (lock.acquire()) {                                   │   │
│  │     SignalDatabase.sessions().store(                       │   │
│  │       accountId, address, record                           │   │
│  │     )                                                      │   │
│  │   }                                                        │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ Reference: TextSecureSessionStore:68-72                    │   │
│  │            SessionTable.store():44-56                      │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  CHECK SESSION EXISTS:                                             │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ fun containsSession(address: SignalProtocolAddress): Boolean {│
│  │   try (lock.acquire()) {                                   │   │
│  │     val record = SignalDatabase.sessions().load(accountId, address)│
│  │     return record != null && record.hasSenderChain()       │   │
│  │   }                                                        │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ // hasSenderChain() returns true if we have a sending chain│   │
│  │ // This means X3DH completed and ratchet is initialized   │   │
│  │                                                            │   │
│  │ Reference: TextSecureSessionStore:75-81                    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ARCHIVE SESSION:                                                  │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ fun archiveSession(address: SignalProtocolAddress) {       │   │
│  │   try (lock.acquire()) {                                   │   │
│  │     val session = SignalDatabase.sessions().load(...)      │   │
│  │     if (session != null) {                                 │   │
│  │       session.archiveCurrentState()  // Reset ratchet      │   │
│  │       SignalDatabase.sessions().store(...)                 │   │
│  │     }                                                      │   │
│  │   }                                                        │   │
│  │ }                                                          │   │
│  │                                                            │   │
│  │ // Used when:                                              │   │
│  │ // - Device is removed from account                        │   │
│  │ // - Session becomes stale                                 │   │
│  │ // - User resets secure session                            │   │
│  │                                                            │   │
│  │ Reference: TextSecureSessionStore:118-126                  │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 6. Complete Flow Diagrams

### Flow 6.1: First Message Send (No Existing Session)

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIRST MESSAGE SEND (Session Initialization)               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Alice sends first message to Bob                                   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 1. MessageSender.send(message, recipient)                     │  │
│  │    └── SignalServiceMessageSender.send()                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 2. Check for existing session                                 │  │
│  │    containsSession(recipient) → false                         │  │
│  │    Reference: SignalServiceMessageSender:2841                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 3. Fetch PreKey bundle from server                            │  │
│  │    GET /v2/keys/{bob_uuid}/*                                  │  │
│  │    Returns: PreKeyBundle for all of Bob's devices             │  │
│  │    Reference: KeysApi:148-154                                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 4. For each device, establish session via X3DH               │  │
│  │    sessionBuilder.process(preKeyBundle)                       │  │
│  │    - Derives shared secret using X3DH                         │  │
│  │    - Initializes double ratchet                               │  │
│  │    - Stores session record                                    │  │
│  │    Reference: SignalServiceMessageSender:2845-2855            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 5. Encrypt message for each device                            │  │
│  │    cipher.encrypt(recipient, content)                         │  │
│  │    - Use double ratchet to derive message key                 │  │
│  │    - AES-CTR encrypt + HMAC                                   │  │
│  │    - Advance ratchet                                          │  │
│  │    Reference: SignalServiceCipher:115-133                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 6. Send to server                                             │  │
│  │    POST /v1/messages/{recipient}                              │  │
│  │    Body: List<OutgoingPushMessage> (one per device)           │  │
│  │    Type: PREKEY_BUNDLE (indicates new session)                │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Flow 6.2: Subsequent Message Send (Existing Session)

```
┌─────────────────────────────────────────────────────────────────────┐
│           SUBSEQUENT MESSAGE SEND (Session Exists)                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Alice sends another message to Bob                                 │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 1. MessageSender.send(message, recipient)                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 2. Check for existing session                                 │  │
│  │    containsSession(recipient) → true                          │  │
│  │    Skip PreKey fetch                                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 3. Load existing session                                      │  │
│  │    sessionStore.loadSession(recipient)                        │  │
│  │    Contains: ratchet state, chain keys, root key              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 4. Encrypt with double ratchet                                │  │
│  │    - Derive message key from sending chain                    │  │
│  │    - Increment chain key                                      │  │
│  │    - AES-CTR encrypt + HMAC                                   │  │
│  │    Reference: SignalServiceCipher:115-133                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 5. Send to server                                             │  │
│  │    POST /v1/messages/{recipient}                              │  │
│  │    Type: CIPHERTEXT (normal message)                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Flow 6.3: Message Receive (First Message)

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIRST MESSAGE RECEIVE (Session Creation)                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Bob receives first message from Alice                              │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 1. WebSocket receives envelope                                │  │
│  │    Envelope {                                                  │  │
│  │      type: PREKEY_BUNDLE,                                     │  │
│  │      sourceServiceId: alice_uuid,                             │  │
│  │      sourceDevice: 1,                                         │  │
│  │      content: [encrypted PreKeySignalMessage]                 │  │
│  │    }                                                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 2. IncomingMessageObserver processes envelope                 │  │
│  │    MessageDecryptor.decrypt(envelope)                         │  │
│  │    Reference: IncomingMessageObserver:314                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 3. Detect PREKEY_BUNDLE type                                  │  │
│  │    Create SignalSessionCipher for source                      │  │
│  │    Reference: SignalServiceCipher:190-198                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 4. Decrypt PreKeySignalMessage                                │  │
│  │    sessionCipher.decrypt(preKeyMessage)                       │  │
│  │    - Alice's ephemeral key is extracted                       │  │
│  │    - X3DH calculation using:                                  │  │
│  │      - Bob's identity key pair                                │  │
│  │      - Bob's signed prekey pair                               │  │
│  │      - Bob's one-time prekey (if used)                        │  │
│  │      - Alice's ephemeral key                                  │  │
│  │    - Session is established!                                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 5. Store new session                                          │  │
│  │    sessionStore.storeSession(alice_address, session_record)   │  │
│  │    Session now exists for future messages                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 6. Schedule PreKey refresh                                    │  │
│  │    PreKeysSyncJob.create()                                    │  │
│  │    One-time prekey was consumed, need to upload replacement   │  │
│  │    Reference: MessageDecryptor:145-150                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 7. Process decrypted message                                  │  │
│  │    Content contains DataMessage, callMessage, etc.            │  │
│  │    Insert into database, show in conversation                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Error Handling and Recovery

### Flow 7.1: Session Errors

```
┌────────────────────────────────────────────────────────────────────┐
│                SESSION ERROR HANDLING                               │
│                MessageDecryptor.decrypt():238-299                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ERROR TYPES AND RECOVERY:                                         │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ ProtocolNoSessionException                                  │   │
│  │ ────────────────────────────                                │   │
│  │ Cause: No session found with sender                        │   │
│  │ Recovery:                                                   │   │
│  │   - Send retry receipt to request PreKey message           │   │
│  │   - Sender will re-establish session                        │   │
│  │   - buildResultForDecryptionError():252                    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ ProtocolInvalidKeyException / ProtocolInvalidKeyIdException│   │
│  │ ───────────────────────────────────────────────────────────│   │
│  │ Cause: Key not found or invalid                            │   │
│  │ Recovery:                                                   │   │
│  │   - Archive existing session                               │   │
│  │   - AutomaticSessionResetJob enqueued                      │   │
│  │   - New session will be established on next message        │   │
│  │   - Reference: MessageDecryptor:239-264                    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ ProtocolUntrustedIdentityException                         │   │
│  │ ──────────────────────────────────                         │   │
│  │ Cause: Sender's identity key changed                       │   │
│  │ Recovery:                                                   │   │
│  │   - Notify user of safety number change                    │   │
│  │   - Require user verification                              │   │
│  │   - Session blocked until user accepts new identity        │   │
│  │   - Reference: MessageDecryptor:241                        │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ ProtocolInvalidMessageException                            │   │
│  │ ───────────────────────────────                            │   │
│  │ Cause: Message decryption failed                           │   │
│  │ Recovery:                                                   │   │
│  │   - Request message resend via retry receipt               │   │
│  │   - May indicate ratchet desync                            │   │
│  │   - Reference: MessageDecryptor:243                        │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ ProtocolDuplicateMessageException                          │   │
│  │ ──────────────────────────────────                         │   │
│  │ Cause: Message already processed                           │   │
│  │ Recovery:                                                   │   │
│  │   - Ignore (already decrypted earlier)                     │   │
│  │   - Prevents replay attacks                                │   │
│  │   - Reference: MessageDecryptor:266-269                    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ ProtocolLegacyMessageException                             │   │
│  │ ──────────────────────────────                             │   │
│  │ Cause: Message from old protocol version                   │   │
│  │ Recovery:                                                   │   │
│  │   - Insert legacy message error placeholder                │   │
│  │   - User instructed to update app                          │   │
│  │   - Reference: MessageDecryptor:288-291                    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Flow 7.2: Mismatched Devices

```
┌────────────────────────────────────────────────────────────────────┐
│                MISMATCHED DEVICES HANDLING                          │
│                SignalServiceMessageSender.handleMismatchedDevices()│
│                :2965-2991                                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  SCENARIO: Recipient has new devices or removed devices            │
│                                                                    │
│  SERVER RESPONSE: MismatchedDevices {                             │
│    extraDevices: [2, 3],      // Devices we sent to but don't exist│
│    missingDevices: [4]        // Devices that exist but we didn't │
│  }                                                                 │
│                                                                    │
│  HANDLING:                                                         │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ 1. Archive sessions for extra devices                       │   │
│  │    for (deviceId in extraDevices) {                         │   │
│  │      aciStore.archiveSession(recipient, deviceId)           │   │
│  │    }                                                        │   │
│  │    Reference: SignalServiceMessageSender:2971               │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ 2. Fetch PreKeys for missing devices                        │   │
│  │    for (deviceId in missingDevices) {                       │   │
│  │      preKey = keysApi.getPreKey(recipient, deviceId)        │   │
│  │      sessionBuilder.process(preKey)                         │   │
│  │    }                                                        │   │
│  │    Reference: SignalServiceMessageSender:2978-2987          │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ 3. Clear sender key shared status                           │   │
│  │    clearSenderKeySharedWith(recipient, mismatchedDeviceIds) │   │
│  │    // Will need to re-share sender keys for group messages  │   │
│  │    Reference: SignalServiceMessageSender:2976               │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ 4. Retry message send                                       │   │
│  │    // Automatically retried by caller                       │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 8. Key Differences: PreKey vs Normal Messages

| Aspect | PreKey Bundle Message | Normal Ciphertext Message |
|--------|----------------------|---------------------------|
| **When Used** | First message in session | Subsequent messages |
| **Envelope Type** | `PREKEY_BUNDLE` | `CIPHERTEXT` |
| **Session State** | Creates new session | Uses existing session |
| **Key Material** | Contains ephemeral key + prekey reference | Contains ratchet header only |
| **Size** | Larger (includes X3DH data) | Smaller |
| **PreKey Consumption** | Consumes one-time prekey | None |
| **Server PreKey Sync** | Required after receive | Not needed |

## Related Documentation

- [Security & Cryptography](Security-Cryptography.md) - Overall security architecture
- [Master Key Flow](Master-Key-Flow.md) - Key management for encryption
- [Message Flow](Message-Flow.md) - Message sending and receiving
- [Business Domain - Messaging Context](Business-Domain.md#messaging-context) - Domain concepts