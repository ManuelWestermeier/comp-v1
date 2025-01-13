# Netzwerkprotokoll V1

Dies ist ein cryptographisches verschlüsseltes dezentrales **Netzwerkprotokoll** für die Kommunikation mit 433 MHz Sendern und Empfängern, das mehrere Gruppen unterstützt.
Die Technologie kann auch für eine Kabelverbindungen verwendet werden.

# Übersicht

- **Wichtige Konzepte** - Die informatischen und cryptographischen Grundprinzipien
- **Signalzustände** - Wie werden die "Einsen und Nullen" gesendet?
- **Paketformat** - Was ist eigentlich ein Paket?
- **Grundlegende Datenübertragung** - Bytes/Zahlen senden und empfangen?
- **Paketübertragungsregeln** - Ab wann kann ein Paket gesendet werden?
- **Netzwerk-Hierarchie** - Wie ist das Netzwerk aufgebaut?
- **Signierung** - Wie kann sich jeder im Netzwerk sicher sein, dass ein Paket wirklich von einem bestimmten Benutzer gesendet wurde?
- **Paketstruktur** - Wie sind die Pakete aufgebaut und wie funktioniert das Netzwerkprotokoll?

# Wichtige Konzepte

- **Hashing**: Ein Hash is eine Einwegfunktion, die bei dem selben Input immer den selben Output ergibt. Von dem Output kann aber kein Input errechnet werden. Außerdem verändert sich der Output selbst bei kleinen Veränderungen stark. Alle Pakete enthalten einen Hash, um Fehler bei der datenübertragung des Pakets zu finden.
- **Signierung**: Alle Pakete in einer Gruppe enthalten einen Hash, um zu validieren, welcher Benutzer es gesendet hat.
- **Verschlüsselung**: Sensible Datenfelder werden unter Verwendung einer Kombination aus Passwort und Salt verschlüsselt.
- **Salt**: Eine Zusatzdatenmenge zu dem Verschlüsselungsschlüssel, der Megngenanalysen von verschlüsselten Daten erschwert.
- **Binäre Zahlen**: Ein Zahlensystem das nur mit den Ziffern 1 und 0 arbeitet. In diesem Fall Strom an (`HIGH`) als 1 und Strom aus als 0 (`LOW`).

# Signalzustände

- Die Verbindung kann entweder auf `HIGH` (aktiv/Strom fliest) oder `LOW` (inaktiv/Strom fliest nicht) gesetzt werden.
- Die Zustände können auch Binärziffern darstellen die zu Binärzahlen zusammengesätzt werden.
- Der Zustand wird in einem Interval (delayTime (***50 Mikrosekunden***) Zeit pro runde) geändert.

# Paketformat

Das Netzwerkprotokoll teilt die Daren die über die Leitung gesendet werden in Byte-Pakete, Pakete und Chunks ein. Dies sin virtuelle Einteilungen.

### Pakete: Jedes Paket folgt einem strukturierten Format:

Pakete setzen sich aus Chunks (kleinere Einheiten des Pakets mit Namen, Wert und Länge) zusammen.
Diese werden in eckigen Klammmern angegeben.

- `[HIGH]`: Markiert den Beginn des Pakets. Es ist immer ein 1 Bit Chunk.
- `[FUNCTION=x|1B]`: Ein festes 1-Byte-Feld, das den Zweck des Pakets definiert.
- Weitere Felder sind im Format `[NAME=VALUE|LENGTH]` angegeben:
  - **NAME**: Name des Chunks.
  - **VALUE**: Wert des Chunks oder dessen Standardwert oder Nichts.
  - **LENGTH**: Feldgröße, angegeben als `xB` (Bytes) oder `xBit` (Bits). Hier können auch vorherige Chunk-Veriabeln verwändet werden.
  - Verschlüsselte Werte werden als `VALUE x (PASSWORD + SALT)` angegeben.
  - Felder können aus verketteten Chunks bestehen.
- `[LOW]`: Markiert das Ende des Pakets. Es ist immer ein 1 Bit Chunk.

#### Beispiel:

`[HIG] [FN=100|1B] [CHUNK1|2B] [LENGTH|1B] [DATA|LENGTH*1B] [LOW]`

#### Byte-Pakete: Die Möglichkeit 1 Byte (8 Bits) und die isFollowing Flag (1Bit) (source code unter dieser Sektion unter "Grundlegende Datenübertragung")

# Grundlegende Datenübertragung

## Sender

In den Eckigen Klammern ist ein Wert. Dieser Wert zeigt den Zustand (`HIGH`/`LOW`).
`HIGH` steht für "ja" oder "1", `LOW` steht für "nein" oder 0.

