# Home-Assistant Klima- und Heizungssteuerung

Messengerbasierte Steuerung von Raumklimageräten und Heizkörperthermostaten in
Home Assistant. Die aktuelle Referenzinstallation verwendet Telegram als
Bedienoberfläche, Local Calendar als persistente Auftragswarteschlange und über
die Home-Assistant-Oberfläche gepflegte Automationen und Skripte.

Das Projekt ist aktiv in einer Home-Assistant-Installation im Einsatz. Es bleibt
ein installationsspezifischer Prototyp: Entity-IDs, Räume und Geräte müssen vor
der Verwendung an die eigene Installation angepasst und sämtliche Funktionen
lokal getestet werden.

Die Entwicklung wurde in erheblichem Umfang durch ChatGPT und Codex unterstützt.
Idee, Anforderungen, fachliche Entscheidungen, Prüfung und Verantwortung liegen
beim Projektinhaber. Eine ausführliche und kritische Einordnung steht im
[`Manifest zum Einsatz künstlicher Intelligenz`](KI-MANIFEST.md).

## Funktionsumfang

- Klima- und Heizungsbefehle per Telegram
- Sofortbefehle und zeitgesteuerte Aufträge
- Statusabfragen
- zentrale Raum-, Alias- und Entity-Zuordnung
- Local-Calendar-basierte Jobwarteschlange
- minütlicher Scheduler
- Klima-Modi `cool`, `heat`, `fan_only`, `dry`, `auto` und `off`
- konfigurierbare Lüfterstufen
- getrennte Messengeradapter und Fachlogik
- Antworten an den privaten Ursprungschat
- standardmäßig deaktivierte asynchrone Telegram-Zielliste

## Architektur

```text
Telegram
  → Telegram Eingang
  → System Message-Router
      ├── System Status
      ├── Klima Router
      │   ├── Klima Status
      │   └── Klima Steuerung
      └── Heizung Steuerung

Klima Steuerung / Heizung Steuerung
  → System Raum Mapping
  → System Zeitplanung
  → Klima Kalender
  → Home-Assistant-Geräte

Klima Scheduler
  → Klima Steuerung / Heizung Steuerung

Asynchrone Nachrichten
  → System Nachricht
  → Telegram Ausgang
```

Die Fachlogik ist vom Messenger getrennt. Weitere Messenger können später durch
zusätzliche Ein- und Ausgangsadapter ergänzt werden.

## Referenzumgebung

Zuletzt praktisch verwendet und teilweise regressionsgetestet mit:

- Home Assistant Container
- Home Assistant Core `2026.10.1`
- Frontend `20260930.3`

Historische und aktuelle Testnachweise stehen in
[`Dokumentation/TESTMATRIX.md`](Dokumentation/TESTMATRIX.md). Historische
Testergebnisse beweisen nicht automatisch die Funktion des aktuellen Stands.

## Voraussetzungen

- laufende Home-Assistant-Installation
- über die UI eingerichtete Telegram-Bot-Integration
- Local Calendar mit der Entity-ID `calendar.klima`
- Helper `input_boolean.klima_scheduler_notifications`
- passende `climate`-, Temperatur- und Feuchtigkeit-Entities
- Administratorzugriff zum Anlegen und Bearbeiten der Skripte und Automationen

## Installation

Dieses Repository ist kein direkt deploybares Home-Assistant-Konfigurationsverzeichnis.
Die YAML-Dateien sind einzeln exportierte UI-Objekte und werden über
**Einstellungen → Automationen & Szenen** angelegt beziehungsweise aktualisiert.

### 1. Telegram-Bot einrichten

1. Telegram-Bot über BotFather anlegen.
2. In Home Assistant unter **Einstellungen → Geräte & Dienste** die Integration
   **Telegram bot** hinzufügen.
3. Bot-Token ausschließlich in der Integration hinterlegen.
4. Jeden privaten Chat, der Befehle senden darf, über **Zulässige Chat ID
   hinzufügen** freigeben.
5. Gruppen nur bei Bedarf als Ausgabeziel freigeben.

`Telegram Eingang` akzeptiert ausschließlich private Einzelchats. Zusätzlich zur
Integration-Allowlist prüft die Automation ohne fest gespeicherte IDs:

```jinja2
trigger.event.data.chat_id | string
== trigger.event.data.user_id | string
```

Damit bleiben allowgelistete Gruppen als Ausgabeziele möglich, werden aber als
Befehlseingang abgewiesen. Für reine Ausgabegruppen wird empfohlen, Telegram Bot
Privacy aktiviert zu lassen und dem Bot keine unnötigen Administratorrechte zu
geben.

### 2. Kalender und Helper anlegen

