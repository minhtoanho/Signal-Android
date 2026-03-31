# Session Establishment & Sender Key Distribution Flow

> **This document explains how Signal-Android establishes sessions and distributes sender keys, including multi-device handling, group membership changes, and identity management.**

## Table of Contents

1. [Overview](#1-overview)
2. [Session Establishment Flow](#2-session-establishment-flow)
3. [Sender Key Distribution Mechanism](#3-sender-key-distribution-mechanism)
4. [Multi-Device Architecture](#4-multi-device-architecture)
5. [Group Membership Changes](#5-group-membership-changes)
6. [Identity Management](#6-identity-management)
7. [Complete Flow Diagrams](#7-complete-flow-diagrams)

---

## 1. Overview

### Key Concepts

| Concept | Purpose | Location |
|---------|---------|----------|
| **Session** | 1:1 encrypted communication channel between two devices | `SessionTable.kt` |
| **Sender Key** | Shared key for efficient group messaging | `SenderKeyTable.kt` |
| **Distribution ID** | Unique identifier for a group's sender key session | `DistributionId` |
| **SignalProtocolAddress** | Identifies a recipient+device combination | `SignalProtocolAddress(name, deviceId)` |
| **Identity Key** | Long-term key for user authentication | `IdentityTable.kt` |

### Architecture Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                             │
│  GroupSendUtil.java, MessageSender.kt                           │
├─────────────────────────────────────────────────────────────────┤
│                    PROTOCOL LAYER                                │
│  SignalServiceMessageSender.java, SignalServiceCipher.java      │
│  SignalGroupSessionBuilder.java, SignalGroupCipher.java         │
├─────────────────────────────────────────────────────────────────┤
│                    CRYPTOGRAPHIC LAYER (libsignal)              │
│  SessionBuilder, SessionCipher, GroupSessionBuilder, GroupCipher│
├─────────────────────────────────────────────────────────────────┤
│                    STORAGE LAYER                                 │
│  SessionTable, SenderKeyTable, SenderKeySharedTable, IdentityTable│
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Session Establishment Flow

### 2.1 When Session Establishment is Triggered

Session establishment occurs when:
1. **First message to a recipient** - No existing session found
2. **Session archived** - Previous session was invalidated
3. **Identity key changed** - Recipient's identity key is different
4. **New device added** - Recipient has a new linked device

### 2.2 Complete Session Establishment Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│            SESSION ESTABLISHMENT ON SEND                             │
│            SignalServiceMessageSender.getEncryptedMessage()          │
│            File: lib/libsignal-service/.../SignalServiceMessageSender.java │
│            Lines: 2831-2870                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ENTRY: sendMessage() detects no session exists                     │
│                                                                     │
│  STEP 1: Check for Existing Session                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ SignalProtocolAddress signalProtocolAddress =                 │  │
│  │   new SignalProtocolAddress(recipient.getIdentifier(), deviceId); │
│  │                                                               │  │
│  │ if (!aciStore.containsSession(signalProtocolAddress)) {      │  │
│  │   // Need to establish session via PreKey bundle             │  │
│  │   fetchPreKeysAndEstablishSession();                          │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2841               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: Fetch PreKey Bundle from Server                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ List<PreKeyBundle> preKeys = getPreKeys(                      │  │
│  │   recipient, sealedSenderAccess, deviceId, story              │  │
│  │ );                                                            │  │
│  │                                                               │  │
│  │ // getPreKeys() calls KeysApi:                                │  │
│  │ // GET /v2/keys/{identifier}/{deviceSpecifier}                │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2843               │  │
│  │            KeysApi.kt:188-259                                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 3: Process PreKey Bundle (X3DH Key Agreement)                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ for (PreKeyBundle preKey : preKeys) {                         │  │
│  │   SignalProtocolAddress preKeyAddress =                       │  │
│  │     new SignalProtocolAddress(recipient.getIdentifier(),      │  │
│  │                                preKey.getDeviceId());         │  │
│  │                                                               │  │
│  │   SignalSessionBuilder sessionBuilder =                       │  │
│  │     new SignalSessionBuilder(sessionLock,                     │  │
│  │       new SessionBuilder(aciStore, preKeyAddress));           │  │
│  │                                                               │  │
│  │   sessionBuilder.process(preKey); // X3DH happens here        │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2845-2855          │  │
│  │            SignalSessionBuilder.java:22-26                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 4: Store Session in Database                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // SessionBuilder.process() calls storeSession():             │  │
│  │ SessionTable.store(serviceId, address, record)                │  │
│  │   └── INSERT INTO sessions (account_id, address, device, record) │  │
│  │                                                               │  │
│  │ Reference: SessionTable.kt:44-56                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  OUTPUT: Session established, ready for message encryption         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Multi-Device Session Establishment

When a recipient has multiple devices, sessions must be established for EACH device:

```
┌─────────────────────────────────────────────────────────────────────┐
│          MULTI-DEVICE SESSION ESTABLISHMENT                          │
│          SignalServiceMessageSender.getEncryptedMessages()           │
│          File: lib/libsignal-service/.../SignalServiceMessageSender.java │
│          Lines: 2800-2828                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  STEP 1: Get All Recipient Devices                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Get sub-devices (linked devices)                           │  │
│  │ List<Integer> subDevices = aciStore.getSubDeviceSessions(     │  │
│  │   recipient.getIdentifier()                                   │  │
│  │ );                                                            │  │
│  │                                                               │  │
│  │ // Build device list: primary + all sub-devices              │  │
│  │ List<Integer> deviceIds = new ArrayList<>(subDevices.size() + 1);│
│  │ deviceIds.add(SignalServiceAddress.DEFAULT_DEVICE_ID); // Primary (1)│
│  │ deviceIds.addAll(subDevices); // Linked devices              │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2811-2815          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: Establish Sessions for Each Device                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ for (int deviceId : deviceIds) {                              │  │
│  │   // Skip our own device (don't send to self)                │  │
│  │   if (recipient.matches(localAddress) && deviceId == localDeviceId) {│
│  │     continue;                                                 │  │
│  │   }                                                           │  │
│  │                                                               │  │
│  │   // Check if session exists for this device                 │  │
│  │   if (aciStore.containsSession(                               │  │
│  │     new SignalProtocolAddress(recipient.getIdentifier(), deviceId)│
│  │   )) {                                                        │  │
│  │     messages.add(getEncryptedMessage(recipient, deviceId, ...));│
│  │   }                                                           │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2821-2825          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  IMPORTANT: Each device has its own session stored as:              │
│  - SessionTable row: (account_id, address, device_id, record)       │
│  - address = recipient's ServiceId (same for all devices)          │
│  - device_id = unique per device (1 = primary, 2+ = linked)        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Sender Key Distribution Mechanism

### 3.1 What is a Sender Key?

Sender Keys enable efficient group messaging:
- **One encryption** for all group members (instead of N individual encryptions)
- **Per-sender, per-group** unique key (DistributionId)
- **Sender Key Session** = (chain key, ratchet state) stored in SenderKeyTable

### 3.2 Sender Key Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│          SENDER KEY STORAGE ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SenderKeyTable (sender_keys)                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Columns:                                                      │  │
│  │   - address: TEXT    (sender's ServiceId)                    │  │
│  │   - device: INTEGER  (sender's device ID)                    │  │
│  │   - distribution_id: TEXT (group identifier)                 │  │
│  │   - record: BLOB     (SenderKeyRecord serialized)            │  │
│  │   - created_at: INTEGER                                      │  │
│  │                                                               │  │
│  │ Unique constraint: (address, device, distribution_id)        │  │
│  │                                                               │  │
│  │ File: app/.../database/SenderKeyTable.kt:27-48               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  SenderKeySharedTable (sender_key_shared)                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Columns:                                                      │  │
│  │   - distribution_id: TEXT (which group's sender key)         │  │
│  │   - address: TEXT    (who has received this key)             │  │
│  │   - device: INTEGER  (which device received it)              │  │
│  │   - timestamp: INTEGER (when it was shared)                  │  │
│  │                                                               │  │
│  │ Purpose: Track which recipients know about a sender key      │  │
│  │          to avoid redundant SKDM sends                        │  │
│  │                                                               │  │
│  │ File: app/.../database/SenderKeySharedTable.kt:23-42         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Sender Key Distribution Flow (When Joining New Conversation)

```
┌─────────────────────────────────────────────────────────────────────┐
│          SENDER KEY DISTRIBUTION ON GROUP SEND                       │
│          SignalServiceMessageSender.sendGroupMessage()               │
│          File: lib/libsignal-service/.../SignalServiceMessageSender.java │
│          Lines: 2480-2613                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  TRIGGER: sendGroupDataMessage() called                             │
│                                                                     │
│  STEP 1: Determine Who Needs Sender Key                            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Get addresses that already have the sender key            │  │
│  │ Set<SignalProtocolAddress> sharedWith =                       │  │
│  │   aciStore.getSenderKeySharedWith(distributionId);           │  │
│  │                                                               │  │
│  │ // Find recipients missing sender key                         │  │
│  │ List<SignalServiceAddress> needsSenderKeyTargets =           │  │
│  │   targetInfo.destinations.stream()                            │  │
│  │     .filter(a -> !sharedWith.contains(a) ||                   │  │
│  │                   targetInfo.sessions.get(a) == null)         │  │
│  │     .collect(Collectors.toList());                            │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2484-2490          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: Create Sender Key Distribution Message (SKDM)             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ if (needsSenderKeyTargets.size() > 0) {                       │  │
│  │   // Create or get existing sender key session               │  │
│  │   SenderKeyDistributionMessage senderKeyDistributionMessage = │  │
│  │     getOrCreateNewGroupSession(distributionId);               │  │
│  │                                                               │  │
│  │   // getOrCreateNewGroupSession implementation:              │  │
│  │   SignalProtocolAddress self =                                │  │
│  │     new SignalProtocolAddress(localAddress.getIdentifier(), localDeviceId);│
│  │   return new SignalGroupSessionBuilder(sessionLock,           │  │
│  │     new GroupSessionBuilder(aciStore))                        │  │
│  │       .create(self, distributionId.asUuid());                 │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2493               │  │
│  │            SignalServiceMessageSender.java:503-506            │  │
│  │            SignalGroupSessionBuilder.java:30-34               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 3: Send SKDM to Recipients Missing Key                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ List<SendMessageResult> results =                             │  │
│  │   sendSenderKeyDistributionMessage(                           │  │
│  │     distributionId,                                           │  │
│  │     needsSenderKeyTargets,                                    │  │
│  │     needsSenderKeySealedSenderAccesses,                       │  │
│  │     senderKeyDistributionMessage,                             │  │
│  │     groupId,                                                  │  │
│  │     urgent,                                                   │  │
│  │     story                                                     │  │
│  │   );                                                          │  │
│  │                                                               │  │
│  │ // sendSenderKeyDistributionMessage creates Content envelope: │  │
│  │ Content content = new Content.Builder()                       │  │
│  │   .senderKeyDistributionMessage(ByteString.of(message.serialize()))│
│  │   .build();                                                   │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2501-2507          │  │
│  │            SignalServiceMessageSender.java:511-528            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 4: Mark Sender Key as Shared with Successful Recipients      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Get successful recipients                                  │  │
│  │ List<SignalServiceAddress> successes = results.stream()       │  │
│  │   .filter(SendMessageResult::isSuccess)                       │  │
│  │   .map(SendMessageResult::getAddress)                         │  │
│  │   .collect(Collectors.toList());                              │  │
│  │                                                               │  │
│  │ // Convert to protocol addresses (with device IDs)           │  │
│  │ Set<SignalProtocolAddress> successAddresses = ...;            │  │
│  │                                                               │  │
│  │ // Mark as shared in SenderKeySharedTable                     │  │
│  │ aciStore.markSenderKeySharedWith(distributionId, successAddresses);│
│  │   └── SenderKeySharedTable.markAsShared(distributionId, addresses)│
│  │       └── INSERT INTO sender_key_shared (...) VALUES (...)   │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2509-2517          │  │
│  │            SenderKeySharedTable.kt:47-58                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 5: Encrypt Group Message with Sender Key                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ SignalServiceCipher cipher = new SignalServiceCipher(...);    │  │
│  │                                                               │  │
│  │ byte[] ciphertext = cipher.encryptForGroup(                   │  │
│  │   distributionId,           // Which sender key to use        │  │
│  │   targetInfo.destinations,  // Recipient addresses            │  │
│  │   targetInfo.sessions,      // Existing sessions               │  │
│  │   sealedSenderAccess.getSenderCertificate(),                  │  │
│  │   content.encode(),          // Message plaintext              │  │
│  │   contentHint,                                                 │  │
│  │   groupId                                                     │  │
│  │ );                                                            │  │
│  │                                                               │  │
│  │ Reference: SignalServiceMessageSender.java:2550-2554          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  OUTPUT: Group message encrypted once, delivered to all recipients  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.4 Sender Key Distribution Job (Dedicated SKDM Send)

For targeted sender key distribution (e.g., new member joining):

```
┌─────────────────────────────────────────────────────────────────────┐
│          SENDER KEY DISTRIBUTION SEND JOB                            │
│          File: app/.../jobs/SenderKeyDistributionSendJob.java       │
│          Lines: 81-139                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  TRIGGER: New member added to group                                │
│                                                                     │
│  STEP 1: Verify Member Still in Group                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Get group and distribution ID                              │  │
│  │ GroupId.V2 groupId = threadRecipient.requireGroupId().requireV2();│
│  │ DistributionId distributionId =                               │  │
│  │   SignalDatabase.groups().getOrCreateDistributionId(groupId); │  │
│  │                                                               │  │
│  │ // Check membership at send time                              │  │
│  │ if (!SignalDatabase.groups().isCurrentMember(groupId, targetRecipientId)) {│
│  │   Log.w(TAG, targetRecipientId + " is no longer a member!");  │  │
│  │   return; // Skip send                                        │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SenderKeyDistributionSendJob.java:90-121           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: Create Sender Key Distribution Message                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ SignalServiceMessageSender messageSender =                    │  │
│  │   AppDependencies.getSignalServiceMessageSender();            │  │
│  │                                                               │  │
│  │ SenderKeyDistributionMessage message =                        │  │
│  │   messageSender.getOrCreateNewGroupSession(distributionId);   │  │
│  │                                                               │  │
│  │ Reference: SenderKeyDistributionSendJob.java:123-125          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 3: Send to Target Recipient                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ SendMessageResult result =                                    │  │
│  │   messageSender.sendSenderKeyDistributionMessage(             │  │
│  │     distributionId,                                           │  │
│  │   Collections.singletonList(address),                         │  │
│  │     access,                                                   │  │
│  │     message,                                                  │  │
│  │     Optional.ofNullable(groupId).map(GroupId::getDecodedId), │  │
│  │     false, false                                              │  │
│  │ ).get(0);                                                     │  │
│  │                                                               │  │
│  │ Reference: SenderKeyDistributionSendJob.java:128              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 4: Mark as Shared on Success                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ if (result.isSuccess()) {                                     │  │
│  │   List<SignalProtocolAddress> addresses =                     │  │
│  │     result.getSuccess().getDevices().stream()                 │  │
│  │       .map(device -> targetRecipient.requireServiceId()       │  │
│  │         .toProtocolAddress(device))                           │  │
│  │       .collect(Collectors.toList());                          │  │
│  │                                                               │  │
│  │   AppDependencies.getProtocolStore().aci()                    │  │
│  │     .markSenderKeySharedWith(distributionId, addresses);      │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SenderKeyDistributionSendJob.java:130-137          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Multi-Device Architecture

### 4.1 Device Types

| Device Type | Device ID | Description |
|-------------|-----------|-------------|
| Primary | 1 (DEFAULT_DEVICE_ID) | First registered device, holds master key |
| Linked | 2, 3, 4, ... | Additional devices linked via QR code |

### 4.2 Device Linking Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│          DEVICE LINKING FLOW                                         │
│          File: app/.../linkdevice/LinkDeviceRepository.kt           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PRIMARY DEVICE                          NEW LINKED DEVICE          │
│  ===============                         ==================         │
│                                                                     │
│  STEP 1: Primary Generates QR Code                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // QR code contains:                                          │  │
│  │ // - ephemeralId (UUID)                                       │  │
│  │ // - publicKey (ECPublicKey)                                  │  │
│  │ URI: sgnl://linkdevice?uuid={ephemeralId}&pub_key={encodedKey}│  │
│  │                                                               │  │
│  │ Reference: LinkDeviceRepository.kt:136-151                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: New Device Scans QR Code                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // New device extracts ephemeral ID and public key           │  │
│  │ // Generates its own key pair                                │  │
│  │ // Sends verification code request to server                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 3: Primary Encrypts Provisioning Data                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // PrimaryProvisioningCipher encrypts:                        │  │
│  │ // - aciIdentityKeyPair                                       │  │
│  │ // - pniIdentityKeyPair                                       │  │
│  │ // - profileKey                                               │  │
│  │ // - master key (for storage service)                         │  │
│  │                                                               │  │
│  │ // File: lib/.../internal/crypto/PrimaryProvisioningCipher.java│  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 4: Link Device API Call                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // POST to link device endpoint                              │  │
│  │ LinkDeviceApi.linkDevice(encryptedProvisioningData)           │  │
│  │                                                               │  │
│  │ // File: lib/.../api/link/LinkDeviceApi.kt                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 5: New Device Decrypts and Stores Keys                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // SecondaryProvisioningCipher decrypts provisioning data    │  │
│  │ // Stores:                                                    │  │
│  │ // - Identity key pairs (same as primary)                    │  │
│  │ // - Profile key                                              │  │
│  │ // - Registration ID (unique per device)                     │  │
│  │                                                               │  │
│  │ // File: lib/.../internal/crypto/SecondaryProvisioningCipher.kt│  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  OUTPUT: New device registered with same identity, different deviceId│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.3 Multi-Device Sync Jobs

When changes occur on primary device, sync to linked devices:

| Job | Purpose | File |
|-----|---------|------|
| `MultiDeviceKeysUpdateJob` | Sync master key, storage key | `app/.../jobs/MultiDeviceKeysUpdateJob.kt` |
| `MultiDeviceContactUpdateJob` | Sync contact list | `app/.../jobs/MultiDeviceContactUpdateJob.java` |
| `MultiDeviceReadUpdateJob` | Sync read receipts | `app/.../jobs/MultiDeviceReadUpdateJob.java` |
| `MultiDeviceConfigurationUpdateJob` | Sync settings | `app/.../jobs/MultiDeviceConfigurationUpdateJob.java` |

### 4.4 Sessions for Multi-Device Recipients

When recipient has multiple devices, each needs its own session:

```
┌─────────────────────────────────────────────────────────────────────┐
│          SESSION STORAGE FOR MULTI-DEVICE RECIPIENT                 │
│          File: app/.../database/SessionTable.kt                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Example: Bob has primary (device 1) and linked tablet (device 2)   │
│                                                                     │
│  SessionTable entries:                                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ account_id | address (Bob's ACI) | device | record           │  │
│  │------------|----------------------|--------|------------------│  │
│  │ my_aci     | bob_aci_uuid        | 1      | SessionRecord_1  │  │
│  │ my_aci     | bob_aci_uuid        | 2      | SessionRecord_2  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Key Points:                                                        │
│  - Same address (Bob's ACI) for both rows                          │
│  - Different device IDs (1 and 2)                                  │
│  - Each device has its own SessionRecord with independent ratchet  │
│                                                                     │
│  Sender Key Storage:                                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ address (My ACI) | device | distribution_id | record          │  │
│  │------------------|--------|----------------|------------------│  │
│  │ my_aci           | 1      | group_uuid     | SenderKeyRecord │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Sender Key Shared With:                                            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ distribution_id | address (Bob's ACI) | device | timestamp    │  │
│  │-----------------|----------------------|--------|-------------│  │
│  │ group_uuid      | bob_aci_uuid         | 1      | 1234567890  │  │
│  │ group_uuid      | bob_aci_uuid         | 2      | 1234567890  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Note: Sender key must be shared with EACH device individually     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Group Membership Changes

### 5.1 Triggers for Session/Sender Key Re-establishment

| Event | Session Impact | Sender Key Impact |
|-------|----------------|-------------------|
| Member joins group | None (new member needs session) | **ROTATE** - new member needs key |
| Member leaves group | None (archived on leave) | **ROTATE** - removed member shouldn't decrypt |
| Identity key changed | Archive session, clear shared | Clear shared status |
| New device added | New session for device | Send SKDM to new device |

### 5.2 Member Join Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│          MEMBER JOIN FLOW                                            │
│          File: app/.../groups/v2/processing/GroupsV2StateProcessor.kt│
│          Lines: 165-200                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  TRIGGER: Server notification of group change with new member      │
│                                                                     │
│  STEP 1: Process Group State Change                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // GroupsV2StateProcessor receives DecryptedGroupChange       │  │
│  │ // Parses newMembers list                                     │  │
│  │ for (DecryptedMember newMember : change.newMembers) {         │  │
│  │   // Add member to local group state                          │  │
│  │   // Update group membership in database                      │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: GroupsV2StateProcessor.kt (process group change)   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: Sender Key Distribution to New Member                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // GroupManagerV2 or GroupSendUtil triggers:                  │  │
│  │ for (newMemberId : newMembers) {                              │  │
│  │   // Enqueue SenderKeyDistributionSendJob                     │  │
│  │   SenderKeyDistributionSendJob job =                          │  │
│  │     new SenderKeyDistributionSendJob(newMemberId, threadId);  │  │
│  │   jobManager.add(job);                                        │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ // This sends SKDM to new member's devices                   │  │
│  │ Reference: SenderKeyDistributionSendJob.java:50-59            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 3: New Member Creates Session if Needed                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // When SKDM is received, new member processes it:           │  │
│  │ SignalGroupSessionBuilder.process(sender, senderKeyDistMsg);  │  │
│  │                                                               │  │
│  │ // This establishes sender key session locally               │  │
│  │ // File: lib/.../crypto/SignalGroupSessionBuilder.java:24-28  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 Member Leave Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│          MEMBER LEAVE FLOW                                           │
│          File: app/.../groups/GroupManagerV2.java                    │
│          Lines: (leaveGroup method)                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  TRIGGER: Member removed or leaves group                           │
│                                                                     │
│  STEP 1: Update Group State                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Remove member from local group state                       │  │
│  │ SignalDatabase.groups().updateGroup(groupId, newGroupState);  │  │
│  │                                                               │  │
│  │ Reference: GroupsV2StateProcessor.kt (handle leave)           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: Rotate Sender Key (Critical for Security)                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Leaving member should not be able to decrypt future msgs  │  │
│  │ // Rotate our sender key for this group                       │  │
│  │ SenderKeyUtil.rotateOurKey(distributionId);                   │  │
│  │                                                               │  │
│  │ // rotateOurKey implementation:                               │  │
│  │ public static void rotateOurKey(DistributionId distributionId) {│  │
│  │   // Delete our sender key session                           │  │
│  │   AppDependencies.getProtocolStore().aci().senderKeys()       │  │
│  │     .deleteAllFor(aci.toString(), distributionId);            │  │
│  │                                                               │  │
│  │   // Clear shared status (will need to resend to everyone)   │  │
│  │   SignalDatabase.senderKeyShared()                            │  │
│  │     .deleteAllFor(distributionId);                            │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SenderKeyUtil.java:18-23                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 3: Next Group Send Distributes New Key                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // On next message send:                                      │  │
│  │ // - Sender key deleted, so new one is created               │  │
│  │ // - Shared status cleared, so SKDM sent to all members      │  │
│  │ // - New key excludes leaving member                         │  │
│  │                                                               │  │
│  │ // Automatic in sendGroupMessage():                           │  │
│  │ // sharedWith is empty, so all current members get SKDM      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.4 Sender Key Rotation

```
┌─────────────────────────────────────────────────────────────────────┐
│          SENDER KEY ROTATION                                         │
│          File: app/.../crypto/SenderKeyUtil.java                     │
│          Lines: 18-23                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  WHEN TO ROTATE:                                                    │
│  - Member leaves group (security)                                   │
│  - Member removed from group (security)                             │
│  - Sender key age exceeds threshold (recommended ~7 days)          │
│  - Identity key change detected                                     │
│                                                                     │
│  ROTATION CODE:                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ public static void rotateOurKey(@NonNull DistributionId distributionId) {│
│  │   try (SignalSessionLock.Lock unused = ReentrantSessionLock.INSTANCE.acquire()) {│
│  │     // Delete our sender key session                         │  │
│  │     // Next encryption will create new key                   │  │
│  │     AppDependencies.getProtocolStore().aci().senderKeys()    │  │
│  │       .deleteAllFor(                                          │  │
│  │         SignalStore.account().requireAci().toString(),       │  │
│  │         distributionId                                        │  │
│  │       );                                                      │  │
│  │                                                               │  │
│  │     // Clear shared status                                    │  │
│  │     // Forces SKDM send to all members on next message       │  │
│  │     SignalDatabase.senderKeyShared()                          │  │
│  │       .deleteAllFor(distributionId);                          │  │
│  │   }                                                           │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SenderKeyUtil.java:18-23                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  AUTOMATIC ROTATION ON AGE:                                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // In GroupSendUtil.sendMessage():                            │  │
│  │ long keyCreateTime = SenderKeyUtil.getCreateTimeForOurKey(distributionId);│
│  │ long keyAge = System.currentTimeMillis() - keyCreateTime;     │  │
│  │                                                               │  │
│  │ if (keyCreateTime != -1 && keyAge > RemoteConfig.senderKeyMaxAge()) {│
│  │   Log.w(TAG, "Rotating sender key due to age");               │  │
│  │   SenderKeyUtil.rotateOurKey(distributionId);                 │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: GroupSendUtil.java:359-365                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Identity Management

### 6.1 Identity Key Storage

```
┌─────────────────────────────────────────────────────────────────────┐
│          IDENTITY KEY STORAGE                                        │
│          File: app/.../crypto/storage/SignalBaseIdentityKeyStore.java│
│          Lines: 66-116                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  IdentityTable Schema:                                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Columns:                                                      │  │
│  │   - address: TEXT (recipient's ServiceId)                    │  │
│  │   - identity_key: BLOB (IdentityKey serialized)              │  │
│  │   - verified: INTEGER (verified status)                      │  │
│  │   - first_use: INTEGER (boolean, first time seeing this key) │  │
│  │   - timestamp: INTEGER (when identity was saved)             │  │
│  │   - nonblocking_approval: INTEGER                            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Saving Identity:                                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ public SaveResult saveIdentity(SignalProtocolAddress address, │  │
│  │                                 IdentityKey identityKey,      │  │
│  │                                 boolean nonBlockingApproval) { │  │
│  │                                                               │  │
│  │   IdentityStoreRecord existingRecord = cache.get(address.getName());│
│  │                                                               │  │
│  │   if (existingRecord == null) {                               │  │
│  │     // NEW identity - first time seeing this recipient       │  │
│  │     cache.save(address.getName(), recipientId, identityKey,  │  │
│  │                 VerifiedStatus.DEFAULT, true, timestamp, nonBlockingApproval);│
│  │     return SaveResult.NEW;                                    │  │
│  │   }                                                           │  │
│  │                                                               │  │
│  │   if (!existingRecord.getIdentityKey().equals(identityKey)) { │  │
│  │     // UPDATE - identity changed!                             │  │
│  │     // This triggers safety number change                     │  │
│  │                                                               │  │
│  │     // Reset verified status if was verified                 │  │
│  │     VerifiedStatus newStatus = existingRecord.getVerifiedStatus()│
│  │       == VerifiedStatus.VERIFIED ? VerifiedStatus.UNVERIFIED │  │
│  │       : VerifiedStatus.DEFAULT;                               │  │
│  │                                                               │  │
│  │     cache.save(address.getName(), recipientId, identityKey,  │  │
│  │                 newStatus, false, timestamp, nonBlockingApproval);│
│  │                                                               │  │
│  │     // CRITICAL: Archive sessions and clear sender keys      │  │
│  │     IdentityUtil.markIdentityUpdate(context, recipientId);    │  │
│  │     AppDependencies.getProtocolStore().aci().sessions()       │  │
│  │       .archiveSiblingSessions(address);                       │  │
│  │     SignalDatabase.senderKeyShared().deleteAllFor(recipientId);│  │
│  │                                                               │  │
│  │     return SaveResult.UPDATE;                                 │  │
│  │   }                                                           │  │
│  │                                                               │  │
│  │   return SaveResult.NO_CHANGE;                                │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SignalBaseIdentityKeyStore.java:74-116             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Identity Change Handling

```
┌─────────────────────────────────────────────────────────────────────┐
│          IDENTITY CHANGE HANDLING                                    │
│          File: app/.../crypto/storage/SignalBaseIdentityKeyStore.java│
│          Lines: 85-106                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  WHEN IDENTITY CHANGES:                                             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 1. Log the change                                             │  │
│  │    Log.i(TAG, "Replacing existing identity for " + address);  │  │
│  │                                                               │  │
│  │ 2. Reset verified status                                      │  │
│  │    // If was VERIFIED -> becomes UNVERIFIED                  │  │
│  │    // If was DEFAULT -> stays DEFAULT                         │  │
│  │                                                               │  │
│  │ 3. Mark identity update in conversation                      │  │
│  │    IdentityUtil.markIdentityUpdate(context, recipientId);     │  │
│  │    // Inserts "Safety number changed" message in chat        │  │
│  │                                                               │  │
│  │ 4. Archive existing sessions                                  │  │
│  │    sessions.archiveSiblingSessions(address);                  │  │
│  │    // Old sessions won't work with new identity              │  │
│  │                                                               │  │
│  │ 5. Clear sender key shared status                             │  │
│  │    senderKeyShared.deleteAllFor(recipientId);                 │  │
│  │    // Will need to re-share sender keys on next send         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  TRUST CHECK FOR SENDING:                                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ public boolean isTrustedForSending(IdentityKey identityKey,   │  │
│  │                                     IdentityStoreRecord record) {│
│  │   if (record == null) {                                       │  │
│  │     // Trust on first use                                     │  │
│  │     return true;                                              │  │
│  │   }                                                           │  │
│  │                                                               │  │
│  │   // Keys must match                                          │  │
│  │   if (!identityKey.equals(record.getIdentityKey())) {         │  │
│  │     Log.w(TAG, "Identity keys don't match");                  │  │
│  │     return false;                                             │  │
│  │   }                                                           │  │
│  │                                                               │  │
│  │   // Must not be UNVERIFIED                                   │  │
│  │   if (record.getVerifiedStatus() == VerifiedStatus.UNVERIFIED) {│  │
│  │     Log.w(TAG, "Needs unverified approval!");                 │  │
│  │     return false;                                             │  │
│  │   }                                                           │  │
│  │                                                               │  │
│  │   // Non-blocking approval check (within 5 seconds)           │  │
│  │   if (isNonBlockingApprovalRequired(record)) {                │  │
│  │     Log.w(TAG, "Needs non-blocking approval!");               │  │
│  │     return false;                                             │  │
│  │   }                                                           │  │
│  │                                                               │  │
│  │   return true;                                                │  │
│  │ }                                                             │  │
│  │                                                               │  │
│  │ Reference: SignalBaseIdentityKeyStore.java:234-256            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 SignalProtocolAddress Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│          SignalProtocolAddress STRUCTURE                             │
│          (from libsignal)                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SignalProtocolAddress {                                            │
│    name: String,      // ServiceId (ACI or PNI) as string          │
│    deviceId: int      // Device identifier (1 = primary)           │
│  }                                                                  │
│                                                                     │
│  Examples:                                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Alice's primary device                                     │  │
│  │ SignalProtocolAddress("alice-aci-uuid", 1)                    │  │
│  │                                                               │  │
│  │ // Alice's linked tablet                                      │  │
│  │ SignalProtocolAddress("alice-aci-uuid", 2)                    │  │
│  │                                                               │  │
│  │ // Bob's primary device                                       │  │
│  │ SignalProtocolAddress("bob-aci-uuid", 1)                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Usage in code:                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Create address for recipient's device                      │  │
│  │ SignalProtocolAddress address =                               │  │
│  │   recipient.requireServiceId().toProtocolAddress(deviceId);   │  │
│  │                                                               │  │
│  │ // Check session exists                                       │  │
│  │ boolean hasSession = aciStore.containsSession(address);       │  │
│  │                                                               │  │
│  │ // Load session                                               │  │
│  │ SessionRecord session = aciStore.loadSession(address);        │  │
│  │                                                               │  │
│  │ // Store session                                              │  │
│  │ aciStore.storeSession(address, sessionRecord);                │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Complete Flow Diagrams

### 7.1 Full Session + Sender Key Flow (New Group Message)

```
┌─────────────────────────────────────────────────────────────────────┐
│          COMPLETE FLOW: NEW GROUP MESSAGE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  USER ACTION: Send message to group                                 │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. GroupSendUtil.sendResendableDataMessage()                 │   │
│  │    File: app/.../messages/GroupSendUtil.java:99-118          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 2. Get DistributionId for group                               │   │
│  │    DistributionId distributionId =                            │   │
│  │      SignalDatabase.groups().getOrCreateDistributionId(groupId);│  │
│  │    File: GroupSendUtil.java:115                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 3. Check Sender Key Age, Rotate if Needed                     │   │
│  │    if (keyAge > maxAge) SenderKeyUtil.rotateOurKey(distId);   │   │
│  │    File: GroupSendUtil.java:359-365                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 4. Determine Sender Key Targets vs Legacy Targets             │   │
│  │    - Check group send endorsements                           │   │
│  │    - Check if sender key already shared                      │   │
│  │    File: GroupSendUtil.java:310-349                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 5. For recipients missing sender key:                         │   │
│  │    a. Check/establish sessions for each device               │   │
│  │       SignalServiceMessageSender.getEncryptedMessage()       │   │
│  │       - Fetch prekeys if no session                          │   │
│  │       - X3DH key agreement                                   │   │
│  │    b. Create SenderKeyDistributionMessage                    │   │
│  │       getOrCreateNewGroupSession(distributionId)             │   │
│  │    c. Send SKDM to all devices of recipient                  │   │
│  │       sendSenderKeyDistributionMessage()                     │   │
│  │    d. Mark sender key as shared                              │   │
│  │       markSenderKeySharedWith(distributionId, addresses)     │   │
│  │    File: SignalServiceMessageSender.java:2480-2520           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 6. Encrypt Message with Sender Key                            │   │
│  │    cipher.encryptForGroup(distributionId, destinations, ...)  │   │
│  │    File: SignalServiceMessageSender.java:2550-2554            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 7. Send Encrypted Group Message                               │   │
│  │    messageApi.sendGroupMessage(ciphertext, ...)               │   │
│  │    File: SignalServiceMessageSender.java:2564                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 8. Sync to Linked Devices (if multi-device)                   │   │
│  │    sendSyncMessage() with transcript                          │   │
│  │    File: SignalServiceMessageSender.java:589-594              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Flow: New Device Joins Existing Group

```
┌─────────────────────────────────────────────────────────────────────┐
│          FLOW: NEW DEVICE JOINS EXISTING GROUP                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SCENARIO: Alice has primary device, links new tablet              │
│            Alice is member of Group X                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. Device Linking Complete                                    │   │
│  │    - New tablet has same identity key                        │   │
│  │    - New tablet has deviceId = 2                             │   │
│  │    File: LinkDeviceRepository.kt                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 2. Storage Service Sync                                       │   │
│  │    - New device downloads group state                        │   │
│  │    - Group membership synced                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 3. New Device Sends First Message to Group                    │   │
│  │    - No existing sessions with other members                 │   │
│  │    - No sender key for group                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 4. Session Establishment for Each Recipient Device            │   │
│  │    for each recipient in group:                               │   │
│  │      for each device of recipient:                           │   │
│  │        if (!hasSession):                                     │   │
│  │          fetchPreKeys() -> X3DH -> storeSession()            │   │
│  │    File: SignalServiceMessageSender.java:2841-2855           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 5. Sender Key Distribution                                    │   │
│  │    - Create SenderKeyDistributionMessage                     │   │
│  │    - Send SKDM to all recipients                             │   │
│  │    - Store in SenderKeyTable                                 │   │
│  │    - Mark shared in SenderKeySharedTable                     │   │
│  │    File: SignalServiceMessageSender.java:2493-2517           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 6. Message Encrypted and Sent                                 │   │
│  │    - Single encryption using sender key                      │   │
│  │    - Delivered to all recipients                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.3 Flow: Member Leaves Group

```
┌─────────────────────────────────────────────────────────────────────┐
│          FLOW: MEMBER LEAVES GROUP                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SCENARIO: Bob leaves Group X                                       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. Leave Request Processed                                    │   │
│  │    - GroupsV2StateProcessor receives change                  │   │
│  │    - GroupManagerV2.commitLeave()                            │   │
│  │    File: GroupsV2StateProcessor.kt, GroupManagerV2.java       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 2. Update Local Group State                                   │   │
│  │    - Remove Bob from member list                             │   │
│  │    - Update group revision                                   │   │
│  │    File: GroupsV2StateProcessor.kt                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 3. Rotate Sender Key (CRITICAL)                               │   │
│  │    SenderKeyUtil.rotateOurKey(distributionId);                │   │
│  │    - Delete our sender key session                           │   │
│  │    - Clear SenderKeySharedTable for this group               │   │
│  │    File: SenderKeyUtil.java:18-23                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 4. Next Group Message Send                                    │   │
│  │    - New sender key created (old one deleted)                │   │
│  │    - sharedWith is empty, so SKDM sent to ALL members        │   │
│  │    - Bob excluded from member list                           │   │
│  │    - Bob cannot decrypt future messages                      │   │
│  │    File: SignalServiceMessageSender.java:2484-2517           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  SECURITY GUARANTEE: Bob cannot decrypt any future group messages   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Files Reference

| File | Purpose | Key Lines |
|------|---------|-----------|
| `SignalServiceMessageSender.java` | Main message sending logic | 500-600 (sender key), 2480-2613 (group send), 2831-2870 (session) |
| `SignalGroupSessionBuilder.java` | Create SKDM | 30-34 |
| `SignalGroupCipher.java` | Group encryption/decryption | 26-38 |
| `SenderKeyDistributionSendJob.java` | Dedicated SKDM send | 81-139 |
| `SenderKeyUtil.java` | Sender key rotation | 18-23 |
| `SenderKeyTable.kt` | Sender key storage | 44-69 |
| `SenderKeySharedTable.kt` | Track shared status | 47-76 |
| `SessionTable.kt` | Session storage | 44-56 |
| `SignalBaseIdentityKeyStore.java` | Identity management | 74-116, 234-256 |
| `GroupsV2StateProcessor.kt` | Group state changes | 165-200 |
| `GroupSendUtil.java` | Group send orchestration | 247-544 |
| `LinkDeviceRepository.kt` | Device linking | 52-151 |

---

## Summary

### Session Establishment
- Triggered when sending to recipient with no existing session
- Uses X3DH key agreement with prekey bundle
- One session per (recipient, device) pair stored in SessionTable

### Sender Key Distribution
- Per-sender, per-group sender key session
- DistributionId uniquely identifies group's sender key
- SKDM sent to recipients who don't have the key
- SenderKeySharedTable tracks who has received the key

### Multi-Device Handling
- Each device has unique deviceId (1 = primary, 2+ = linked)
- Sessions established independently for each device
- Sender keys shared with each device individually
- Device linking uses provisioning cipher to transfer identity keys

### Group Membership Changes
- Member join: Send SKDM to new member
- Member leave: Rotate sender key (delete + clear shared status)
- Identity change: Archive sessions, clear sender key shared status

### Identity Management
- IdentityKey stored per recipient in IdentityTable
- Identity change detection triggers session archival
- Safety number verification for changed identities
- Trust model: Trust on First Use (TOFU) with verification options