# 📚 Signal Chat POC: Complete Use Case Flows (Technical Deep Dive)

This document contains **detailed technical sequence diagrams** for every requested use case.
Each diagram visualizes the interaction between:
*   **User**
*   **Client DB** (Local SQLCipher)
*   **Client E2EE** (Signal Protocol Engine)
*   **Server** (API/Firestore)
*   **Server DB** (Cloud Storage)

---

## 🏗️ 1. User Management

### 1.1 User Gets Registered
**Simple explanation:** When a user first installs the app, the app creates secret security keys on their phone. These keys are used to lock and unlock messages. The "public" part of these keys is sent to the server so friends can find and start a safe chat with the user.

```mermaid
sequenceDiagram
    participant User
    participant CDB as 💾 Client DB
    participant E2EE as 🔐 Client E2EE
    participant Server as ☁️ Server API
    participant SDB as 🗄️ Server DB

    User->>CDB: 1. Input Phone & Basic Info
    CDB->>E2EE: 2. Generate Identity Key Pair
    CDB->>E2EE: 3. Generate Registration ID
    CDB->>E2EE: 4. Generate PreKeys (0-100) & Initial SignedPreKey
    
    E2EE->>CDB: 5. Store Private Keys (Encrypted)
    
    CDB->>Server: 6. Upload Public Key Bundle
    Server->>SDB: 7. Store Bundle inside /keys/{userId}
    SDB-->>Server: Success
    Server-->>User: 8. Registration Complete (200 OK)
```

### 1.2 User Updates Account Info
**Simple explanation:** When a user changes their name or profile picture, the app updates the info on the phone and also sends it to the server. This way, other users can see the updated profile.

```mermaid
sequenceDiagram
    participant User
    participant CDB as 💾 Client DB
    participant Server as ☁️ Server
    participant SDB as 🗄️ Server DB

    User->>CDB: 1. Change Name to "Alice Pro"
    CDB->>CDB: 2. Update Local User Table
    CDB->>Server: 3. POST /profile {name: "Alice Pro"}
    Server->>SDB: 4. Update /users/{id} Document
    SDB-->>Server: Ack
    Server-->>User: 5. Profile Updated
```

### 1.3 User Looks for Friends Among Contacts
**Simple explanation:** The app looks at the phone numbers in the contact list. It turns these numbers into hidden codes and sends them to the server. The server checks which of these people are already using the app and tells your phone, so you can start chatting.

```mermaid
sequenceDiagram
    participant User
    participant Phone as 📱 Contact Book
    participant CDB as 💾 Client DB
    participant Server as ☁️ Server
    participant SDB as 🗄️ Server DB

    User->>Phone: 1. "Sync Contacts"
    Phone-->>CDB: 2. Return Raw List (+123, +456...)
    CDB->>CDB: 3. Hash Numbers (SHA256)
    
    CDB->>Server: 4. Send List of Hashes
    Server->>SDB: 5. Query matching User Hashes
    SDB-->>Server: Return Matches (UserIDs)
    
    Server-->>CDB: 6. Return Registered User List
    CDB->>CDB: 7. Save to 'Contacts' Table
    CDB-->>User: 8. Show Available Friends
```

---

## 📨 2. Peer-to-Peer Messaging (1-on-1)

### 2.1 User A Sends Message to User B (First Time)
**Simple explanation:** To start a chat, User A's app gets User B's public keys from the server. Using these keys, the app sets up a secure connection and locks the message so only User B can read it.

```mermaid
sequenceDiagram
    participant UA as User A
    participant CA_DB as 💾 Client A DB
    participant CA_E2E as 🔐 Client A E2E
    participant Server as ☁️ Server
    participant SDB as 🗄️ Server DB
    participant CB_E2E as 🔐 Client B E2E
    participant CB_DB as 💾 Client B DB
    participant UB as User B

    UA->>CA_DB: 1. "Hi B"
    CA_DB->>CA_DB: 2. Check Session? (No)
    
    CA_DB->>Server: 3. Get PreKey Bundle (B)
    Server->>SDB: 4. Fetch Bundle
    SDB-->>Server: Return Bundle
    Server-->>CA_E2E: 5. Receive {IK_B, SPK_B, OPK_B}
    
    CA_E2E->>CA_E2E: 6. X3DH Handshake (Derive Root Key)
    CA_E2E->>CA_DB: 7. Save New Session
    CA_E2E->>CA_E2E: 8. Encrypt "Hi B" (PreKeyMessage)
    
    CA_E2E->>Server: 9. Send Message
    Server->>SDB: 10. Store in B's Inbox
    
    Note over Server, UB: Delivery depends on Online Status (See 2.3)
```

