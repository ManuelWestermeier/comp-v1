# Networking Protocol V1

This is a multigroup encrypted **Networking Protocol** designed for communication using 433 MHz transmitters and receivers.

### Overview

- **Network Hierarchy**
- **Signing**
- **Signal States**: The connection can either be pulled `HIGH` (active) or remain `LOW` (idle).
- **Transmission Timing**:
  - Each bit is sent every 1000 microseconds.
  - Bits are grouped into bytes, which are further organized into packets.
- **Packet Structure**:
  - Each packet begins with a `[HIGH]` signal to mark its start.
  - The packet data contains binary representations of various fields.
  - Packets end with a `[LOW]` signal, indicating the next packet's beginning.

# Network Hierarchy

The **NETWORK** is the physical connection established using 433 MHz RF modules.

The **GROUPs** are virtual networks that implement encryption for secure communication.  
Users within a GROUP can handle the connection process or create up to **65,536 GROUPs**.
Each GROUP can have up to **65,536 users**.
Each user is a member of one or more GROUPs and can connect to multiple networks simultaneously.

Users can perform the following actions:

- Send encrypted messages to other users within the GROUP.
- Broadcast messages to all members of the GROUP.
- Broadcast messages to the entire NETWORK.

## Hierarchy Overview

```plaintext
NETWORK
  ├── GROUP 1
  │     ├── USER 1
  │     ├── USER 2
  │     └── USER 3
            ...
  ├── GROUP 2
  │     ├── USER 27224
  │     └── USER 2885
            ...
  └── GROUP 3
        ├── USER 6
        ├── USER 7
        ├── USER 8
        └── USER 9
            ...
  ...
```

This hierarchy ensures a structured organization of users within secure and scalable virtual GROUPs, supported by a robust physical NETWORK.

# Signing

1. **Random Value Generation**  
   The user generates a random 4-byte value, referred to as `SIGN_VALUE`.

2. **Hashing the Value**  
   The user hashes `SIGN_VALUE` and extracts the last 4 bytes of the hash, referred to as `SIGN_VALUE_HASH`.

3. **Sharing the Hash**  
   The value of `SIGN_VALUE_HASH` is shared with all members of the group.

4. **Signing a Message**  
   To sign a message, the user sends:

   - The original random value (`SIGN_VALUE`)
   - The next hashed random value (`NEXT_SIGN_VALUE_HASH`) to identify the next message (it have to be send in the same pocket)

This ensures that each message is uniquely identifiable and establishes the hash for the following message.
The format is (` ... [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] ... `).


# Packet Transmission Rules

1.  **Start Conditions**:

    - To send a packet, the sender ensures the connection remains `LOW` for a specified _max send time_.
    - If the connection goes `HIGH` during this period, the sender must retry.

2.  **Collision Avoidance**:

    - Devices use the time since the last packet's transmission to wait for a random interval between 1000 and 50000 microseconds before attempting to pull the connection `HIGH`.
    - If the line stays `LOW`, the sender may proceed to transmit a packet.

# Packet Format

Each packet follows a structured format:

- `[HIGH]`: Marks the start of the packet.
- `[FUNCTION=x|1B]`: A fixed 1-byte field defining the packet's purpose.
- Additional fields are specified in a `NAME=VALUE|LENGTH` format:
  - **NAME**: Field name.
  - **VALUE**: Field value or its default value.
  - **LENGTH**: Field size, expressed as `xB` (bytes) or `xBit` (bits).
  - Encrypted values are denoted as `VALUE x (PASSWORD + SALT)`.
  - Fields may consist of concatenated chunks.

Example:

`[HIGH] [FUNCTION=x|1B] ... [LOW]`

---

# Packet Types

## Authorize

#### **1\. IS HERE**

Used to discover network groups.

`[HIGH] [FUNCTION=1|1B] [ANSWER_ID=random()|1B] [GROUP_NAME_LENGTH=L|1B] [GROUP_NAME_STRING=...|L*1B] [HASH|1B] [LOW]`

#### **2\. HERE IS**

##### YES

Confirms the group's existence.

All users in the group can respond to this packet, but the first user in the group — determined by the shortest random delay time — will send the reply.

