# Testmatrix – Klima- und Heizungssteuerung

Stand: 2026-10-12

Diese Datei trennt historische Testergebnisse strikt von aktuellen Regressionstests. Historische Ergebnisse beziehen sich auf frühere Entwicklungsstände und beweisen nicht die Korrektheit der aktuell im Repository abgelegten YAML-Inhalte.

## Statusbegriffe

- **Historisch bestätigt:** Vom Nutzer für einen früheren Stand als erfolgreich bestätigt.
- **Historisch fehlgeschlagen:** In einem früheren Stand beobachteter Fehler.
- **Fehlerhaft dokumentiert:** Frühere Erfolgsaussage wurde korrigiert und gilt nicht als Nachweis.
- **Aktuell offen:** In der aktuellen HA-Installation nicht gesichert getestet.
- **Aktuell bestätigt:** Darf erst nach erneutem Test in der aktuellen HA-Installation vergeben werden.

## Historisch bestätigte Tests

### 02.09.2026 – Kalender-Kollisionen im Arbeitszimmer

| Kombination | Historisches Ergebnis | Einordnung |
|---|---|---|
| Heizkörper-Heizen + Raumklima Kühlen | Kollision erkannt | Historisch bestätigt |
| Heizkörper-Heizen + Raumklima Entfeuchten | Kollision erkannt | Historisch bestätigt |
| Heizkörper-Heizen + Raumklima Heizen | Kombination zugelassen | Historisch bestätigt |
| Heizkörper-Heizen + Raumklima Lüften | Früher als zugelassen und erfolgreich dokumentiert | **Fehlerhaft dokumentiert bzw. nicht verlässlich als Test bestätigt** |

Die aktuelle Sollregel erlaubt Heizkörper-Heizen plus Raumklima-`fan_only` ausdrücklich. Diese fachliche Entscheidung ersetzt jedoch keinen aktuellen Test.

### Anfang September 2026 – System Zeitplanung

Der Nutzer bestätigte die getesteten Syntaxvarianten für Sofortbefehle, `ab`, `bis`, `von ... bis`, Datumsangaben und `für ... danach` als funktionsfähig. Referenzfälle:

```text
klima kühlen az 24 mittel
klima kühlen az 24 mittel ab 18
klima kühlen az 24 mittel bis 22
klima kühlen az 24 mittel von 18 bis 22
klima kühlen az 24 mittel morgen von 18 bis 22
klima kühlen az 24 mittel für 30min danach aus
heizung az 22 für 2h danach aus
```

Auch damalige Tests von `Klima - Kalender` wurden allgemein als erfolgreich bestätigt. Nicht jede Randbedingung wurde einzeln nachgewiesen.

## Historisch beobachtete Fehler

### 31.08.2026 – ältere Klima-Befehlssyntax

```text
klima az kühlen 24 max ab 16:15
→ unbekannte Parameter: ab, 16:15

klima az kühlen 24 max 16:15 für 15min danach lüften auto
→ unbekannter Parameter: 16:15
```

Diese Raum-zuerst-Syntax ist historisch und nicht mehr die Sollsyntax. Die kanonische Reihenfolge beginnt mit dem Modus, beispielsweise `klima kühlen az 24`.

### 04.09.2026 – Heizungsparser

- `heizung alle aus` wurde zunächst nicht erkannt.
- `heizung aus` interpretierte `aus` fälschlich als Raum.
- Die übrigen damals getesteten Befehle wurden als funktionierend bestätigt.
- Eine spätere Parserkorrektur wurde vorgeschlagen; ein eindeutiger erfolgreicher Retest dieser beiden Fälle ist nicht belegt.

## Aktueller Sollstand, noch nicht implementiert

Die Begriffe sind herstellerneutral:

- `raumklima`: Split-Klimageräte beziehungsweise vergleichbare Systeme
- `heizkoerper`: separat steuerbare Heizkörperthermostate
- `innengeraet`: Raumklimagerät im Raum
- `aussengeraet`: technische Gruppe mehrerer Innengeräte

Kernregeln:

- Außengerätegruppen: AZ/WZ sowie SZ/KU.
- Gleiches Innengerät mit überlappenden eigenständigen Aufträgen: immer `conflict`.
- Heizkörperjobs untereinander: immer `allowed`.
- Heizkörper-Heizen plus Raumklima im selben Raum:
  - `heat`, `fan_only`, `off`: `allowed`
  - `cool`, `dry`, `auto`: `conflict`
