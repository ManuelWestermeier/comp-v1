# Netzwerkprotokoll V1

Dies ist ein verschlüsseltes dezentrales **Netzwerkprotokoll** für die Kommunikation mit 433 MHz Sendern und Empfängern, das mehrere Gruppen unterstützt.
Die Technologie kann auch für eine Kabelverbindung verwendet werden.

### Übersicht

- **Netzwerk-Hierarchie**
- **Signierung**
- **Signalzustände**: Die Verbindung kann entweder auf `HIGH` (aktiv) oder `LOW` (inaktiv) gesetzt werden. Die Zustände können auch Binärziffern darstellen.
- **Übertragungstiming**:
  - Jedes Bit wird alle 50 Mikroseunden gesendet.
  - Bits werden in Bytes gruppiert, die wiederum in Pakete organisiert werden.
- **Paketstruktur**:
  - Jedes Paket beginnt mit einem `[HIGH]`-Signal, um den Beginn zu kennzeichnen.
  - Die Paketdaten enthalten binäre Darstellungen verschiedener Felder.
  - Pakete enden mit einem `[LOW]`-Signal, das den Beginn des nächsten Pakets anzeigt.

# Netzwerk-Hierarchie

Das **NETZWERK** ist die physische Verbindung, die mit 433 MHz RF-Modulen (oder mit einer Kabelverbindung) hergestellt wird.

Die **GRUPPEn** sind virtuelle Netzwerke, die Verschlüsselung für eine sichere Kommunikation implementieren.  
Benutzer innerhalb einer GRUPPE können den Verbindungsprozess verwalten, daten senden oder bis zu **65.536 BENUTZER** einladen.  
Jeder Benutzer ist Mitglied einer oder mehrerer GRUPPEn und kann gleichzeitig mit mehreren Netzwerken verbunden sein.

Benutzer können folgende Aktionen durchführen:

- Verschlüsselte Nachrichten an andere Benutzer innerhalb der GRUPPE senden.
- Nachrichten an alle Mitglieder der GRUPPE broadcasten.
- Nachrichten an das gesamte NETZWERK senden.
- Nachrichten im gesamte NETZWERK mit MAC-Adressen senden.

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
   - Den nächsten gehashten zufälligen Wert (`NEXT_SIGN_VALUE_HASH`), um die nächste Nachricht zu identifizieren (dies muss im gleichen Paket gesendet werden)

Benutzer die überprüfen wollen, ob eine Nachricht vom richtigen Benutzer gesendet wurde, können den HASH des Gesendeten `SIGN_VALUE_HASH` mit dem Hash `SIGN_VALUE_HASH` abgleichen.
Dies stellt sicher, dass jede Nachricht eindeutig identifizierbar ist und den Hash für die folgende Nachricht festlegt.  

Das Format lautet:
1. Erstes Packet: (` ... [LAST_SIGN_VALUE|4B] ... `)
2. Folgende Packete: (` ... [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] ... `).

# Paketübertragungsregeln

1.  **Startbedingungen**:

    - Um ein Paket zu senden, stellt der Sender sicher, dass die Verbindung für eine festgelegte _maximale Sendezeit_ auf `LOW` bleibt.
    - Wenn die Verbindung während dieses Zeitraums auf `HIGH` wechselt, muss der Sender den Versuch wiederholen.

2.  **Kollisionsvermeidung**:

    - Geräte verwenden die Zeit seit der letzten Paketübertragung, um für ein zufälliges Intervall zwischen 1000 und 50000 Mikroseunden zu warten, bevor sie versuchen, die Verbindung auf `HIGH` zu ziehen.
    - Bleibt die Leitung auf `LOW`, kann der Sender mit der Übertragung des Pakets fortfahren.

# Paketformat

Jedes Paket folgt einem strukturierten Format:

- `[HIGH]`: Markiert den Beginn des Pakets.
- `[FUNCTION=x|1B]`: Ein festes 1-Byte-Feld, das den Zweck des Pakets definiert.
- Weitere Felder sind im Format `NAME=VALUE|LENGTH` angegeben:
  - **NAME**: Name des Feldes.
  - **VALUE**: Wert des Feldes oder dessen Standardwert.
  - **LENGTH**: Feldgröße, angegeben als `xB` (Bytes) oder `xBit` (Bits).
  - Verschlüsselte Werte werden als `VALUE x (PASSWORD + SALT)` angegeben.
  - Felder können aus verketteten Chunks bestehen.

Beispiel:

`[HIGH] [FUNCTION=x|1B] ... [LOW]`

---

# Pakettypen

## Autorisieren

#### **1. IS HERE**

Wird verwendet, um Netzwerkgruppen zu entdecken.

`[HIGH] [FUNCTION=1|1B] [ANSWER_ID=random()|1B] [GROUP_NAME_LENGTH=L|1B] [GROUP_NAME_STRING=...|L*1B] [HASH|1B] [LOW]`

#### **2. HERE IS**

##### YES

Bestätigt die Existenz der Gruppe.

