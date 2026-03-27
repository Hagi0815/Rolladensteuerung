# Rolladensteuerung

Modul für IP-Symcon ab Version 8.0. Autor: Christian Hagedorn

Steuert einen Rollladen, eine Markise oder eine ähnliche Abdunkelungseinrichtung nach konfigurierbaren Regeln.

## Inhaltsverzeichnis

1. [Funktionsumfang](#1-funktionsumfang)
2. [Voraussetzungen](#2-voraussetzungen)
3. [Installation](#3-installation)
4. [Funktionsreferenz](#4-funktionsreferenz)
5. [Konfiguration](#5-konfiguration)
6. [Statusvariablen](#6-statusvariablen)
7. [Steuerungslogik](#7-steuerungslogik)
8. [Anhang](#8-anhang)

---

## 1. Funktionsumfang

- Hoch-/Runterfahren zu vorgegebenen Zeiten (Wochenplan)
- Tagerkennung über IsDay-Variable oder Helligkeitsvergleich
- **Getrennte Quellenauswahl für morgens und abends:** Wochenplan oder IsDay – je Richtung exklusiv einstellbar
- Urlaubs- und Feiertagsberücksichtigung
- Sonnenschutz inkl. Nachführen nach Sonnenstand (einfach und präzise Variante)
- Beschattung nach Helligkeit
- Kontakte zum Öffnen des Rollladens mit **Integer-Modus** (drei Zustände: geschlossen / gekippt / geöffnet)
- Kontakte nur aktiv während der Nachtphase (zwischen Abend-Schließen und Morgen-Öffnen)
- Beim automatischen Schließen wird die Kontaktposition (z.B. Fenster gekippt) berücksichtigt
- Kontakte zum Schließen des Rollladens
- **Lichtsteuerung** bei Kontaktzustandsänderung (Boolean, String oder Float-Variable schaltbar, optional mit Freigabe-Variable)
- Notfall-Kontakt (öffnet sofort, Automatik bleibt aktiv)
- Erkennung manueller Bedienung mit konfigurierbarer Sperrzeit
- Verzögerung bei Tag/Nacht-Wechsel (optional zufällig)
- Aktivierung/Deaktivierung über Statusvariable (nur durch Benutzer oder externe Boolean-Variable)
- Sofortige Statusmeldung bei Auslösung (Auslöser in LAST_MESSAGE) und bei manueller Bedienung
- Herstellerunabhängig (alle Aktoren mit Statusvariable + RequestAction)
- Automatische Profilrichtungserkennung (0=zu/1=auf und 0=auf/100=zu werden korrekt behandelt)

---

## 2. Voraussetzungen

- IP-Symcon ab Version 8.0
- Aktor mit einer Statusvariable vom Typ Integer oder Float
- Die Statusvariable muss über `RequestAction` steuerbar sein (nicht emuliert)
- Geeignete Profildarstellung: „Rolladen" oder „Legacy Profil" mit korrektem Min/Max
- Das Modul erkennt die Profilrichtung automatisch anhand der Profilassoziationen

---

## 3. Installation

### 3.1 Modul laden

Das Modul über die Modulverwaltung in IP-Symcon installieren (Bibliothek hinzufügen über die GUID `{153CE11A-48A3-48F4-A022-140A3F7509DB}`).

### 3.2 Rollladeninstanz anlegen

Im Objektbaum `Instanz hinzufügen` → `Rolladensteuerung` suchen. Pro Rollladen wird eine Instanz angelegt.

### 3.3 Rollladen prüfen

Vor der Konfiguration sicherstellen, dass der Rollladen korrekt in Symcon eingerichtet ist:
- Positionsvariable mit adaptivem Icon (z.B. „Jalousie") im Webfront prüfen
- Geöffnet und Geschlossen müssen korrekt dargestellt werden

![image](docs/Rollladen_geöffnet.jpg)
![image](docs/Rollladen_geschlossen.jpg)

---

## 4. Funktionsreferenz

```php
BLC_ControlBlind(int $InstanceID, bool $considerDeactivationTimes): bool
```
Führt einen vollständigen Steuerungslauf durch. `$considerDeactivationTimes = true` berücksichtigt die konfigurierte Sperrzeit nach automatischer Bewegung.

---

## 5. Konfiguration

### 5.1 Wochenplan

Für die Grundfahrzeiten ist ein Wochenplan-Ereignis in IP-Symcon anzulegen:

![image](docs/Wochenplan.jpg)

**Wichtig:**
- Genau zwei Aktionen mit IDs **1** (Tag / Rollladen auf) und **2** (Nacht / Rollladen zu) anlegen
- Die Aktionen selbst bleiben ohne Funktion (leerer PHP-Code)
- Maximal ein Zeitpunkt für Aktion 1 (Auffahrzeit) pro Gruppe
- IP-Symcon setzt zu Mitternacht automatisch einen Startzustand (Aktion 2) – das ist korrekt

Der Wochenplan ist **Pflicht** – auch wenn ausschließlich IsDay verwendet wird. In diesem Fall genügt ein Wochenplan mit einer 24-Stunden-Zeitspanne für Aktion 2.

### 5.2 Tagerkennung (optional)

Zwei Möglichkeiten:
1. **IsDay-Variable** – z.B. vom Location-Modul
2. **Helligkeitsvergleich** – Helligkeitsvariable + Schwellwertvariable (optional mit Durchschnitt über n Minuten, erfordert Archivierung)

![image](docs/Helligkeitsschwellwert.jpg)

Übersteuernde feste Tagesstart-/Tagesendezeiten können zusätzlich als String-Variablen (Format `HH:MM`) angegeben werden.

### 5.3 Quellenauswahl morgens / abends

Für jede Richtung (Auffahren morgens, Zufahren abends) wird die Steuerquelle **exklusiv** festgelegt:

| Einstellung | Verhalten |
|---|---|
| **Wochenplan** | Nur der Wochenplan entscheidet. IsDay/Helligkeit wird für diese Richtung komplett ignoriert. |
| **IsDay / Helligkeit** | Nur der Sensor entscheidet. Der Wochenplan wird für diese Richtung komplett ignoriert. |

Wenn beide Richtungen aktiv sind, gilt: **Schließen hat Vorrang**. Wenn also abends IsDay=false den Rollladen geschlossen hat, fährt er morgens erst auf wenn die Morgen-Quelle „Tag" signalisiert.

### 5.4 Beschattung nach Sonnenstand (optional)

Benötigt: Azimuth-Variable und Altitude-Variable (z.B. vom Location-Modul), Azimuth-Bereich (von/bis).

Optional: Helligkeitssensor und Schwellwert (Beschattung nur bei ausreichender Helligkeit), Temperaturvariable (Hitzeschutz: bei >27 °C +15 %, bei >30 °C auf 90 % geschlossen).

**Zwei Berechnungsvarianten:**
- **Einfach** (nur Fassadenfenster): Zwei Behanghöhen bei zwei Sonnenhöhen angeben → lineare Interpolation
- **Präzise** (auch Dachfenster): Fensterausrichtung, Neigung, Höhe, Brüstungshöhe und maximale Eindringtiefe der Sonne angeben

### 5.5 Beschattung nach Helligkeit (optional)

Bis zu zwei Helligkeitsschwellwerte mit je einer Rollladenposition. Bei Überschreitung wird der Rollladen auf die zugehörige Position gefahren. Steuerbar über eine Aktivierungsvariable.

Wenn beide Beschattungsarten aktiv sind, gilt der restriktivere Wert (weiter geschlossen).

### 5.6 Kontakte zum Öffnen (optional)

Bis zu zwei Kontakte. Die Kontakte sind **nur während der Nachtphase aktiv** (zwischen dem abendlichen Schließen und dem morgendlichen Öffnen). Beim automatischen Schließen wird die aktuelle Kontaktstellung berücksichtigt.

**Integer-Modus** (Checkbox aktivieren): Der Kontakt kann drei Zustände abbilden:

| Zustand | Bedingung | Rollladenposition |
|---|---|---|
| Geschlossen | Wert < Kipp-Schwelle | normale Steuerung |
| Gekippt | Wert ≥ Kipp-Schwelle | Position bei gekippt |
| Geöffnet | Wert ≥ Öffnen-Schwelle | Position bei geöffnet |

Beim automatischen Schließen wird der Rollladen auf die zur aktuellen Kontaktstellung passende Position gefahren – nicht auf die normale Nachtposition.

### 5.7 Lichtsteuerung bei Kontakten (optional)

Für Kontakt 1 und 2 können bei jedem der drei Zustände (geschlossen / gekippt / geöffnet) Variablen geschaltet werden. Die Lichtsteuerung wird **nur bei Zustandsänderung** ausgeführt.

Pro Stellung konfigurierbar:
- **Aktiv** (Checkbox): Lichtsteuerung für diesen Zustand einschalten
- **Variable**: Ziel-Variable (Boolean, String oder Float)
- **Typ**: Boolean / String / Float
- **Wert**: Der zu setzende Wert – bei String-Variablen mit Profil aus dem Dropdown wählbar

**Freigabe-Variable** (optional): Lichtsteuerung des gesamten Kontakts nur aktiv wenn diese Boolean-Variable `true` ist.

### 5.8 Kontakte zum Schließen (optional)

Bis zu zwei Kontakte. Solange ein Kontakt aktiv ist, wird der Rollladen auf maximal die konfigurierte Maximalhöhe begrenzt. Auch nur während der Nachtphase aktiv.

Priorität zwischen Öffnen- und Schließen-Kontakten konfigurierbar.

### 5.9 Notfall-Kontakt (optional)

Bei aktivem Notfall-Kontakt fährt der Rollladen sofort auf. Die Automatik **bleibt aktiv** – Deaktivierung nur durch manuellen Eingriff. Der Notfall-Kontakt ist immer aktiv, unabhängig von der Tagesphase.

### 5.10 Experteneinstellungen

| Parameter | Standard | Beschreibung |
|---|---|---|
| DeactivationAutomaticMovement | 20 Min | Sperrzeit nach automatischer Bewegung (verhindert zu häufiges Fahren) |
| DeactivationManualMovement | 120 Min | Sperrzeit nach manueller Bedienung; 0 = bis zum nächsten Tag/Nacht-Wechsel |
| MinMovement | 5 % | Mindestabweichung ab der eine Bewegung ausgeführt wird |
| MinMovementAtEndPosition | 2,5 % | Mindestabweichung für Endpositionen |
| ShowNotUsedElements | false | Nicht konfigurierte Felder im Formular einblenden |

---

## 6. Statusvariablen

| Variable | Typ | Beschreibung |
|---|---|---|
| `ACTIVATED` | Boolean | Aktiviert/Deaktiviert die automatische Steuerung. Beim Einschalten werden manuelle Eingriffe zurückgesetzt. Kann nur durch den Benutzer oder eine externe Boolean-Variable geändert werden. |
| `LAST_MESSAGE` | String | Wird sofort bei Auslösung beschrieben (Auslöser + „Steuerung läuft..."), danach mit dem Ergebnis aktualisiert. Format: `HH:MM:SS | Grund | Öffnung: XX% | Auslöser: ...`. Auch bei manueller Bedienung sofort aktualisiert. Archivierung empfohlen. |

---

## 7. Steuerungslogik

### Auslöser

Das Modul reagiert **nicht zyklisch** sondern ausschließlich auf Ereignisse:
- Wochenplan-Schaltpunkte
- Änderung der IsDay-Variable oder Helligkeitssensor
- Änderung von Kontakten (Öffnen, Schließen, Notfall)
- Änderung der Beschattungs-Aktivatoren
- Verzögerungstimer (für konfigurierten Tag/Nacht-Wechsel-Verzögerung)

### Prioritätskette (höchste zuerst)

1. **Notfall-Kontakt** – fährt sofort auf, immer aktiv
2. **Öffnen-Kontakte / Schließen-Kontakte** – nur während Nachtphase aktiv
3. **Beschattung** (Sonne / Helligkeit) – nur tagsüber
4. **Morgen/Abend-Steuerung** – je nach konfigurierter Quellenauswahl

### Quellenlogik

Morgens und abends haben je eine exklusive Quelle. Wenn abends IsDay den Rollladen geschlossen hat, bleibt er geschlossen bis die Morgen-Quelle „Tag" signalisiert – der Wochenplan kann ihn in diesem Fall nicht auffahren.

---

## 8. Anhang

### GUID

| Modul | Typ | GUID |
|:---:|:---:|:---:|
| Rolladensteuerung | Device | `{C588F944-0032-411C-9848-3638D019CDB8}` |
| Bibliothek | Library | `{153CE11A-48A3-48F4-A022-140A3F7509DB}` |
