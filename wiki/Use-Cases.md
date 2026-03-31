# Use Cases

> Key user interactions and business workflows in Signal Android.

## Primary Use Cases

### UC-01: Register Account

**Actor**: New User
**Goal**: Create a new Signal account

```mermaid
flowchart TD
    A[Open App] --> B{Has Account?}
    B -->|No| C[Enter Phone Number]
    C --> D[Receive Verification Code]
    D --> E[Enter Verification Code]
    E --> F{Set PIN?}
    F -->|Yes| G[Create PIN for Backup]
    F -->|No| H[Skip PIN]
    G --> I[Account Created]
    H --> I
    B -->|Yes| J[Enter PIN/Biometric]
    J --> K[Access Account]
```

**Preconditions**:
- Valid phone number
- Network connectivity
- Not rate-limited

**Main Flow**:
1. User opens Signal
2. User enters phone number
3. Signal sends verification code via SMS/voice
4. User enters verification code
5. (Optional) User creates PIN for SVR2 backup
6. Signal generates cryptographic keys
7. Account is created on server

**Alternative Flows**:
- **Device Transfer**: Transfer account from another device via QR code
- **Restore Backup**: Restore from local or cloud backup

**Code Location**: `feature/registration/`

---

### UC-02: Send Direct Message

**Actor**: Registered User
**Goal**: Send an encrypted message to a contact

```mermaid
flowchart TD
    A[Select Conversation] --> B[Compose Message]
    B --> C{Add Attachment?}
    C -->|Yes| D[Select Media]
    D --> E[Compress/Encrypt]
    C -->|No| F[Encrypt Message]
    E --> F
    F --> G[Send via WebSocket]
    G --> H{Success?}
    H -->|Yes| I[Mark as Sent]
    H -->|No| J[Queue for Retry]
    J --> K[Show Pending]
```

**Preconditions**:
- User has Signal account
- Recipient is a Signal user
- Network connectivity (or queued for later)

**Main Flow**:
1. User selects or creates conversation
2. User composes message text
3. (Optional) User adds attachments
4. Signal encrypts message using Signal Protocol
5. Signal sends encrypted message to server
6. Signal stores message in local database
7. Server delivers to recipient(s)

**Business Rules**:
- Messages are end-to-end encrypted
- Failed sends are queued with exponential backoff
- Disappearing messages auto-delete after timer

**Code Location**: `messages/`, `conversation/`

---

### UC-03: Receive Message

**Actor**: Registered User
**Goal**: Receive and read an encrypted message

```mermaid
flowchart TD
    A[Push Notification] --> B[Wake App]
    B --> C[Fetch Message]
    C --> D[Decrypt Message]
    D --> E{From Known Contact?}
    E -->|Yes| F[Display Message]
    E -->|No| G{Message Request?}
    G -->|Yes| H[Show Request]
    G -->|No| F
    H --> I{Accept?}
    I -->|Yes| F
    I -->|No| J[Block/Archive]
```

**Main Flow**:
1. Signal receives push notification
2. App wakes and fetches encrypted message
3. Signal decrypts message using Signal Protocol
4. Signal stores in local database
5. UI displays message in conversation

**Business Rules**:
- Unknown senders create message requests
- Read receipts sent if enabled
- Message appears in notification

**Code Location**: `push/`, `messages/`

---

### UC-04: Create Group

**Actor**: Registered User
**Goal**: Create a new Signal group

```mermaid
flowchart TD
    A[Start Group Creation] --> B[Select Members]
    B --> C[Set Group Name]
    C --> D{Set Permissions?}
    D -->|Yes| E[Configure Access Control]
    D -->|No| F[Create Group]
    E --> F
    F --> G[Generate Group ID]
    G --> H[Create Group V2 on Server]
    H --> I[Send Invitations to Members]
    I --> J[Group Created]
```

**Preconditions**:
- User has Signal account
- Selected members are Signal users
- User has network connectivity

**Main Flow**:
1. User initiates group creation
2. User selects members from contacts
3. User sets group name and avatar
4. (Optional) User configures permissions
5. Signal creates Group V2 on server
6. Signal sends invitations to members
7. Group appears in user's conversation list

**Business Rules**:
- Groups use Group V2 protocol
- Maximum 1,000 members
- Creator is automatically admin

**Code Location**: `groups/`

---

### UC-05: Make Voice/Video Call

**Actor**: Registered User
**Goal**: Initiate an encrypted voice or video call

```mermaid
flowchart TD
    A[Select Contact] --> B[Initiate Call]
    B --> C[Generate Call Keys]
    C --> D[Send Call Offer]
    D --> E{Answered?}
    E -->|Yes| F[Establish WebRTC Connection]
    F --> G[Start Media Streams]
    G --> H[Call Active]
    E -->|No| I[Call Ended]
    E -->|Declined| J[Show Declined]
```