- Am Anfang wird die "Leitung" auf `HIGH` gesetzt, was den Start des Bytepakets kennzeichnet.
- Am Ende wird die "Leitung" auf `LOW` gesetzt, was dafür sorgt, dass die Letung bei dem nächsten Paket am Anfang wieder auf `HIGH` gestzt werden kann (Zustandsänderung).

#### Einfache Darstellung: [XY] dauern ein Zeitinterval (delayTime/50Microsekunden)

`[HIGH] [IS_FOLLOWING] [BIT_8] [BIT_7] [BIT_6] [BIT_5] [BIT_4] [BIT_3] [BIT_2] [BIT_1] [LOW]`

```cpp
//WF=With isFollowingFlag
void rawSendByteWF(uint8_t value, int pin, int delayTime, bool isFollowing)
{
    // begin des Pakets
    digitalWrite(pin, HIGH);
    delayMicroseconds(delayTime); // Pause

    // das erste Bit zeigt, ob das Paket auf ein anders Paket folgt oder der Start eines neuen Pakets ist
    digitalWrite(pin, isFollowing ? HIGH : LOW);
    delayMicroseconds(delayTime); // Pause

    // Jeder Daten-Bit eines Bytes (8 Bits) senden (LSB-first)
    for (uint8_t i = 0; i < 8; i++)
    {
        // Jedes Bit extrahieren und dann senden
        bool bit = (value & (1 << i)) != 0;
        digitalWrite(pin, bit ? HIGH : LOW);

        // Pause
        delayMicroseconds(delayTime);
    }

    // Ende des Pakets
    digitalWrite(pin, LOW);
    delayMicroseconds(delayTime); // Pause
}
```

## Empfänger

```cpp
// Datentyp zur vereinfachung
struct RawPacket
{
    bool isFollowing;
    uint8_t data;
};

//WF=With isFollowingFlag
RawPacket rawReadByteWF(uint8_t pin, int delayTime)
{
    RawPacket packet;
    packet.data = 0;

    // Auf das Startsignal warten
    while (digitalRead(pin) != HIGH)
        ;

    // 1.5 * delayTime (50 Microsekunden) warten (damit es bei der Hälfte des nächsten Bits anfängt den Wert auszulesen)
    delayMicroseconds(delayTime * 1.5);

    // isFollowingFlag auslesen
    packet.isFollowing = (digitalRead(pin) == HIGH);
    delayMicroseconds(delayTime);

    // Jedes Bit auslesen und zu einem Byte zusammensätzen
    for (uint8_t i = 0; i < 8; i++)
    {
        if (digitalRead(pin) == HIGH)
        {
            packet.data |= (1 << i); // LSB-first
        }
        delayMicroseconds(delayTime);
    }

    return packet;
}
```

# Paketübertragungsregeln

1.  **Startbedingungen**:

    - Um ein Paket zu senden, stellt der Sender sicher, dass die Verbindung für eine festgelegte "maximale Sendezeit" (13 _ delayTime) auf `LOW` bleibt.
      Da pro Byte-Paket mindestens 1 mal der Zustand `HIGH` übertragen werden muss (am Start) und das Senden eines Byte-Paket ca. 12 _ delayTime + 1 _ delayTime (Puffer) dauert kann mann sagen, dass wenn die Leitung 13 _ delayTime (50 Microsekunden) lang auf `LOW` steht Nichts gesendet wurde.
      Und der Nutzer mit dem sicheren Paketlesen andfangen kann.
    - Wenn die Verbindung während dieses Zeitraums auf `HIGH` wechselt, muss der Sender den Versuch wiederholen.

### Code Beispiel

```cpp
void waitForBytePacketEnd()
{
    // den Zeitpunkt, an dem (connection.sendDelay * 13) lange nichts gesändet wurde, was heißt, dass das letzte Paket zu Ende ist
    auto timeToWait = micros() + (connection.sendDelay * 13);
    while (true)
    {
        auto now = micros();
        //wenn lange genug gewartet wurde, returnt die funktion und der code dahinter kann reibungslos ausgeführt werden.
        if (now > timeToWait)
        {
            return;
        }
        // wenn doch etwas gesendet wird, wird der timer zurück gesetzt
        else if (digitalRead(connection.inpPin) == HIGH)
        {
            auto timeToWait = now + (connection.sendDelay * 13);
        }
    }
}
```

2.  **Kollisionsvermeidung**:

    - Geräte verwenden die Zeit seit der letzten Paketübertragung, um für ein zufälliges Intervall zwischen 1000 und 50000 Mikroseunden zu warten, bevor sie versuchen, die Verbindung auf `HIGH` zu ziehen. Wenn der Sender mehrmals versucht ein Paket zu senden, wird die maxiamle Zufallszeit verkürzt, dass es warscheinlicher wird, das Paket als nächstes zu senden.
    - Bleibt die Leitung bis die zufällige Wartezeit vorbei ist auf `LOW`, kann der Sender mit der Übertragung des Pakets fortfahren.