### 2.2 User A Sends Message to Known User B (Ongoing)
**Simple explanation:** Once the chat has started, the app uses a different key for every single message. This makes the conversation extremely secure. The app locks the message, sends it, and prepares a new key for the next one.

```mermaid
sequenceDiagram
    participant UA as User A
    participant CA_DB as 💾 Client A DB
    participant CA_E2E as 🔐 Client A E2E
    participant Server as ☁️ Server
    
    UA->>CA_DB: 1. "How are you?"
    CA_DB->>CA_E2E: 2. Load Session (B)
    CA_E2E->>CA_E2E: 3. Ratchet Forward -> New Message Key
    CA_E2E->>CA_E2E: 4. Encrypt (SignalMessage)
    CA_E2E->>Server: 5. Send Message
    CA_DB->>CA_DB: 6. Update Session State (Ratchet moved)
```

### 2.3 Online vs Offline Client Handling (The Inbox)
**Simple explanation:** If the receiver is online, the message is delivered immediately. If they are offline, the server saves the message in a temporary 'Inbox'. As soon as the receiver opens the app, they receive all saved messages, and the server deletes them forever.

```mermaid
sequenceDiagram
    participant Server as ☁️ Server
    participant SDB as 🗄️ Server DB
    participant CB_Net as 📡 Client B Network
    participant CB_DB as 💾 Client B DB

    Note over Server: Message Arrives for B
    Server->>SDB: 1. Persist to /inbox/b/messages/{id}
    
    alt B is Online
        Server->>CB_Net: 2. Stream Event
        CB_Net->>SDB: 3. Fetch Message Payload
        CB_Net->>CB_DB: 4. Process & Decrypt
        CB_Net->>SDB: 5. Ack (Delete Verified)
        SDB-->>Server: Message Removed
    else B is Offline
        Note over SDB: Message stays in DB
        Note over CB_Net: ...Time Passes...
        CB_Net->>Server: 6. User B Comes Online (Connect)
        Server->>CB_Net: 7. "You have 5 pending messages"
        CB_Net->>SDB: 8. Fetch Batch
        CB_Net->>CB_DB: 9. Decrypt All
        CB_Net->>SDB: 10. Delete All from Server
    end
```

### 2.4 User A Deletes Chat with User B
**Simple explanation:** Deleting a chat only removes it from your own phone. It does not delete anything from the other person's phone or the server. This is a local action for your own privacy.

```mermaid
sequenceDiagram
    participant UA as User A
    participant CA_DB as 💾 Client A DB
    
    UA->>CA_DB: 1. Delete Chat "Bob"
    CA_DB->>CA_DB: 2. DELETE FROM messages WHERE chat_id=Bob
    CA_DB->>CA_DB: 3. DELETE FROM sessions WHERE id=Bob
    Note over UA: No request sent to Server or Bob.
    Note over UA: Deletion is local only.
```

### 2.5 User B Denies Chat Deletion
**Simple explanation:** If you try to delete a message for everyone, the other person's app receives a request. If they choose to deny it, the message stays on their phone.

```mermaid
sequenceDiagram
    participant UA as User A
    participant Server as Server
    participant UB as User B
    
    UA->>Server: 1. Send "DeleteRequest" for MsgID: 123
    Server->>UB: 2. Deliver "DeleteRequest"
    UB->>UB: 3. "User A wants to delete msg 123. Allow?"
    UB->>Server: 4. "No" (Deny)
    Note over UB: Message remains on B's device.
```

---

## 👥 3. Group Operations

### 3.1 User Creates a Group
**Simple explanation:** When you create a group and add friends, the server makes a record of the group. Your app then sends a special group-key to each member individually so everyone can talk to each other securely.

```mermaid
sequenceDiagram
    participant User
    participant CDB as 💾 Client DB
    participant E2EE as 🔐 Client E2EE
    participant Server as ☁️ Server
    participant SDB as 🗄️ Server DB

    User->>CDB: 1. Create Group "Team" (Add Bob, Charlie)
    CDB->>Server: 2. POST /groups/create {members: [A,B,C]}
    Server->>SDB: 3. Create Group Document
    SDB-->>Server: Return GroupID
    
    Note over E2EE: Sender Key Setup
    E2EE->>E2EE: 4. Generate 'SenderKey' Chain
    E2EE->>E2EE: 5. Encrypt SenderKey for Bob (1-to-1)
    E2EE->>E2EE: 6. Encrypt SenderKey for Charlie (1-to-1)
    
    E2EE->>Server: 7. Send 'Distribution Messages'
    Server->>SDB: 8. Route to B and C Inboxes
```

