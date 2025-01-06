# Networking Protocol V1

### Pockets

Every pocket follows this concept:

- `[HIGH]` are impulses that shows the start of the pocket. After the pocket the connection is pulled `[LOW]`.
- `[FUNCTION]` is a fixed length 1 byte integer representing the function being performed.

all pockets are split into chunks: `[NAME=VALUE|LENGTH]`.

- NAME is the name of the chunk
- VALUE is the value or the default value of the chunk
- LENGTH is the length of the chunk. It is xB = x Bytes or xBit = x Bits
- `VALUE x (PASSWORD + SALT)` means that the value is encrypted by the password and salt
- `( (NAME=VALUE|LENGTH) + (NAME=VALUE|LENGTH) )` means a concatenation of smaller chunks

the default is:

```css
 [HIGH] [FUNCTION=x|1B] ... [LOW]
```

### The Pocket Types

#### IS HERE

```css
[HIGH] [FUNCTION=1|1B] [ANSWERID=random()|1B] [GROUP_NAME_LENGTH=L|1B] [GROUP_NAME_STRING=...|LB] [HASH|1B] [LOW]
```

#### HERE IS

##### YES

```css
[HIGH] [FUNCTION=2|1B] [ANSWERID=ANSWER_ID_FROM_IS_HERE_POCKET|1B] [GROUP_ID=GROUP_ID|2B] [CONNECT_ID=1B] [VERIFY_BYTE=1B] [SALT=random()|1B] [HASH|2B] [LOW]
```

##### NO

(if the network isn't present it gets no answer back after 1-2s)

#### JOIN

```css
[HIGH] [FUNCTION=3|1B] [GROUP_ID|2B] [CONNECT_ID=1B] [VERIFY_BYTE=VERIFY_BYTE x (PASSWORD + SALT)|1B] [HASH|1B] [LOW]
```

#### ACCEPT

```css
[HIGH] [FUNCTION=4|1B] [CONNECT_ID=1B] /*everything after here is encrypted using password & salt from before*/ [GROUP_ID|2B] [USER_ID|2B] [IS_ACCEPTED=(yes=1;no=0)|1B] [CURRENT_SALT=2B] [SALT_MODIFIER_PER_POCKET=( (MODIFYER=+-*/|2Bit) + (VALUE|14Bit) )|2B] [HASH|2B] [LOW]
```

#### JOINED

```css
[HIGH] [FUNCTION=5|1B] [GROUP_ID|2B] /*everything after here is encrypted using password & salt from before*/ [USER_ID=2B] [CURRENT_SALT=SALT_MODIFYED|2B] [HASH|2B] [LOW]
```

#### SEND

```css
[HIGH] [FUNCTION=6|1B] [GROUP_ID|2B] /*everything after here is encrypted using password & salt from before*/ [USER_ID|2B] [USER_DESTINATION|2B] [DATA_LENGTH=L|1B] [HASH|4B] [DATA=...|LB] [LOW]
```
