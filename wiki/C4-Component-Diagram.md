# C4 Component Diagram

> **Level 3**: Shows the major components within the Signal Android application container and how they interact.

## App Module Components

```mermaid
C4Component
    title Component Diagram - Signal Android App Module

    Container_Boundary(app, "Signal Android App") {
        Component(conversationUI, "Conversation UI", "ConversationActivity, ConversationFragment", "Chat screen interface")
        Component(conversationListUI, "Conversation List UI", "ConversationListActivity, ConversationListFragment", "Message list screen")
        Component(registrationUI, "Registration UI", "RegistrationActivity, RegistrationViewModel", "Account registration flow")
        Component(callUI, "Call UI", "WebRtcCallActivity", "Voice/video call interface")
        Component(paymentsUI, "Payments UI", "PaymentsActivity, Wallet", "Payment management")
        
        Component(messageViewModel, "Message ViewModel", "ConversationViewModel", "Chat state management")
        Component(groupManager, "Group Manager", "GroupManagerV2", "Group creation and management")
        Component(recipientManager, "Recipient Manager", "RecipientRepository", "Contact/recipient handling")
        
        Component(messageDB, "Message DB", "MessageTable, ThreadTable", "Message persistence")
        Component(groupDB, "Group DB", "GroupTable", "Group data persistence")
        Component(recipientDB, "Recipient DB", "RecipientTable", "Recipient data persistence")
        Component(callDB, "Call DB", "CallTable, CallLinkTable", "Call history persistence")
        Component(paymentDB, "Payment DB", "PaymentTable", "Transaction persistence")
        
        Component(messageSender, "Message Sender", "MessageSender", "Outgoing message handling")
        Component(messageReceiver, "Message Receiver", "IncomingMessageProcessor", "Incoming message processing")
        Component(jobManager, "Job Manager", "JobManager", "Background job scheduling")
        
        Component(crypto, "Crypto", "SignalProtocol, SessionCipher", "Encryption/decryption")
        Component(signalService, "Signal Service", "SignalServiceMessageSender", "Network communication")
        Component(pushReceiver, "Push Receiver", "FirebaseReceiver, SignalServiceReceiver", "Push notification handling")
    }

    Rel(conversationUI, messageViewModel, "Uses")
    Rel(conversationListUI, messageViewModel, "Uses")
    Rel(messageViewModel, messageDB, "Reads/Writes")
    Rel(messageViewModel, recipientDB, "Reads")
    Rel(messageViewModel, groupManager, "Uses")
    Rel(messageViewModel, recipientManager, "Uses")
    
    Rel(groupManager, groupDB, "Reads/Writes")
    Rel(groupManager, signalService, "Syncs groups")
    Rel(recipientManager, recipientDB, "Reads/Writes")
    
    Rel(messageSender, crypto, "Encrypts")
    Rel(messageSender, signalService, "Sends")
    Rel(messageSender, messageDB, "Stores")
    Rel(messageSender, jobManager, "Queues jobs")
    
    Rel(pushReceiver, jobManager, "Triggers")
    Rel(messageReceiver, crypto, "Decrypts")
    Rel(messageReceiver, messageDB, "Stores")
    Rel(messageReceiver, signalService, "Fetches")
    
    Rel(callUI, callDB, "Reads/Writes")
    Rel(paymentsUI, paymentDB, "Reads/Writes")
    
    Rel(registrationUI, signalService, "Registers")
    Rel(registrationUI, crypto, "Generates keys")
    
    UpdateLayoutConfig($c4ShapeInRow="4", $c4BoundaryInRow="1")
```

## Major Components by Domain

### Messaging Domain

| Component | Package | Responsibility |
|-----------|---------|----------------|
| **ConversationActivity** | `conversation.v2` | Main chat screen UI |
| **ConversationViewModel** | `conversation.v2` | Chat state and business logic |
| **ConversationItem** | `conversation` | Individual message rendering |
| **MessageSender** | `messages` | Outgoing message pipeline |
| **IncomingMessageProcessor** | `messages` | Incoming message handling |
| **MessageTable** | `database` | Message persistence |
| **ThreadTable** | `database` | Conversation thread persistence |

### Groups Domain