### 3.2 User Sends Message to Group
**Simple explanation:** When you send a group message, your app locks it once using your group-key. The server then sends this locked message to everyone in the group. Each member uses your key to unlock and read it.

```mermaid
sequenceDiagram
    participant User
    participant E2EE as 🔐 Client E2EE
    participant Server as ☁️ Server
    participant Group as 👥 Group Members

    User->>E2EE: 1. "Hello Team"
    E2EE->>E2EE: 2. Load My Sender Key
    E2EE->>E2EE: 3. Encrypt ONCE -> Ciphertext
    
    E2EE->>Server: 4. Send {GroupId, Ciphertext}
    Server->>Group: 5. Fan-out to Bob & Charlie
    
    Note over Group: Bob uses A's Sender Key to decrypt.
    Note over Group: Charlie uses A's Sender Key to decrypt.
```

### 3.3 User Leaves Group
**Simple explanation:** When a user leaves a group, the server removes them from the member list. A system message is sent to the group to let everyone know that the user has left.

```mermaid
sequenceDiagram
    participant User
    participant Server as ☁️ Server
    participant SDB as 🗄️ Server DB

    User->>Server: 1. Leave Group {GroupId}
    Server->>SDB: 2. Remove UserID from 'members' array
    Server->>Server: 3. System Msg: "Alice left" -> Group Inbox
```

---

## 🛡️ 4. Administration & Moderation

### 4.1 User Reports User B
**Simple explanation:** If a user is bothersome, you can report them. Your app sends a report to the admins. You can also choose to attach a few recent messages so the admins can see what happened.

```mermaid
sequenceDiagram
    participant User
    participant App as 📱 Client App
    participant Server as ☁️ Server
    participant AdminDB as 🗄️ Admin DB

    User->>App: 1. Report User B (Spam)
    App->>App: 2. Attach Last 5 Messages (Optional)
    App->>Server: 3. POST /reports {target: B, reason: Spam}
    Server->>AdminDB: 4. Create Ticket
    Note over AdminDB: Admins review ticket.
```

### 4.2 Administrator Bans User
**Simple explanation:** An admin can block a user if they break rules. The server disables their account, and the user will see a "Suspended" message on their phone.

```mermaid
sequenceDiagram
    participant Admin
    participant Server as ☁️ Server
    participant SDB as 🗄️ Server DB
    participant Target as 📱 Banned User

    Admin->>Server: 1. Ban User B
    Server->>SDB: 2. Set user status = 'BANNED'
    Server->>SDB: 3. Revoke Auth Tokens
    
    Target->>Server: 4. Try Connect / Send
    Server-->>Target: 5. 403 Forbidden (Banned)
    Target->>Target: 6. Show "Account Suspended"
```

### 4.3 Administrator Broadcasts Message
**Simple explanation:** Admins can send an announcement to every single user at once (like for maintenance). These messages are sent in plain text, not encrypted, as they are public system messages.

```mermaid
sequenceDiagram
    participant Admin
    participant Server as ☁️ Server
    participant AllUsers as 👥 All Users
    
    Admin->>Server: 1. Send Broadcast "Maintenance at 2PM"
    Server->>Server: 2. Iterate All Active Users
    Server->>AllUsers: 3. Stream Message (System Channel)
    Note over AllUsers: Not E2E Encrypted (Plaintext System Msg)
```

### 4.4 Admin Queries DB (Stats)
**Simple explanation:** Admins can check reports inside the system, such as seeing which users have been offline for a long time or how many people are using the app.

```mermaid
sequenceDiagram
    participant Admin
    participant Server as ☁️ Server
    participant SDB as 🗄️ Server DB

    Admin->>Server: 1. Query: "Users offline > 30 days"
    Server->>SDB: 2. SELECT * FROM users WHERE last_seen < NOW-30d
    SDB-->>Server: 3. Return List
    Server-->>Admin: 4. Show Report
```

---

## 🏷️ 5. Special Features (Tags & "//" Entries)

### 5.1 User Tags User B (@mention)
**Simple explanation:** When you type @ followed by a name, the app finds that person's ID. It adds a tag to the message so the mentioned person knows they were called out in the chat.

```mermaid
sequenceDiagram
    participant User
    participant App as 📱 App
    participant E2EE as 🔐 E2EE

    User->>App: 1. Type "Hello @Bob"
    App->>App: 2. Detect Tag -> Resolve Bob's UUID
    App->>E2EE: 3. Encrypt Body: "Hello @{uuid-of-bob}"
    App->>App: 4. Add Metadata: mentions=[BobUUID]
    
    Note over E2EE: Metadata is usually plaintext header
    Note over E2EE: OR encrypted inside signal envelope (secure).
    
    E2EE->>Server: 5. Send Message
```