- Dieselben Kombinationen in unterschiedlichen Räumen:
  - `cool`, `dry`, `auto`: `warning`
  - `heat`, `fan_only`, `off`: `allowed`
- Warnungen blockieren nicht und beginnen mit `⚠️`.
- Raumklima-Modi verschiedener Innengeräte desselben Außengeräts:

| Modus | `heat` | `cool` | `dry` | `fan_only` | `auto` | `off` |
|---|---|---|---|---|---|---|
| `heat` | allowed | conflict | conflict | conflict | conflict | allowed |
| `cool` | conflict | allowed | allowed | allowed | conflict | allowed |
| `dry` | conflict | allowed | allowed | allowed | conflict | allowed |
| `fan_only` | conflict | allowed | allowed | allowed | conflict | allowed |
| `auto` | conflict | conflict | conflict | conflict | conflict | allowed |
| `off` | allowed | allowed | allowed | allowed | allowed | allowed |

`auto + auto` ist bewusst konservativ `conflict`. Verschiedene Außengeräte erzeugen keinen technischen Raumklima-Konflikt.

Der verworfene Entwurf einer pauschalen `fan_only`-Sperre darf nicht implementiert werden.

## Ergebnis der Implementierungsprüfung

`Skripte/Klima Kalender.yaml` besitzt weiterhin zwei fest codierte Prüfblöcke. Noch nicht implementiert sind:

- herstellerneutrale Gerätesysteme und Außengerätegruppen,
- vollständige Modusmatrix,
- Heizkörper-/Raumklima-Warnungen über Raumgrenzen,
- `allowed`/`warning`/`conflict`,
- Prüfung unmittelbarer Befehle und Livezustände,
- automatische und manuelle Überschreibung,
- gerichtete Priorität `heat` über `fan_only`,
- stabile Auftragsgruppen,
- UID-basierte Kalenderänderungen,
- zeitliche Teilung und Fehlerkompensation.

Die produktive Installation wurde auf Home Assistant Core `2026.10.1` und Frontend `20260930.3` aktualisiert. Für den Core-Tag `2026.10.1` wurde auf Quellcodeebene verifiziert: Local Calendar unterstützt intern Create/Update/Delete, normale HA-Script-Actions stellen jedoch weiterhin kein Update/Delete bereit und `calendar.get_events` liefert keine UID. Die hier veröffentlichte produktive Variante verwendet deshalb ausschließlich die vorhandenen UI-Script-Actions und enthält keinen Kalender-Mutationsadapter.

## Aktuell offene allgemeine Nachweise

- produktiver Scheduler-Durchlauf
- Heizungs-Zwischenbestätigung vor der Geräteaktion
- Scheduler-Aufruf mit `acknowledge: false`
- vollständiger Telegram-End-to-End-Test aller Befehle
- Persistenz und Ausführung nach HA-Neustart
- zweite Installation mit abweichenden Entity-IDs
- erneuter Nachweis der historisch fehlerhaften Heizungsbefehle
- zukünftige produktive Kalenderänderungen einschließlich UID-Zugriff, Fehlerkompensation und uneindeutiger Mutationsergebnisse
- sämtliche nachfolgenden neuen Kollisions- und Überschreibregeln

## Aktuelle Regressionstestmatrix

Kein nachfolgender Fall ist allein durch seine Aufnahme als erfolgreich bestätigt. Detaillierte Begründungen stehen in `Dokumentation/KONFLIKTDESIGN.md`.

### T0 – Telegram-Zugriff und Mehrchat-Routing