# Netzwerk-Hierarchie

Das **NETZWERK** ist die physische Verbindung, die mit 433 MHz RF-Modulen (oder mit einer Kabelverbindung) hergestellt wird.

Die **GRUPPEn** sind virtuelle Netzwerke, die Verschlüsselung für eine sichere Kommunikation implementieren. Es kann bis zu **65.536 GRUPPEn** geben.
Benutzer innerhalb einer GRUPPE können den Verbindungsprozess verwalten, daten senden oder bis zu **65.536 BENUTZER** einladen.  
Jeder Benutzer ist Mitglied einer oder mehrerer GRUPPEn und kann gleichzeitig mit mehreren Netzwerken verbunden sein.
In Jeder der Gruppen is jeder Benutzer gleichberechtigt.

Benutzer können folgende Aktionen durchführen:

- Verschlüsselte Nachrichten an andere Benutzer innerhalb der GRUPPE senden.
- Nachrichten an alle Mitglieder der GRUPPE broadcasten.
- Nachrichten an das gesamte NETZWERK senden.
- Nachrichten im gesamte NETZWERK mit MAC-Adressen senden.
- Auf Pakete antworten.

## Übersicht der Hierarchie

```plaintext
NETZWERK (433Mhz / Kabelverbindung)
  ├── GRUPPE 1
  │     ├── BENUTZER 1
  │     ├── BENUTZER 2
  │     └── BENUTZER 3
            ...
  ├── GRUPPE 2
  │     ├── BENUTZER 27224
  │     └── BENUTZER 2885
            ...
  └── GRUPPE 3
        ├── BENUTZER 6
        ├── BENUTZER 7
        ├── BENUTZER 8
        └── BENUTZER 9
            ...
  ...
```

Diese Hierarchie gewährleistet eine strukturierte Organisation der Benutzer innerhalb sicherer und skalierbarer virtueller GRUPPEn, unterstützt durch ein robustes physisches NETZWERK.

# Signierung

1. **Generierung eines zufälligen Werts**  
   Der Benutzer generiert einen zufälligen 4-Byte-Wert, bezeichnet als `SIGN_VALUE`.

2. **Hashen des Werts**  
   Der Benutzer hasht den Wert `SIGN_VALUE` und extrahiert die letzten 4 Bytes des Hashs, bezeichnet als `SIGN_VALUE_HASH`.

3. **Teilen des Hashs**  
   Der Wert von `SIGN_VALUE_HASH` wird mit allen Mitgliedern der Gruppe geteilt.

4. **Signieren einer Nachricht**  
   Um eine Nachricht zu signieren, sendet der Benutzer:

   - Den ursprünglichen zufälligen Wert (`SIGN_VALUE`)
   - Den nächsten gehashten zufälligen Wert (`NEXT_SIGN_VALUE_HASH`), um die nächste Nachricht zu identifizieren (diese muss im gleichen Paket gesendet werden)

Benutzer die überprüfen wollen, ob eine Nachricht von dem richtigen Benutzer gesendet wurde, können den HASH des Gesendeten Werts `SIGN_VALUE` mit dem Hash `SIGN_VALUE_HASH` abgleichen.

Da eine Hashfunktion eine Einwegfunktion ist kann kein übereinstimmender Wert ausgängig von dem Hash generiert werden.
Dies stellt sicher, dass jede Nachricht eindeutig identifizierbar ist.

Das Format lautet:

1. Erstes Packet: (`... [CURRENT_SIGN_HASH|4B] ...`) (Dies zeigt den Nutzern den Aktuellen Hash).
2. Folgende Packete: (`... [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] ...`) (Kann ein Paket "unterschreiben" und Bringt den Hash für das Nächste paket mit).

---

# Pakettypen

## Autorisieren

#### **1. IS HERE**

Wird verwendet, um Gruppen zu finden.

`[HIGH] [FUNCTION=1|1B] [PACKET_ID=random()|2B] [ANSWER_ID=random()|2B] [GROUP_NAME_LENGTH=L|1B] [GROUP_NAME_STRING=...|L*1B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [LOW]`

#### **2. HERE IS**

##### JA

Bestätigt die Existenz der Gruppe.

Alle Benutzer in der Gruppe können auf dieses Paket antworten, aber der erste Benutzer in der Gruppe (bestimmt durch die kürzeste zufällige Verzögerungszeit) sendet die Antwort.

`[HIGH] [FUNCTION=2|1B] [PACKET_ID=random()|2B] [ANSWER_ID|2B] [GROUP_ID|2B] [CONNECT_ID|2B] [VERIFY_BYTES=ranom()|4B] [SALT=random()|2B] [HASH|4B] [LOW]`