### 5.2 User Queries All Users with Specific Tag
**Simple explanation:** You can search for people using tags, like searching for "#Doctor". The server looks through everyone's profile and shows you a list of people who have that tag.

```mermaid
sequenceDiagram
    participant User
    participant Server as ☁️ Server
    participant SDB as 🗄️ Server DB

    User->>Server: 1. Search Users with Tag "#Doctor"
    Server->>SDB: 2. Query Index: users_by_tag
    SDB-->>Server: 3. Return Matching Profiles
    Server-->>User: 4. List of Doctors
```

### 5.3 User Enters New "//" Entry (Slash Command / Note)
**Simple explanation:** If you type something starting with "//", like "//todo", the app saves it as a quick note on your phone. This information stays private on your device.

```mermaid
sequenceDiagram
    participant User
    participant CDB as 💾 Client DB
    participant Server as ☁️ Server (Sync)

    User->>CDB: 1. Save "//todo Buy Milk"
    CDB->>CDB: 2. Parse Trigger "//"
    CDB->>CDB: 3. Save to 'Notes' Table locally
    
    Note over CDB: Optionally sync to self-devices
    CDB->>Server: 4. Send Sync Message (to Self)
```

---

## 🛠️ Developer Implementation Guide

This section is for developers to understand the local storage and data structures used in the app.

### 🗄️ 1. Frontend Database Schema (SQLCipher)
The mobile app uses an encrypted SQL database (SQLCipher) to store keys and messages. There are **6 main tables**:

1.  **`identity`**: Stores your own secret identity keys.
2.  **`prekeys`**: Stores one-time keys used to start new chats.
3.  **`signed_prekeys`**: Stores medium-term security keys.
4.  **`sessions`**: Stores the current "state" of your 1-on-1 chats (who you are talking to and which key is next).
5.  **`sender_keys`**: Stores group chat keys for every group member.
6.  **`messages`**: Stores the actual chat history (decrypted text, image paths, etc.).

#### 📝 SQL Table Example (`messages`)
```sql
CREATE TABLE messages (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    conversation_id  TEXT,    -- User ID or Group ID
    sender_id        TEXT,
    body             TEXT,    -- Decrypted message text
    type             INTEGER, -- 1=Text, 2=Image, 3=System
    status           INTEGER, -- 0=Sending, 1=Sent, 2=Read
    timestamp        INTEGER,
    is_me            BOOLEAN, -- 1 if sent by you
    attachment_path  TEXT     -- Path to local image/video file
);
```

---

### 📋 2. Common Data Models (JSON Examples)
These are the data formats used when sending/receiving information from the server.

#### 👤 2.1 User Profile Discovery
*When searching for a friend via their phone hash.*
```json
{
  "userId": "user_abc_123",
  "displayName": "Alice Smith",
  "status": "Available",
  "avatarUrl": "https://server.com/avatars/alice.jpg",
  "isRegistered": true
}
```

#### 🔐 2.2 Key Bundle (Security Setup)
*Fetched when starting a new chat for the first time.*
```json
{
  "registrationId": 54321,
  "identityKey": "base64_public_identity_key...",
  "signedPreKey": {
    "id": 1,
    "key": "base64_signed_prekey...",
    "signature": "base64_signature..."
  },
  "oneTimePreKey": "base64_single_use_key..."
}
```

#### 📩 2.3 Message Envelope (The Locked Box)
*The format used to send any encrypted message.*
```json
{
  "senderId": "alice_uid",
  "senderDeviceId": 1,
  "messageType": 3, 
  "ciphertext": "base64_encrypted_data_blob...",
  "timestamp": 1706500000000
}
```

#### 🖼️ 2.4 Media Pointer (Attachments)
*Included inside an encrypted message for photos/videos.*
```json
{
  "type": "image",
  "downloadUrl": "https://storage.com/files/img_99.bin",
  "decryptionKey": "base64_random_aes_key...",
  "iv": "base64_initialization_vector...",
  "fileName": "photo.jpg",
  "fileSize": 1048576
}
```

#### 🛡️ 2.5 Admin Report
*When reporting a user for bad behavior.*
```json
{
  "reporterId": "user_123",
  "targetId": "bad_user_456",
  "reason": "SPAM",
  "evidenceMessages": ["MsgID_1", "MsgID_2"]
}
```

---

> This guide provides everything a developer needs to implement the flows described in the diagrams above: the database structures to save data and the JSON formats to talk to the server.
