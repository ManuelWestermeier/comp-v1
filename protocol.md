# Networking Protocol V1

This is a multigroup encrypted **Networking Protocol** designed for communication using 433 MHz transmitters and receivers.

### Overview

- **Signal States**: The connection can either be pulled `HIGH` (active) or remain `LOW` (idle).
- **Transmission Timing**:
  - Each bit is sent every 1000 microseconds.
  - Bits are grouped into bytes, which are further organized into packets.
- **Packet Structure**:
  - Each packet begins with a `[HIGH]` signal to mark its start.
  - The packet data contains binary representations of various fields.
  - Packets end with a `[LOW]` signal, indicating the next packet's beginning.

### Packet Transmission Rules

1.  **Start Conditions**:

    - To send a packet, the sender ensures the connection remains `LOW` for a specified _max send time_.
    - If the connection goes `HIGH` during this period, the sender must retry.

2.  **Collision Avoidance**:

    - Devices use the time since the last packet's transmission to wait for a random interval between 1000 and 50000 microseconds before attempting to pull the connection `HIGH`.
    - If the line stays `LOW`, the sender may proceed to transmit a packet.

### Packet Format

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

### Packet Types

#### **1\. IS HERE**

Used to discover network groups.

`[HIGH] [FUNCTION=1|1B] [ANSWER_ID=random()|1B] [GROUP_NAME_LENGTH=L|1B] [GROUP_NAME_STRING=...|LB] [HASH|1B] [LOW]`

#### **2\. HERE IS**

##### YES

Confirms the group's existence.

`[HIGH] [FUNCTION=2|1B] [ANSWER_ID|1B] [GROUP_ID|2B] [CONNECT_ID|1B] [VERIFY_BYTES|2B] [SALT=random()|1B] [HASH|2B] [LOW]`

##### NO

No response if the group is not present (timeout: 1-2 seconds).

---

#### **3\. JOIN**

Request to join a group.

`[HIGH] [FUNCTION=3|1B] [GROUP_ID|2B] [CONNECT_ID|1B] [VERIFY_BYTES x (PASSWORD + SALT)|2B] [HASH|2B] [LOW]`

#### **4\. ACCEPT**

Response to a join request.

`[HIGH] [FUNCTION=4|1B] [CONNECT_ID|1B] /* Encrypted data starts here */ [GROUP_ID|2B] [USER_ID|2B] [IS_ACCEPTED=1|1B] [CURRENT_SALT|2B] [SALT_MODIFIER_PER_PACKET=(MODIFIER + VALUE)|2B] [HASH|2B] [LOW]`

#### **5\. JOINED**

Acknowledges successful group joining.

`[HIGH] [FUNCTION=5|1B] [GROUP_ID|2B] /* Encrypted data starts here */ [USER_ID|2B] [CURRENT_SALT|2B] [HASH|2B] [LOW]`

---

#### **6\. SEND**

Send data to a specific user.

`[HIGH] [FUNCTION=6|1B] [GROUP_ID|2B] /* Encrypted data starts here */ [USER_ID|2B] [USER_DESTINATION|2B] [DATA_LENGTH=L|1B] [HASH|4B] [DATA|LB] [LOW]`

#### **7\. SEND TO MULTIPLE USERS**

Broadcast data to multiple specific users.

`[HIGH] [FUNCTION=7|1B] [GROUP_ID|2B] /* Encrypted data starts here */ [USER_ID|2B] [USERS_LENGTH=L|2B] [USER_DESTINATIONS|L*2B] [DATA_LENGTH=L|1B] [HASH|4B] [DATA|LB] [LOW]`

#### **8\. BROADCAST INNER GROUP**

Broadcast data within a group.

`[HIGH] [FUNCTION=8|1B] [GROUP_ID|2B] /* Encrypted data starts here */ [USER_ID|2B] [DATA_LENGTH=L|1B] [HASH|4B] [DATA|LB] [LOW]`

#### **9\. BROADCAST INNER NETWORK**

Broadcast data to all devices in the network.

`[HIGH] [FUNCTION=9|1B] [GROUP_ID|2B] [DATA_LENGTH=L|1B] [HASH|4B] [DATA|LB] [LOW]`

---

### Key Concepts

- **Hashing**: All packets contain a hash to validate their integrity.
- **Encryption**: Sensitive data fields are encrypted using a combination of a password and a salt.

This protocol ensures secure and reliable communication across multiple devices.