| ID | Fall | Erwartung | Status |
|---|---|---|---|
| T0-01 | Elternteil 1 sendet aus allowgelistetem Einzelchat | Befehl wird verarbeitet; Primärantwort nur an Elternteil 1 | Aktuell bestätigt am 2026-10-13; nach Einbau der Direktchat-Bedingung durch erneuten Klima-Fachbefehl bestätigt |
| T0-02 | Kind 1 sendet aus anderem allowgelistetem Einzelchat | Befehl wird verarbeitet; Primärantwort nur an Kind 1 | Aktuell offen |
| T0-03 | nicht allowgelisteter privater Chat sendet | keine Verarbeitung durch die Projekt-Automation | Aktuell offen |
| T0-04 | Mitglied schreibt in allowgelisteter Gruppe | keine Verarbeitung durch die Klima-/Heizungsautomation; Gruppe bleibt nur mögliches Ausgabeziel | Nicht praktisch ausführbar: Bot erhält in der aktuellen Telegram-Gruppenkonfiguration keine entsprechenden Nachrichten; HA-Direktchat-Bedingung bleibt unbestätigte zweite Schutzschicht |
| T0-05 | dasselbe Gruppenmitglied schreibt dem Bot privat, Einzelchat nicht allowgelistet | keine Verarbeitung; Gruppenzulassung überträgt sich nicht auf Privatkontakt | Aktuell offen |
| T0-06 | asynchrone Ausgabe deaktiviert, Ziel ist Platzhalter | keine Sendung, kein Platzhalterfehler, strukturiert `skipped` | Aktuell bestätigt am 2026-10-13 |
| T0-07 | asynchrone Ausgabe aktiviert, ein Gruppenziel | Zustellung ausschließlich an konfigurierte Gruppe | Aktuell offen |
| T0-08 | asynchrone kommaseparierte Zielliste mit Leerraum/Dublette | normalisierte einmalige Zustellung pro Ziel | Aktuell offen |
| T0-09 | Elternaufsicht deaktiviert, Ziel ist Platzhalter | nur Primärantwort, kein Platzhalterfehler | Aktuell offen |
| T0-10 | Kind sendet, Elternaufsicht mit einem Ziel aktiviert | Primärantwort an Kind; Aufsichtsmeldung mit Absender, Befehl und Ergebnis an Elternziel | Aktuell offen |
| T0-11 | Elternaufsicht mit mehreren kommaseparierten Zielen | einmalige Meldung an jedes normalisierte Aufsichtsziel | Aktuell offen |
| T0-12 | Aufsichtsziel entspricht Ursprungschat | Ursprung erhält nur Primärantwort, keine Dublette | Aktuell offen |
| T0-13 | optionale Aufsichtsmeldung scheitert nach erfolgreicher Primärsendung | Primärantwort bleibt erfolgreich; Aufsichtsfehler wird diagnostizierbar | Aktuell offen |
| T0-14 | unmittelbarer Heizungsbefehl aus zweitem Einzelchat | sofortige unverbindliche Zwischenmeldung und spätere Endantwort nur an Ursprung | Aktuell offen |
| T0-15 | Scheduler ruft Heizung mit `acknowledge: false` auf | keine Zwischenmeldung | Aktuell offen |
| T0-16 | interner Heizungsaufruf ohne `reply_context` | keine Umleitung der Zwischenmeldung an asynchrones Ziel | Aktuell offen |

### R1 – Raumklima am selben Außengerät

Die vollständige 6×6-Matrix ist für verschiedene Innengeräte desselben Außengeräts in beiden Auftragsrichtungen zu prüfen. Mindestfälle:

| ID | Bestehend | Neu | Räume | Erwartung | Status |
|---|---|---|---|---|---|
| R1-01 | `heat` | `heat` | AZ/WZ | `allowed` | Aktuell offen |
| R1-02 | `heat` | `cool` | AZ/WZ | `conflict` | Aktuell offen |
| R1-03 | `cool` | `heat` | AZ/WZ | `conflict` | Aktuell offen |
| R1-04 | `heat` | `fan_only` | AZ/WZ | `conflict` | Aktuell offen |
| R1-05 | `fan_only` | `heat` | AZ/WZ | `conflict` | Aktuell offen |
| R1-06 | `cool` | `dry` | AZ/WZ | `allowed` | Aktuell offen |
| R1-07 | `dry` | `fan_only` | AZ/WZ | `allowed` | Aktuell offen |
| R1-08 | `auto` | `auto` | AZ/WZ | `conflict` | Aktuell offen |
| R1-09 | `off` | aktiver Modus | AZ/WZ | `allowed` | Aktuell offen |
| R1-10 | `heat` | `cool` | SZ/KU | `conflict` | Aktuell offen |
| R1-11 | `heat` | `cool` | AZ/SZ | `allowed` | Aktuell offen |
| R1-12 | beliebig | beliebig | gleiches Innengerät | `conflict` | Aktuell offen |
| R1-13 | unbekannter Modus | aktiver Modus | gleiches Außengerät | `conflict` mit Diagnose | Aktuell offen |

R1-04 und R1-05 sind die ausdrücklich geforderten Reihenfolgetests Heizen/Lüften.

### R2 – Heizkörper untereinander

| ID | Fall | Erwartung | Status |
|---|---|---|---|
| R2-01 | zwei Heizkörperjobs im selben Raum | `allowed` | Aktuell offen |
| R2-02 | zwei Heizkörperjobs in verschiedenen Räumen | `allowed` | Aktuell offen |

### R3 – Heizkörper und Raumklima im selben Raum