`[HIGH] [FUNCTION=2|1B] [ANSWER_ID|1B] [GROUP_ID|2B] [CONNECT_ID|1B] [VERIFY_BYTES|2B] [SALT=random()|1B] [HASH|2B] [LOW]`

##### NO

No response if the group is not present (timeout: 1-2 seconds).

#### **3\. JOIN**

Request to join a group.

`[HIGH] [FUNCTION=3|1B] [GROUP_ID|2B] [CONNECT_ID|1B] [VERIFY_BYTES x (PASSWORD + SALT)|2B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

#### **4\. ACCEPT**

Response to a join request.

All users in the group can respond to this packet, but the first user in the group — determined by the shortest random delay time — will send the reply.

##### YES

Confirms the group's existence.

`[HIGH] [FUNCTION=4|1B] [GROUP_ID|2B] [CONNECT_ID|1B] /* Encrypted data starts here */ [USER_ID|2B] [CURRENT_SALT|2B] [SALT_MODIFIER_PER_PACKET=(MODIFIER + VALUE)|2B] [HASH|2B] [LOW]`

##### NO

No response if the group is not present (timeout: 1-2 seconds).

#### **5\. JOINED**

Acknowledges successful group joining.

`[HIGH] [FUNCTION=5|1B] [GROUP_ID|2B] /* Encrypted data starts here */ [USER_ID|2B] [CURRENT_SALT|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

---

## Get the Signs of the users in the group

#### **6\. WHO IS IN THE GROUP**

Ask who is in the group.

`[HIGH] [FUNCTION=6|1B] [GROUP_ID|2B] /* Encrypted data starts here */ [USER_ID|2B] [CURRENT_SALT|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

#### **7\. I AM IN THE GROUP**

Say that you are in the group.

`[HIGH] [FUNCTION=7|1B] [GROUP_ID|2B] /* Encrypted data starts here */ [USER_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

#### **8\. WRONG SIGN**

Indicate if a user sends the wrong sign hash.

`[HIGH] [FUNCTION=8|1B] [GROUP_ID|2B] /* Encrypted data starts here */ [USER_ID|2B] [USER_WITH_WRONG_SIGN_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

#### **9\. WRONG SIGN PACKET IS CORRUPTED**

Indicate when a hacker falsely claims a user has a wrong sign, but the sign is valid.

`[HIGH] [FUNCTION=8|1B] [GROUP_ID|2B] /* Encrypted data starts here */ [USER_ID|2B] [HACKER_USER_WITH_WRONG_SIGN_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

---

## Send

#### **10\. SEND**

Send data to a specific user.

`[HIGH] [FUNCTION=10|1B] [GROUP_ID|2B] [DATA_LENGTH=L|1B] /* Encrypted data starts here */ [USER_ID|2B] [USER_DESTINATION|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [DATA|L*1B] [LOW]`

#### **11\. SEND TO MULTIPLE USERS**

Broadcast data to multiple specific users.

`[HIGH] [FUNCTION=11|1B] [GROUP_ID|2B] [USERS_LENGTH=UL|2B] [DATA_LENGTH=DL|1B] /* Encrypted data starts here */ [USER_ID|2B] [USER_DESTINATIONS|UL*2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [DATA|DL*1B] [LOW]`

#### **12\. BROADCAST INNER GROUP**

Broadcast data within a group.

`[HIGH] [FUNCTION=12|1B] [GROUP_ID|2B] [DATA_LENGTH=L|1B] /* Encrypted data starts here */ [USER_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [DATA|L*1B] [LOW]`

#### **20\. BROADCAST INNER NETWORK**

Broadcast data to all devices in the network.

`[HIGH] [FUNCTION=20|1B] [DATA_LENGTH=L|1B] [HASH|4B] [DATA|L*1B] [LOW]`

#### **21\. SEND TO MAC INNER NETWORK**

Send a message to an user in the network by the mac-adress.

`[HIGH] [FUNCTION=21|1B] [MAC_ADRESS|4B] [DATA_LENGTH=L|1B] [HASH|4B] [DATA|L*1B] [LOW]`

---

### Key Concepts

- **Hashing**: All packets contain a hash to validate their integrity.
- **Signing**: All packets in a group contain a hash to validate which user send it.
- **Encryption**: Sensitive data fields are encrypted using a combination of a password and a salt.

This protocol ensures secure and reliable communication across multiple devices.