Alle Benutzer in der Gruppe können auf dieses Paket antworten, aber der erste Benutzer in der Gruppe — bestimmt durch die kürzeste zufällige Verzögerungszeit — sendet die Antwort.

`[HIGH] [FUNCTION=2|1B] [ANSWER_ID|1B] [GROUP_ID|2B] [CONNECT_ID|1B] [VERIFY_BYTES|2B] [SALT=random()|1B] [HASH|2B] [LOW]`

##### NO

Keine Antwort, wenn die Gruppe nicht vorhanden ist (Timeout: 1-2 Sekunden).

#### **3. JOIN**

Anfrage zum Beitreten einer Gruppe.

`[HIGH] [FUNCTION=3|1B] [GROUP_ID|2B] [CONNECT_ID|1B] [VERIFY_BYTES x (PASSWORD + SALT)|2B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

#### **4. ACCEPT**

Antwort auf eine Beitrittsanfrage.

Alle Benutzer in der Gruppe können auf dieses Paket antworten, aber der erste Benutzer in der Gruppe — bestimmt durch die kürzeste zufällige Verzögerungszeit — sendet die Antwort.

##### YES

Bestätigt die Existenz der Gruppe.

`[HIGH] [FUNCTION=4|1B] [GROUP_ID|2B] [CONNECT_ID|1B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [CURRENT_SALT|2B] [SALT_MODIFIER_PER_PACKET=(MODIFIER + VALUE)|2B] [HASH|2B] [LOW]`

##### NO

Keine Antwort, wenn die Gruppe nicht vorhanden ist (Timeout: 1-2 Sekunden).

#### **5. JOINED**

Bestätigt den erfolgreichen Beitritt zur Gruppe.

`[HIGH] [FUNCTION=5|1B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [CURRENT_SALT|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

---

## Erhalten der Signaturen der Benutzer in der Gruppe

#### **6. WHO IS IN THE GROUP**

Fragt, wer in der Gruppe ist.

`[HIGH] [FUNCTION=6|1B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [CURRENT_SALT|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

#### **7. I AM IN THE GROUP**

Meldet, dass Sie in der Gruppe sind.

`[HIGH] [FUNCTION=7|1B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

#### **8. WRONG SIGN**

Weist darauf hin, wenn ein Benutzer den falschen Signatur-Hash sendet.

`[HIGH] [FUNCTION=8|1B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [USER_WITH_WRONG_SIGN_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

#### **9. WRONG SIGN PACKET IS CORRUPTED**

Weist darauf hin, wenn ein Hacker fälschlicherweise behauptet, ein Benutzer habe eine falsche Signatur, die jedoch gültig ist.

`[HIGH] [FUNCTION=9|1B] [GROUP_ID|2B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [HACKER_USER_WITH_WRONG_SIGN_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

---

## Senden

#### **10. SEND**

Sendet Daten an einen bestimmten Benutzer.

`[HIGH] [FUNCTION=10|1B] [GROUP_ID|2B] [DATA_LENGTH=L|1B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [USER_DESTINATION|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [DATA|L*1B] [LOW]`

#### **11. SEND TO MULTIPLE USERS**

Sendet Daten an mehrere spezifische Benutzer.

`[HIGH] [FUNCTION=11|1B] [GROUP_ID|2B] [USERS_LENGTH=UL|2B] [DATA_LENGTH=DL|1B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [USER_DESTINATIONS|UL*2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [DATA|DL*1B] [LOW]`

#### **12. BROADCAST INNER GROUP**

Sendet Daten innerhalb einer Gruppe an alle Mitglieder.

`[HIGH] [FUNCTION=12|1B] [GROUP_ID|2B] [DATA_LENGTH=L|1B] /* Verschlüsselte Daten beginnen hier */ [USER_ID|2B] [LAST_SIGN_VALUE|4B] [CURRENT_SIGN_HASH|4B] [HASH|4B] [DATA|L*1B] [LOW]`

#### **20. BROADCAST INNER NETWORK**

Sendet Daten an alle Geräte im Netzwerk.

`[HIGH] [FUNCTION=20|1B] [DATA_LENGTH=L|1B] [HASH|4B] [DATA|L*1B] [LOW]`

#### **21. SEND TO MAC INNER NETWORK**

Sendet eine Nachricht an einen Benutzer im Netzwerk über die MAC-Adresse.

`[HIGH] [FUNCTION=21|1B] [MAC_ADRESS|4B] [DATA_LENGTH=L|1B] [HASH|4B] [DATA|L*1B] [LOW]`

---

### Wichtige Konzepte

- **Hashing**: Alle Pakete enthalten einen Hash, um die Integrität zu validieren.
- **Signierung**: Alle Pakete in einer Gruppe enthalten einen Hash, um zu validieren, welcher Benutzer es gesendet hat.
- **Verschlüsselung**: Sensible Datenfelder werden unter Verwendung einer Kombination aus Passwort und Salz verschlüsselt.

Dieses Protokoll gewährleistet eine sichere und zuverlässige Kommunikation über mehrere Geräte hinweg.
