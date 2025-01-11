# Netzwerkprotokoll V1

Dies ist ein multigruppen-verschlüsseltes **Netzwerkprotokoll**, entwickelt für die Kommunikation mithilfe von 433-MHz-Sendern und -Empfängern.

### Überblick

- **Netzwerkhierarchie**
- **Signaturen**
- **Signalzustände**: Die Verbindung kann entweder auf `HIGH` (aktiv) gezogen werden oder bleibt `LOW` (inaktiv).
- **Übertragungszeit**:
  
  - Jedes Bit wird alle 1000 Mikrosekunden gesendet.
  - Bits werden in Bytes gruppiert, die wiederum in Pakete organisiert werden.
- **Paketstruktur**:
  
  - Jedes Paket beginnt mit einem `[HIGH]`-Signal zur Markierung des Starts.
  - Die Paketdaten enthalten Binärdarstellungen verschiedener Felder.
  - Pakete enden mit einem `[LOW]`-Signal, das den Anfang des nächsten Pakets anzeigt.

* * *

## Netzwerkhierarchie

Das **Netzwerk** ist die physische Verbindung, die durch 433-MHz-RF-Module hergestellt wird.

**Gruppen** sind virtuelle Netzwerke, die zur sicheren Kommunikation Verschlüsselung implementieren.  
Nutzer innerhalb einer Gruppe können Verbindungen verwalten oder bis zu **65.536 Gruppen** erstellen.  
Jede Gruppe kann bis zu **65.536 Nutzer** haben.  
Ein Nutzer kann Mitglied in mehreren Gruppen sein und sich gleichzeitig mit mehreren Netzwerken verbinden.

**Funktionen der Nutzer**:

- Verschlüsselte Nachrichten an andere Nutzer innerhalb der Gruppe senden.
- Nachrichten an alle Mitglieder der Gruppe broadcasten.
- Nachrichten an das gesamte Netzwerk broadcasten.

### Hierarchie-Übersicht

```plaintext


plaintext


Code kopieren
Netzwerk
  ├── Gruppe 1
  │     ├── Nutzer 1
  │     ├── Nutzer 2
  │     └── Nutzer 3
            ...
  ├── Gruppe 2
  │     ├── Nutzer 27224
  │     └── Nutzer 2885
            ...
  └── Gruppe 3
        ├── Nutzer 6
        ├── Nutzer 7
        ├── Nutzer 8
        └── Nutzer 9
            ...
  ...
```

Diese Hierarchie gewährleistet eine strukturierte Organisation der Nutzer in sicheren und skalierbaren virtuellen Gruppen, unterstützt durch ein robustes physisches Netzwerk.

* * *

## Signaturen

1. **Zufallswert erzeugen**  
   Der Nutzer generiert einen zufälligen 4-Byte-Wert, genannt `SIGN_VALUE`.
2. **Hash-Wert berechnen**  
   Der Nutzer hasht `SIGN_VALUE` und extrahiert die letzten 4 Bytes des Hashes, genannt `SIGN_VALUE_HASH`.
3. **Hash teilen**  
   Der Wert von `SIGN_VALUE_HASH` wird mit allen Gruppenmitgliedern geteilt.
4. **Nachricht signieren**  
   Um eine Nachricht zu signieren, sendet der Nutzer:
   
   - Den ursprünglichen Zufallswert (`SIGN_VALUE`).
   - Den Hash des nächsten Zufallswertes (`NEXT_SIGN_VALUE_HASH`), um die nächste Nachricht zu identifizieren (dies muss im gleichen Paket gesendet werden).

Dies stellt sicher, dass jede Nachricht eindeutig identifizierbar ist und den Hash für die folgende Nachricht etabliert.

* * *

## Paketübertragungsregeln

1. **Startbedingungen**:
   
   - Um ein Paket zu senden, stellt der Sender sicher, dass die Verbindung für eine festgelegte maximale Sendezeit `LOW` bleibt.
   - Geht die Verbindung währenddessen auf `HIGH`, muss der Sender erneut versuchen.
2. **Kollisionsvermeidung**:
   
   - Geräte verwenden die Zeit seit der letzten Paketübertragung, um ein zufälliges Intervall zwischen 1000 und 50.000 Mikrosekunden zu warten, bevor sie die Verbindung auf `HIGH` ziehen.
   - Bleibt die Leitung `LOW`, kann der Sender ein Paket übertragen.

* * *

## Paketformat

Jedes Paket folgt einem strukturierten Format:

- `[HIGH]`: Markiert den Start des Pakets.
- `[FUNCTION=x|1B]`: Ein festes 1-Byte-Feld, das den Zweck des Pakets definiert.
- Weitere Felder werden im Format `NAME=VALUE|LENGTH` angegeben:
  
  - **NAME**: Name des Feldes.
  - **VALUE**: Wert des Feldes oder dessen Standardwert.
  - **LENGTH**: Feldgröße in `xB` (Bytes) oder `xBit` (Bits).
  - Verschlüsselte Werte werden als `VALUE x (PASSWORD + SALT)` dargestellt.
  - Felder können aus zusammengesetzten Daten bestehen.

Beispiel:

`[HIGH] [FUNCTION=x|1B] ... [LOW]`

* * *

## Paketarten

### Autorisierung

#### **1. IS HERE**

Wird verwendet, um Gruppen im Netzwerk zu entdecken.

`[HIGH] [FUNCTION=1|1B] [ANSWER_ID=random()|1B] [GROUP_NAME_LENGTH=L|1B] [GROUP_NAME_STRING=...|L*1B] [HASH|1B] [LOW]`

#### **2. HERE IS**

##### JA

Bestätigt die Existenz der Gruppe.  
Alle Nutzer der Gruppe können antworten, jedoch antwortet der erste Nutzer mit der kürzesten zufälligen Verzögerungszeit.

`[HIGH] [FUNCTION=2|1B] [ANSWER_ID|1B] [GROUP_ID|2B] [CONNECT_ID|1B] [VERIFY_BYTES|2B] [SALT=random()|1B] [HASH|2B] [LOW]`

##### NEIN

Keine Antwort, falls die Gruppe nicht existiert (Timeout: 1–2 Sekunden).

#### **3. JOIN**

Anfrage zum Beitreten einer Gruppe.

`[HIGH] [FUNCTION=3|1B] [GROUP_ID|2B] [CONNECT_ID|1B] [VERIFY_BYTES x (PASSWORD + SALT)|2B] [CURRENT_SIGN_HASH|4B] [HASH|2B] [LOW]`

... *(weiterführende Paketarten analog übersetzen)* ...

* * *

Dieses Netzwerkprotokoll ermöglicht eine sichere und zuverlässige Kommunikation zwischen mehreren Geräten.