Jede Kombination in beiden Richtungen:

| ID | Raumklima-Modus | Erwartung | Status |
|---|---|---|---|
| R3-01 | `heat` | `allowed` | Aktuell offen |
| R3-02 | `fan_only` | `allowed` | Aktuell offen |
| R3-03 | `cool` | `conflict` | Aktuell offen |
| R3-04 | `dry` | `conflict` | Aktuell offen |
| R3-05 | `auto` | `conflict` | Aktuell offen |
| R3-06 | `off` | `allowed` | Aktuell offen |

### R4 – Heizkörper und Raumklima in verschiedenen Räumen

Jede Kombination in beiden Richtungen und sowohl innerhalb derselben als auch zwischen verschiedenen Außengerätegruppen:

| ID | Raumklima-Modus | Erwartung | Status |
|---|---|---|---|
| R4-01 | `cool` | `warning`, Auftrag zulässig | Aktuell offen |
| R4-02 | `dry` | `warning`, Auftrag zulässig | Aktuell offen |
| R4-03 | `auto` | `warning`, Auftrag zulässig | Aktuell offen |
| R4-04 | `heat` | `allowed` | Aktuell offen |
| R4-05 | `fan_only` | `allowed` | Aktuell offen |
| R4-06 | `off` | `allowed` | Aktuell offen |

Jede Warnantwort muss mit `⚠️` beginnen.

### R5 – Prüfquellen

| ID | Bestehend | Neu | Erwartung | Status |
|---|---|---|---|---|
| R5-01 | Kalenderjob | Kalenderjob | Matrixentscheidung | Aktuell offen |
| R5-02 | Job desselben Batches | Job | identische Matrixentscheidung | Aktuell offen |
| R5-03 | Reservierung | unmittelbarer Befehl | identische Matrixentscheidung | Aktuell offen |
| R5-04 | live aktiver Konfliktmodus | unmittelbarer Befehl ohne Force | blockiert; Hinweis nennt Problem und Konfliktgerät | Aktuell offen |
| R5-05 | live aktiver Konfliktmodus | unmittelbarer Befehl mit Force | Konfliktgerät auf `off`, danach neuer Befehl | Aktuell offen |
| R5-06 | mehrere Warnungen | Auftrag | alle Warnungen, Auftrag zulässig | Aktuell offen |
| R5-07 | mehrere Konflikte | Batch | nichts teilweise angelegt | Aktuell offen |

### R6 – Automatisches Überschreiben

| ID | Automatisch | Bestehend | Neu | Scope | Erwartung | Status |
|---|---:|---|---|---|---|---|
| R6-01 | aus | `fan_only` | `heat` | gleiches Innengerät | blockiert | Aktuell offen |
| R6-02 | an | `fan_only` | `heat` | gleiches Innengerät | ersetzt | Aktuell offen |
| R6-03 | an | `fan_only` | `heat` | gleiches Außengerät | ersetzt | Aktuell offen |
| R6-04 | an | `fan_only` | `heat` | verschiedene Außengeräte | keine Verdrängung | Aktuell offen |
| R6-05 | an | `heat` | `fan_only` | gleiches Innengerät | blockiert, nicht ersetzt | Aktuell offen |
| R6-06 | an | Heizkörper-Heizen | Raumklima `fan_only` | gleicher Raum | `allowed`, keine Ersetzung | Aktuell offen |

### R7 – Manuelles Überschreiben

| ID | Manuell | Parameter | Erwartung | Status |
|---|---:|---|---|---|
| R7-01 | aus | `-überschreiben` | klare Ablehnung | Aktuell offen |
| R7-02 | an | keiner | Konflikt bleibt blockiert | Aktuell offen |
| R7-03 | an | `-überschreiben` | technisch mutierbare Konflikte ersetzt | Aktuell offen |
| R7-04 | an | `-force` | identisches Verhalten | Aktuell offen |
| R7-05 | an | gesetzt bei `warning` | zulässig, keine Mutation | Aktuell offen |
| R7-06 | an | gesetzt bei Fremdereignis | keine Mutation des Fremdereignisses | Aktuell offen |
| R7-07 | an | Token nur als Wortbestandteil | nicht als Force erkannt | Aktuell offen |
| R7-08 | an | Force gegen live aktives Konfliktgerät | Gerät auf `off`, neuer Befehl ausgeführt | Aktuell offen |
| R7-09 | an | Force gegen Livezustand und eigenen Kalenderjob | Livegerät aus und Kalenderjob konsistent ersetzt | Aktuell offen |
| R7-10 | an | Konfliktgerät aus, neuer Gerätebefehl scheitert | Konfliktgerät bleibt aus; Zielgerät wird ebenfalls auf `off` gesetzt; deutliche Meldung | Aktuell offen |
| R7-11 | an | zusätzliches Sicherheitsabschalten des Zielgeräts scheitert | harte Fehlermeldung nennt den unbestätigten Zielzustand | Aktuell offen |