**Preconditions**:
- User has Signal account
- Recipient is a Signal user
- Microphone/camera permission (if video)

**Main Flow**:
1. User selects contact and initiates call
2. Signal generates call cryptographic keys
3. Signal sends call offer via server
4. Recipient receives ringing notification
5. On answer, WebRTC connection established
6. Encrypted media streams begin
7. Call UI shows video/audio controls

**Technical Details**:
- Uses RingRTC (Signal's WebRTC fork)
- Media encrypted with SRTP
- Signaling via WebSocket
- TURN servers for NAT traversal

**Code Location**: `components/webrtc/`, `ringrtc/`

---

### UC-06: Send Payment

**Actor**: Registered User
**Goal**: Send MobileCoin to a contact

```mermaid
flowchart TD
    A[Select Recipient] --> B[Enter Amount]
    B --> C[Review Transaction]
    C --> D{Confirm?}
    D -->|Yes| E[Authenticate]
    E --> F[Calculate Fee]
    F --> G[Submit Transaction]
    G --> H{Success?}
    H -->|Yes| I[Update Balance]
    H -->|No| J[Show Error]
```

**Preconditions**:
- User has Signal account
- User has set up MobileCoin wallet
- User has sufficient balance + fee
- Recipient has payments enabled

**Main Flow**:
1. User selects recipient for payment
2. User enters amount and optional note
3. Signal estimates transaction fee
4. User confirms payment
5. User authenticates (PIN/biometric)
6. Signal submits transaction to MobileCoin network
7. Transaction is recorded locally
8. Payment appears in chat

**Business Rules**:
- Payments are irreversible
- Fees are paid to MobileCoin network
- Geographic restrictions may apply

**Code Location**: `payments/`

---

### UC-07: Link Desktop Device

**Actor**: Registered User
**Goal**: Link a desktop client to existing account

```mermaid
flowchart TD
    A[Open Device Settings] --> B[Show QR Code]
    B --> C[Scan with Desktop]
    C --> D[Desktop Sends Request]
    D --> E{Approve?}
    E -->|Yes| F[Generate Device Key]
    F --> G[Send Provisioning Message]
    G --> H[Desktop Syncs]
    E -->|No| I[Cancel]
```

**Preconditions**:
- User has Signal account on mobile
- Desktop app is installed
- Both devices have network connectivity

**Main Flow**:
1. User opens linked devices settings
2. Signal displays QR code with provisioning info
3. User scans QR code with desktop app
4. Desktop sends provisioning request
5. Mobile prompts user to approve
6. On approval, Signal generates device-specific keys
7. Desktop receives provisioning message
8. Desktop syncs account state

**Business Rules**:
- Primary device is always mobile
- Each device has unique identity key
- Messages are encrypted per-device

**Code Location**: `linkdevice/`

---

### UC-08: Backup and Restore

**Actor**: Registered User
**Goal**: Backup and restore Signal data

#### Backup Flow

```mermaid
flowchart TD
    A[Open Backup Settings] --> B[Enter PIN/Biometric]
    B --> C[Generate Backup Key]
    C --> D[Create Backup File]
    D --> E[Encrypt Local Data]
    E --> F{Backup Location?}
    F -->|Local| G[Save to Storage]
    F -->|Cloud| H[Upload to Cloud]
```

#### Restore Flow

```mermaid
flowchart TD
    A[Registration Screen] --> B[Select Restore]
    B --> C{Backup Source?}
    C -->|Local| D[Select Backup File]
    C -->|Cloud| E[Download from Cloud]
    D --> F[Enter Backup Key]
    E --> F
    F --> G[Decrypt Backup]
    G --> H[Restore Database]
    H --> I[Account Restored]
```

**Backup Types**:
- **Full Backup**: Complete database backup (local file)
- **SVR2 Backup**: Cryptographic keys backed up to Signal servers
- **Media Backup**: Attachments backed up separately

**Code Location**: `backup/`

---

## Protocol Use Cases

### UC-09: Establish Session

**Actor**: Sender Device
**Goal**: Create encrypted channel with recipient device

```mermaid
flowchart TD
    A[Send Message] --> B{Session Exists?}
    B -->|Yes| C[Encrypt with Session]
    B -->|No| D[Fetch PreKey Bundle]
    D --> E[Validate Signed PreKey]
    E --> F[Perform X3DH Key Agreement]
    F --> G[Derive Root Key]
    G --> H[Initialize Double Ratchet]
    H --> I[Store Session]
    I --> C
    C --> J[Send Encrypted Message]
```

**Preconditions**:
- Sender has Signal account
- Recipient is registered on Signal
- Network connectivity

**Main Flow**:
1. Sender initiates message send
2. Signal checks for existing session with recipient device
3. If no session, fetch PreKey bundle from server
4. Validate signed PreKey signature with identity key
5. Perform X3DH key agreement:
   - DH1 = DH(IK_sender, SPK_recipient)
   - DH2 = DH(EK_sender, IK_recipient)
   - DH3 = DH(EK_sender, SPK_recipient)
   - DH4 = DH(EK_sender, OPK_recipient) if available
6. Derive root key from combined DH outputs
7. Initialize Double Ratchet state
8. Store session in SessionTable
9. Encrypt message with new session

**Multi-Device Handling**:
- For recipients with multiple devices, establish session for each device
- Each device gets unique session with independent ratchet state
- All sessions use same identity key but different PreKeys

**Code Location**: `lib/libsignal-service/.../SignalServiceMessageSender.java:2831-2870`

---

### UC-10: Distribute Sender Key

**Actor**: Group Member
**Goal**: Share group encryption key with other members

```mermaid
flowchart TD
    A[Send Group Message] --> B{Sender Key Exists?}
    B -->|No| C[Create Sender Key Session]
    B -->|Yes| D{Key Age OK?}
    D -->|No| C
    D -->|Yes| E[Check Shared Status]
    C --> F[Get Recipients Needing Key]
    E --> F
    F --> G{Recipients Need Key?}
    G -->|Yes| H[Create SKDM]
    H --> I[Send SKDM to Each Device]
    I --> J[Mark as Shared]
    G -->|No| K[Encrypt with Sender Key]
    J --> K
    K --> L[Send Group Message]
```

**Preconditions**:
- Sender is member of Signal group
- DistributionId exists for group

**Main Flow**:
1. Sender initiates group message send
2. Signal checks sender key age, rotates if too old
3. Signal checks which recipients have received this sender key
4. For recipients missing the key:
   - Create SenderKeyDistributionMessage (SKDM)
   - Send SKDM to all devices of each recipient
   - Mark sender key as shared in SenderKeySharedTable
5. Encrypt message once with sender key
6. Send encrypted message to all recipients

**Security Guarantees**:
- One encryption for all recipients (efficient)
- Sender key rotated when membership changes
- Removed members cannot decrypt future messages

**Code Location**: `lib/libsignal-service/.../SignalServiceMessageSender.java:2480-2613`

---

### UC-11: Handle Group Membership Change

**Actor**: System
**Goal**: Maintain group security when membership changes

```mermaid
flowchart TD
    A[Membership Change Detected] --> B{Change Type?}
    B -->|Member Added| C[Get Group State]
    B -->|Member Removed| D[Update Local Group]
    C --> E[Enqueue SKDM to New Member]
    D --> F[Rotate Sender Key]
    E --> G[New Member Can Decrypt]
    F --> H[Clear Shared Status]
    H --> I[Next Send Distributes New Key]
```

**Member Added Flow**:
1. Server notifies of new member
2. Local group state updated
3. SenderKeyDistributionSendJob enqueued
4. SKDM sent to new member's devices
5. Member can decrypt future group messages

**Member Removed Flow**:
1. Server notifies of member removal
2. Local group state updated
3. Sender key rotated (deleted + cleared shared status)
4. Next group message creates new sender key
5. New key distributed to remaining members only
6. Removed member cannot decrypt future messages

**Code Location**: `app/.../groups/v2/processing/GroupsV2StateProcessor.kt`, `app/.../crypto/SenderKeyUtil.java:18-23`

---

## Secondary Use Cases

### UC-09: Report Spam/Block User

**Actor**: Registered User
**Goal**: Block unwanted contact

1. User views conversation or profile
2. User selects "Block" option
3. Signal adds recipient to blocked list
4. Future messages are silently discarded
5. User can optionally report as spam

**Code Location**: `blocked/`

---

### UC-10: Set Disappearing Messages

**Actor**: Registered User
**Goal**: Configure message auto-deletion

1. User opens conversation settings
2. User selects disappearing messages
3. User chooses duration (off, 30s, 5m, 1h, 1d, 1w)
4. Signal updates conversation settings
5. New messages auto-delete after duration

**Code Location**: `conversation/`

---

### UC-11: Create Call Link

**Actor**: Registered User
**Goal**: Create shareable call link for group call

1. User selects "Create Call Link"
2. Signal generates unique call link
3. User can share link with anyone
4. Recipients can join via link
5. Call host can administer call

**Code Location**: `calls/links/`

---

### UC-12: Send Story

**Actor**: Registered User
**Goal**: Share ephemeral content with contacts

1. User captures or selects media
2. User adds text, drawings, or filters
3. User selects audience (all contacts, custom)
4. Signal posts story
5. Story expires after 24 hours

**Code Location**: `stories/`

---

## Related Documentation

- [Business Domain](Business-Domain.md) - Domain concepts and terminology
- [C4 Component Diagram](C4-Component-Diagram.md) - Technical implementation
- [Features](Features.md) - Feature overview