| Component | Package | Responsibility |
|-----------|---------|----------------|
| **GroupManagerV2** | `groups` | Group V2 creation and management |
| **LiveGroup** | `groups` | Real-time group state observer |
| **GroupTable** | `database` | Group persistence |
| **GroupCall** | `calls.links` | Group call management |

### Recipients/Contacts Domain

| Component | Package | Responsibility |
|-----------|---------|----------------|
| **RecipientRepository** | `recipients` | Recipient data access |
| **RecipientTable** | `database` | Recipient persistence |
| **ContactSelection** | `contacts` | Contact selection UI |
| **ContactSync** | `contacts` | System contact sync |

### Calls Domain

| Component | Package | Responsibility |
|-----------|---------|----------------|
| **WebRtcCallActivity** | `components.webrtc.v2` | Call screen UI |
| **CallManager** | `ringrtc` | Call state management |
| **CallTable** | `database` | Call history persistence |
| **CallLinkTable** | `database` | Call link persistence |

### Payments Domain

| Component | Package | Responsibility |
|-----------|---------|----------------|
| **Wallet** | `payments` | Wallet management |
| **PaymentTable** | `database` | Transaction persistence |
| **MobileCoinIntegration** | `payments` | MobileCoin SDK integration |

### Registration Domain

| Component | Package | Responsibility |
|-----------|---------|----------------|
| **RegistrationActivity** | `registration` | Registration UI (in feature module) |
| **RegistrationViewModel** | `registration` | Registration state |
| **SignalServiceAccountManager** | `service` | Account management API |

### Crypto Domain

| Component | Package | Lines | Responsibility |
|-----------|---------|-------|----------------|
| **SignalProtocol** | `libsignal` | N/A | Core Signal Protocol implementation |
| **SessionCipher** | `libsignal` | N/A | Message encryption/decryption |
| **IdentityKeyUtil** | `crypto` | varies | Identity key management |
| **PreKeyUtil** | `crypto` | varies | PreKey generation and management |
| **SenderKeyUtil** | `app/.../crypto/SenderKeyUtil.java` | 18-50 | Group encryption keys, rotation |

### Key Methods with Code References

| Method | File | Lines | Purpose |
|--------|------|-------|---------|
| `getEncryptedMessage()` | `SignalServiceMessageSender.java` | 2831-2870 | Check session, fetch prekeys, encrypt |
| `sendGroupMessage()` | `SignalServiceMessageSender.java` | 2480-2613 | Sender key distribution, group encryption |
| `getOrCreateNewGroupSession()` | `SignalServiceMessageSender.java` | 503-506 | Create SKDM for group |
| `process()` (X3DH) | `SignalSessionBuilder.java` | 22-26 | X3DH key agreement |
| `create()` (Sender Key) | `SignalGroupSessionBuilder.java` | 30-34 | Create sender key session |
| `rotateOurKey()` | `SenderKeyUtil.java` | 18-23 | Rotate sender key on membership change |
| `markAsShared()` | `SenderKeySharedTable.kt` | 47-58 | Mark sender key shared with recipient |
| `storeSession()` | `SessionTable.kt` | 44-56 | Store session record |
| `saveIdentity()` | `SignalBaseIdentityKeyStore.java` | 74-116 | Save identity key, detect changes |

### Protocol Layer (libsignal-service)

| Component | File | Lines | Responsibility |
|-----------|------|-------|----------------|
| **SignalServiceMessageSender** | `lib/.../SignalServiceMessageSender.java` | 170-3114 | Outgoing messages, session establishment, sender key distribution |
| **SignalServiceMessageReceiver** | `lib/.../SignalServiceMessageReceiver.java` | 50-800 | Incoming messages, envelope fetching |
| **SignalServiceCipher** | `lib/.../SignalServiceCipher.java` | 69-284 | Envelope encryption/decryption |
| **SignalSessionBuilder** | `lib/.../SignalSessionBuilder.java` | 12-27 | X3DH session establishment wrapper |
| **SignalGroupSessionBuilder** | `lib/.../SignalGroupSessionBuilder.java` | 14-34 | Sender key session creation |
| **SignalGroupCipher** | `lib/.../SignalGroupCipher.java` | 15-38 | Group message encryption/decryption |
| **SignalSealedSessionCipher** | `lib/.../SignalSealedSessionCipher.java` | 34-83 | Sealed sender encryption |