### R8 – Zeitliche Überdeckung

| ID | Alt | Neu | Erwartung | Status |
|---|---|---|---|---|
| R8-01 | 18–22 | 18–22 | vollständig ersetzt | Aktuell offen |
| R8-02 | 18–22 | 17–23 | vollständig ersetzt | Aktuell offen |
| R8-03 | 18–22 | 19–23 | linker Rest 18–19 | Aktuell offen |
| R8-04 | 18–22 | 17–20 | rechter Rest 20–22 | Aktuell offen |
| R8-05 | 18–22 | 19–21 | Reste 18–19 und 21–22 | Aktuell offen |
| R8-06 | 18–22 | 22–23 | keine Überlappung | Aktuell offen |

### R9 – Auftragsgruppen, Fehler und Fremdereignisse

| ID | Fall | Erwartung | Status |
|---|---|---|---|
| R9-01 | Primärjob vollständig entfernt | Folgeaktion ebenfalls entfernt | Aktuell offen |
| R9-02 | Kürzung ohne rechten Rest | alte Folgeaktion entfernt | Aktuell offen |
| R9-03 | Teilung mit rechtem Rest | Folgeaktion genau einmal erhalten | Aktuell offen |
| R9-04 | mehrere Konflikte | konsistenter Gesamtplan | Aktuell offen |
| R9-05 | Update schlägt fehl | Kompensation und harte Fehlermeldung | Aktuell offen |
| R9-06 | rechter Rest kann nicht erstellt werden | kein stiller Datenverlust | Aktuell offen |
| R9-07 | Kompensation schlägt ebenfalls fehl | keine Geräteaktion; Operations-ID und Administratorhinweis | Aktuell offen |
| R9-08 | Ereignis ohne Projektmetadaten | niemals automatisch mutieren | Aktuell offen |
| R9-09 | manueller/projektfremder Termin | geschützt | Aktuell offen |
| R9-10 | parallele Befehle | zentrale Serialisierung | Aktuell offen |
| R9-11 | Neustart während Teilmutation | Zustand erkannt und gemeldet | Aktuell offen |
| R9-12 | Core `2026.10.1`: UI-Script-Actions | kein natives Update/Delete; `calendar.get_events` ohne UID | Quellcode bestätigt |

### R10 – Historische Zeitplanung erneut prüfen

Alle historisch bestätigten Referenzbefehle erneut in der aktuellen HA-Installation ausführen. Zusätzlich prüfen:

- ungültige Uhrzeit,
- vergangener absoluter Zeitpunkt,
- Datum ohne Startzeit,
- `für` ohne Folgeaktion,
- `danach` ohne definierten Endzeitpunkt,
- unvereinbare Kombination von `bis`/`von … bis` mit `für`,
- Force-Tokens an zulässigen Positionen, sobald implementiert.

### R11 – Heizungsparser erneut prüfen

```text
heizung alle aus
heizung aus
heizung 22
heizung az 22
heizung az 22 für 2h danach aus
```

Die ersten beiden Fälle sind gezielte Regressionen der historischen Fehler.

### R12 – Scheduler und Benachrichtigungen

1. Raumklima-Job startet in der vorgesehenen Minute.
2. Heizkörperjob startet in der vorgesehenen Minute.
3. Scheduler übergibt bei Heizkörperjobs `acknowledge: false`.
4. Scheduler-Heizungsjob erzeugt keine Zwischenbestätigung.
5. Erfolgsmeldung respektiert `input_boolean.klima_scheduler_notifications`.
6. Fehlermeldungen werden unabhängig vom Helper gesendet.
7. Geplanter Job wird nach HA-Neustart weiterhin ausgeführt.
8. Metadatenzeile wird nie als Teil des Gerätebefehls ausgeführt.

## Dokumentationsregel

Nach jedem aktuellen Test sind mindestens Datum, HA-Core-Version, getesteter Befehl, Vorzustand, erwartetes Ergebnis, tatsächliches Ergebnis und Status zu protokollieren. Nur tatsächlich beobachtete Erfolge erhalten **Aktuell bestätigt**.