- Local Calendar mit Entity-ID `calendar.klima`
- Toggle-Helper mit Entity-ID `input_boolean.klima_scheduler_notifications`

Der Helper steuert Erfolgsmeldungen des Schedulers. Fehlermeldungen können davon
unabhängig erzeugt werden.

### 3. Skripte anlegen

Die elf Dateien unter [`Skripte/`](Skripte/) als einzelne Skripte übernehmen.
Die erwarteten Entity-IDs sind:

```text
script.heizung_steuerung
script.klima_kalender
script.klima_router
script.klima_status
script.klima_steuerung
script.system_message_router
script.system_nachricht
script.system_raum_mapping
script.system_status
script.system_zeitplanung
script.telegram_ausgang
```

Home Assistant kann aus dem Alias abweichende Entity-IDs erzeugen. In diesem Fall
müssen die Aufrufe konsistent angepasst oder die Skript-Entity-IDs entsprechend
umbenannt werden.

### 4. Räume und Geräte anpassen

In [`Skripte/System Raum Mapping.yaml`](Skripte/System%20Raum%20Mapping.yaml)
müssen mindestens angepasst werden:

- Räume und Raumkürzel,
- Aliase,
- Temperatur- und Feuchtigkeitssensoren,
- Raumklimageräte,
- Heizkörperthermostate,
- Kalender-Entity,
- unterstützte Modi und Lüfterstufen.

Die mitgelieferten Entity-IDs stammen aus der Referenzinstallation und werden in
anderen Installationen in der Regel nicht existieren.

### 5. Automationen anlegen

Die beiden Dateien unter [`Automationen/`](Automationen/) übernehmen:

```text
Telegram Eingang
Klima Scheduler
```

Anschließend beide Objekte zunächst kontrolliert aktivieren und die Testfälle aus
[`Dokumentation/TESTMATRIX.md`](Dokumentation/TESTMATRIX.md) verwenden.

### 6. Asynchrone Telegram-Ausgabe optional aktivieren

Direkte Antworten verwenden immer die Laufzeit-Chat-ID des privaten
Ursprungschats. Zielose Scheduler- und Systemmeldungen sind in der öffentlichen
Fassung dagegen sicher deaktiviert:

```yaml
asynchronous_notifications_enabled: false
asynchronous_targets_csv: "__TELEGRAM_ASYNC_CHAT_IDS__"
```

Für die lokale Installation kann in `Telegram Ausgang` aktiviert werden:

```yaml
asynchronous_notifications_enabled: true
asynchronous_targets_csv: "CHAT_ID_1, CHAT_ID_2"
```

Alle Ziele müssen in der Telegram-Integration allowgelistet sein. Reale IDs
dürfen nicht in einen Commit übernommen werden. Die kommaseparierte Liste wird
normalisiert; leere und doppelte Einträge werden entfernt.

## Befehle

### Hilfe und Systemstatus

```text
hilfe
status
status detail
```

### Klimastatus

```text
klima
klima status
klima status az
klima status extern
klima kalender
```

### Klimasteuerung

```text
klima kühlen az 24
klima kühlen az 24 max
klima heizen wz 22
klima lüften sz auto
klima entfeuchten ku
klima aus az
```

Ohne Raum wird der Befehl auf alle konfigurierten Klimageräte angewendet:

```text
klima kühlen 24
klima heizen 22
```

### Heizungssteuerung

```text
heizung az 22
heizung ba 23
heizung wc 22
heizung az aus
```

Bei Heizkörpern bedeutet `aus` in der Referenzkonfiguration 4 °C Frostschutz.

`klima heizen ba ...` und `klima heizen wc ...` werden an die
Heizungssteuerung delegiert, weil dort keine separaten Raumklimageräte
konfiguriert sind.

### Zeitplanung

```text
klima kühlen az 24 ab 18 Uhr
klima kühlen az 24 bis 20 Uhr
klima kühlen az 24 von 18 bis 20 Uhr
klima kühlen az 24 morgen von 18 bis 20 Uhr
klima kühlen az 24 für 30min danach aus
heizung az 22 für 2h danach aus
```

Weitere unterstützte Datums- und Zeitvarianten sind in den Skripten und der
Testmatrix dokumentiert.

## Sicherheit und vertrauliche Werte

Das Repository enthält absichtlich keine realen:

- Telegram-Bot-Tokens,
- Telegram-Chat- oder Benutzer-IDs,
- Passwörter,
- API-Schlüssel,
- privaten Schlüssel.

Bot-Token und Chat-Allowlist werden in der Telegram-Bot-Integration gepflegt.
Lokale `secrets.yaml`-, Credential- und Umgebungsdateien werden durch
[`.gitignore`](.gitignore) ausgeschlossen.