##### NEIN

Keine Antwort, wenn die Gruppe nicht vorhanden ist (Timeout: 1-2 Sekunden).

#### **3. JOIN**

Anfrage zum Beitreten einer Gruppe.

`[HIGH] [FUNCTION=3|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] [CONNECT_ID|2B] [VERIFY_BYTES x (PASSWORD + SALT)|4B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [LOW]`

#### **4. ACCEPT**

Antwort auf eine Beitrittsanfrage.

Alle Benutzer in der Gruppe können auf dieses Paket antworten, aber der erste Benutzer in der Gruppe (bestimmt durch die kürzeste zufällige Verzögerungszeit) sendet die Antwort.

##### JA

Bestätigt die Existenz der Gruppe.

`[HIGH] [FUNCTION=4|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] [CONNECT_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [CURRENT_SALT|4B] [ERROR_IDENTIFYER=random()|2B] [SALT_MODIFIER_PER_PACKET=(MODIFIER + VALUE)|2B] [HASH|4B] [LOW]`

##### NEIN

Keine Antwort, wenn die Gruppe nicht vorhanden ist (Timeout: 1-2 Sekunden).

#### **5. JOINED**

Bestätigt den erfolgreichen Beitritt zur Gruppe.

`[HIGH] [FUNCTION=5|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [ERROR_IDENTIFYER|2B] [USER_ID|2B] [CURRENT_SALT|4B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [LOW]`

---

## Erhalten der Signaturen der Benutzer in der Gruppe

#### **6. WHO IS IN THE GROUP**

Fragt, wer in der Gruppe ist.

`[HIGH] [FUNCTION=6|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [CURRENT_SALT|4B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [LOW]`

#### **7. I AM IN THE GROUP**

Meldet, dass man in der Gruppe ist.

`[HIGH] [FUNCTION=7|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [LOW]`

#### **8. WRONG SIGN**

Weist darauf hin, wenn ein Benutzer den falschen Signatur-Hash sendet.

`[HIGH] [FUNCTION=8|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [USER_WITH_WRONG_SIGN_ID|2B] [WRONG_SIGN_PACKET_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [LOW]`

#### **9. WRONG SIGN PACKET IS CORRUPTED**

Weist darauf hin, wenn ein Hacker fälschlicherweise behauptet, ein Benutzer habe eine falsche Signatur, die jedoch gültig ist. Diese Kann nur durch einen anderen Benutzer gesendet werden (nicht HACKER).

`[HIGH] [FUNCTION=9|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [HACKER_USER_WITH_WRONG_SIGN_ID|2B] [WRONG_SIGN_PACKET_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [LOW]`

---

## Senden

#### **10. SEND**

Sendet Daten an einen bestimmten Benutzer.

`[HIGH] [FUNCTION=10|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] [DATA_LENGTH=L|1B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [USER_DESTINATION|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|6B] [DATA|L*1B] [LOW]`

#### **11. SEND TO MULTIPLE USERS**

Sendet Daten an mehrere spezifische Benutzer.

`[HIGH] [FUNCTION=11|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] [USERS_LENGTH=UL|2B] [DATA_LENGTH=DL|1B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [USER_DESTINATIONS|UL*2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|6B] [DATA|DL*1B] [LOW]`

#### **12. BROADCAST INNER GROUP**

Sendet Daten innerhalb einer Gruppe an alle Mitglieder.

`[HIGH] [FUNCTION=12|1B] [PACKET_ID=random()|2B] [GROUP_ID|2B] [DATA_LENGTH=L|1B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|6B] [DATA|L*1B] [LOW]`

#### **20. BROADCAST INNER NETWORK**

Sendet Daten an alle Geräte im Netzwerk.

`[HIGH] [FUNCTION=20|1B] [PACKET_ID=random()|2B] [DATA_LENGTH=L|1B] [HASH|6B] [DATA|L*1B] [LOW]`

#### **21. SEND TO MAC INNER NETWORK**

Sendet eine Nachricht an einen Benutzer im Netzwerk über die MAC-Adresse.

`[HIGH] [FUNCTION=21|1B] [PACKET_ID=random()|2B] [MAC_ADRESS|4B] [DATA_LENGTH=L|1B] [HASH|6B] [DATA|L*1B] [LOW]`

#### **30. PACKET DATA ERROR**

Wenn der hash nicht mit den gesendeten Daten übereinstimmt, wird dieses Paket gesendet.

`[HIGH] [FUNCTION=30|1B] [PACKET_ID=random()|2B] [ERROR_PACKET_ID=random()|2B] [HASH|1B] [LOW]`
packet

---

Dieses Protokoll gewährleistet eine sichere und zuverlässige Kommunikation über mehrere Geräte hinweg.