### Protocol Storage Layer

| Component | File | Lines | Responsibility |
|-----------|------|-------|----------------|
| **SessionTable** | `app/.../database/SessionTable.kt` | 22-240 | Session records per (address, device) |
| **SenderKeyTable** | `app/.../database/SenderKeyTable.kt` | 27-78 | Sender key sessions per distribution ID |
| **SenderKeySharedTable** | `app/.../database/SenderKeySharedTable.kt` | 23-150 | Tracks which recipients have sender key |
| **IdentityTable** | `app/.../database/IdentityTable.kt` | 48-250 | Identity keys with verification status |
| **TextSecureSessionStore** | `app/.../crypto/storage/TextSecureSessionStore.java` | 24-203 | Session store implementation |
| **SignalSenderKeyStore** | `app/.../crypto/storage/SignalSenderKeyStore.java` | 20-150 | Sender key store implementation |
| **SignalBaseIdentityKeyStore** | `app/.../crypto/storage/SignalBaseIdentityKeyStore.java` | 40-256 | Identity store implementation |

### Multi-Device Layer

| Component | File | Lines | Responsibility |
|-----------|------|-------|----------------|
| **LinkDeviceRepository** | `app/.../linkdevice/LinkDeviceRepository.kt` | 52-151 | Device linking via QR code |
| **PrimaryProvisioningCipher** | `lib/.../crypto/PrimaryProvisioningCipher.java` | 15-80 | Encrypt provisioning data for new device |
| **SecondaryProvisioningCipher** | `lib/.../crypto/SecondaryProvisioningCipher.kt` | 20-90 | Decrypt provisioning data on new device |
| **MultiDevice*Jobs** | `app/.../jobs/MultiDevice*.java` | varies | Sync jobs for linked devices |

### Protocol Jobs

| Job | File | Lines | Purpose |
|-----|------|-------|---------|
| **SenderKeyDistributionSendJob** | `app/.../jobs/SenderKeyDistributionSendJob.java` | 38-160 | Send SKDM to new group members |
| **RefreshPreKeysJob** | `app/.../jobs/RefreshPreKeysJob.java` | 25-120 | Refresh prekey supply from server |
| **RotateSignedPreKeyJob** | `app/.../jobs/RotateSignedPreKeyJob.java` | 20-80 | Rotate signed prekey periodically |

### Network Domain

| Component | Package | Responsibility |
|-----------|---------|----------------|
| **SignalServiceMessageSender** | `libsignal-service` | Outgoing message API |
| **SignalServiceMessageReceiver** | `libsignal-service` | Incoming message API |
| **PushServiceClient** | `push` | FCM handling |
| **WebSocketConnection** | `libsignal-service` | Real-time connection |

## Component Interactions

### Sending a Direct Message

```mermaid
sequenceDiagram
    participant UI as ConversationUI
    participant VM as ConversationViewModel
    participant MS as MessageSender
    participant Crypto as Crypto
    participant Net as SignalService
    participant DB as MessageTable

    UI->>VM: sendMessage(text)
    VM->>MS: send(recipient, message)
    MS->>Crypto: encrypt(message, recipientKey)
    Crypto-->>MS: encryptedMessage
    MS->>DB: store(message, encryptedMessage)
    MS->>Net: send(encryptedMessage)
    Net-->>MS: sendResult
    MS-->>VM: sendComplete
    VM-->>UI: updateUI(sent)
```

### Receiving a Message

```mermaid
sequenceDiagram
    participant Push as FCM
    participant Job as JobManager
    participant MR as MessageReceiver
    participant Crypto as Crypto
    participant DB as MessageTable
    participant UI as ConversationUI

    Push->>Job: wake(notification)
    Job->>MR: processMessage()
    MR->>MR: fetchFromServer()
    MR->>Crypto: decrypt(encryptedMessage)
    Crypto-->>MR: decryptedMessage
    MR->>DB: store(message)
    DB-->>UI: notifyObservers()
    UI->>UI: displayMessage()
```

### Group Message Flow