Vor jedem Commit sollten YAML, Traces, Logs, Screenshots und Dokumentation erneut
auf vertrauliche Werte geprüft werden.

## Aktuelle Grenzen

- Die Konfiguration ist noch stark auf die Referenzinstallation zugeschnitten.
- Die aktuelle produktive Kollisionsprüfung bildet noch nicht das vollständige
  Fachdesign aus [`Dokumentation/KONFLIKTDESIGN.md`](Dokumentation/KONFLIKTDESIGN.md)
  ab.
- Elternaufsicht, frei konfigurierbare zentrale Gerätegruppen und manuelles
  Überschreiben sind geplant, aber nicht Bestandteil dieses Stands.
- Die Heizungs-Zwischenmeldung besitzt noch keinen messengerneutralen
  Ursprungskontext. Bei deaktivierter asynchroner Ausgabe wird sie übersprungen.
- Zielose Scheduler-Nachrichten werden standardmäßig nicht gesendet.
- Nicht jede historische Syntax oder Randbedingung ist im aktuellen Stand erneut
  getestet.
- Der experimentelle Kalenderadapter ist nicht Bestandteil dieser
  Veröffentlichung und wird von der produktiven Variante nicht verwendet.

## Dokumentation

- [`Dokumentation/TESTMATRIX.md`](Dokumentation/TESTMATRIX.md) – historische und
  aktuelle Tests
- [`Dokumentation/KONFLIKTDESIGN.md`](Dokumentation/KONFLIKTDESIGN.md) – noch
  nicht vollständig umgesetztes Fachdesign
- [`Dokumentation/SECRETS_KONZEPT.md`](Dokumentation/SECRETS_KONZEPT.md) –
  Telegram- und Secret-Konzept
- [`Dokumentation/CONTRIBUTOR_KONZEPT.md`](Dokumentation/CONTRIBUTOR_KONZEPT.md)
  – Contributor- und Dual-Licensing-Konzept
- [`KI-MANIFEST.md`](KI-MANIFEST.md) – transparenter und kritischer Umgang mit
  KI-Unterstützung in diesem Projekt

Weitere Installations-, Einrichtungs- und Bedienungsanleitungen sollen unter
`Dokumentation/` ergänzt werden.

## Beiträge

Issues für Fehlerberichte, Fragen und Vorschläge sind willkommen. Bitte vor
größeren Arbeiten zuerst ein Issue eröffnen. Pull Requests werden in der ersten
Projektphase nur nach vorheriger Abstimmung beziehungsweise Einladung angenommen.

Details stehen in [`CONTRIBUTING.md`](CONTRIBUTING.md). Vor dem ersten Merge ist
die Zustimmung zu [`CLA.md`](CLA.md) erforderlich. Beitragende werden nach ihrer
gewünschten öffentlichen Bezeichnung in
[`CONTRIBUTORS.md`](CONTRIBUTORS.md) genannt.

## Lizenz

Copyright © 2026 René Reichelt

Nichtkommerzielle Nutzung ist unter
[Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](LICENSE)
gestattet. Änderungen und Weiterveröffentlichungen müssen die Lizenzbedingungen,
Namensnennung und Share-Alike-Vorgaben beachten.

Kommerzielle Nutzung ist von der öffentlichen Lizenz nicht umfasst und erfordert
eine separate schriftliche Vereinbarung. Weitere Informationen stehen in
[`COMMERCIAL-LICENSE.md`](COMMERCIAL-LICENSE.md). Anfragen können an
[lizenz@rrsb.de](mailto:lizenz@rrsb.de) gerichtet werden.

Dieses Repository ist wegen der NonCommercial-Bedingung öffentlich einsehbar und
weiterverwendbar, aber nicht Open Source im Sinne der Open Source Initiative.

## Projekt unterstützen

In Konzeption, Implementierung, Tests und Dokumentation dieses Projekts sind
bereits mehr als 50 Stunden privater Entwicklungszeit eingeflossen.

Wenn dir das Projekt hilft und du seine Weiterentwicklung freiwillig unterstützen
möchtest, kannst du mir über PayPal einen Kaffee ausgeben:

[Projekt freiwillig über PayPal unterstützen](https://www.paypal.me/RReichelt0)

Die Unterstützung ist freiwillig. Sie gewährt keinen Anspruch auf Support,
bestimmte Funktionen, Gewährleistung oder eine kommerzielle Lizenz.

## Haftungshinweis

Automatisierungen können reale Heizungs- und Klimageräte schalten. Prüfe alle
Entity-IDs, Temperaturgrenzen, Zeitpläne und Sicherheitsfunktionen vor der
Aktivierung. Die bereitgestellten Inhalte werden ohne Gewährleistung angeboten;
maßgeblich sind die Bedingungen in `LICENSE`.