```mermaid
sequenceDiagram
    participant UI as ConversationUI
    participant GM as GroupManager
    participant MS as MessageSender
    participant Crypto as Crypto
    participant Net as SignalService

    UI->>GM: sendGroupMessage(text, groupId)
    GM->>Crypto: getSenderKey(groupId)
    Crypto-->>GM: senderKey
    GM->>MS: sendWithSenderKey(message, groupId)
    MS->>Crypto: encryptWithSenderKey(message, senderKey)
    Crypto-->>MS: encryptedGroupMessage
    MS->>Net: sendToGroup(encryptedGroupMessage, groupId)
    Net-->>MS: distributionResult
    MS-->>UI: sendComplete
```

### Session Establishment Flow

```mermaid
sequenceDiagram
    participant App as App Layer
    participant SSM as SignalServiceMessageSender
    participant KeysApi as KeysApi
    participant Builder as SessionBuilder
    participant Store as SessionTable
    participant Server as Signal Server

    App->>SSM: send(recipient, message)
    SSM->>SSM: checkSession(recipient)
    alt No Session Exists
        SSM->>KeysApi: getPreKeys(recipient)
        KeysApi->>Server: GET /v2/keys/{recipient}
        Server-->>KeysApi: PreKeyBundle
        KeysApi-->>SSM: PreKeyBundle
        
        loop For each device
            SSM->>Builder: process(preKey)
            Note over Builder: X3DH Key Agreement
            Builder->>Store: storeSession(address, record)
        end
    end
    SSM->>SSM: encrypt(message)
    SSM->>Server: send(encryptedMessage)
```

### Sender Key Distribution Flow

```mermaid
sequenceDiagram
    participant App as GroupSendUtil
    participant SSM as SignalServiceMessageSender
    participant GSB as SignalGroupSessionBuilder
    participant SKStore as SenderKeySharedTable
    participant Server as Signal Server

    App->>SSM: sendGroupMessage(distributionId, recipients)
    SSM->>SKStore: getSharedWith(distributionId)
    SKStore-->>SSM: sharedRecipients
    
    loop For recipients not in sharedRecipients
        SSM->>GSB: create(self, distributionId)
        GSB-->>SSM: SenderKeyDistributionMessage
        SSM->>Server: sendSKDM(recipient, SKDM)
        Server-->>SSM: success
        SSM->>SKStore: markShared(distributionId, recipient)
    end
    
    SSM->>SSM: encryptForGroup(message)
    SSM->>Server: sendGroupMessage(ciphertext)
```

## Database Tables

### Core Tables

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| **ThreadTable** | Conversations | `_id`, `recipient_id`, `snippet`, `date` |
| **MessageTable** | Messages | `_id`, `thread_id`, `body`, `type`, `date_received` |
| **RecipientTable** | Contacts/users | `_id`, `phone`, `username`, `profile_name` |
| **GroupTable** | Groups | `_id`, `group_id`, `title`, `members` |

### Encryption Tables

| Table | Purpose |
|-------|---------|
| **IdentityTable** | Identity keys for users |
| **SessionTable** | Signal Protocol sessions |
| **OneTimePreKeyTable** | One-time prekeys |
| **SignedPreKeyTable** | Signed prekeys |
| **KyberPreKeyTable** | Post-quantum prekeys |
| **SenderKeyTable** | Group sender keys |

### Feature Tables

| Table | Purpose |
|-------|---------|
| **CallTable** | Call history |
| **CallLinkTable** | Call links for group calls |
| **PaymentTable** | Payment transactions |
| **AttachmentTable** | Message attachments |
| **StickerTable** | Sticker packs |
| **StorySendTable** | Stories |

## Job System Components

| Job Type | Purpose |
|----------|---------|
| **PushProcessJob** | Process incoming push notifications |
| **MessageSendJob** | Send outgoing messages |
| **AttachmentUploadJob** | Upload attachments to CDN |
| **AttachmentDownloadJob** | Download attachments from CDN |
| **GroupUpdateJob** | Sync group state with server |
| **ProfileUploadJob** | Upload user profile |

## Related Documentation

- [Container Diagram](C4-Container-Diagram.md) - High-level containers
- [Database](Database.md) - Complete database schema
- [Module Structure](Module-Structure.md) - Module organization
- [Session & Sender Key Distribution](Session-And-SenderKey-Distribution-Flow.md) - Technical deep dive with code references
- [E2EE Session Initialization](E2EE-Session-Initialization-Flow.md) - X3DH and prekey exchange details