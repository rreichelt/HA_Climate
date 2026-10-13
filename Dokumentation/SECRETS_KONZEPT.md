# Konzept – Telegram-Zugangsdaten und veröffentlichungsfähiges Repository

Stand: 2026-10-13

## Ziel

Die über die Home-Assistant-Oberfläche gepflegten Automationen und Scripts sollen ohne Telegram-Token, Chat-ID oder Benutzer-ID im Repository veröffentlicht werden können. Die Lösung soll möglichst ohne weitere Custom Integration auskommen und den UI-Workflow erhalten.

## Verifizierte Rahmenbedingungen

Für Home Assistant Core `2026.10.1` ist offiziell dokumentiert:

1. `!secret` wird im UI-YAML-Editor für Automationen und Scripts nicht unterstützt.
2. Werden Secrets in `automations.yaml` oder `scripts.yaml` verwendet, können die betreffenden YAML-Objekte nicht normal in der UI bearbeitet werden; separat eingebundene Objekte sind dort nur lesbar.
3. Secret-Werte in Automationen können Administratoren außerdem in YAML-Ansichten und Traces offengelegt werden.
4. Die Telegram-Bot-Integration verwaltet erlaubte Chat-IDs als UI-konfigurierte Subentries.
5. Nur allowgelistete Chats dürfen über den Bot senden beziehungsweise empfangen.
6. Für jede konfigurierte Chat-ID erzeugt die Integration eine `notify`-Entity.
7. Die Implementierung von Core `2026.10.1` verwendet bei genau einer Telegram-Bot-Config-Entry und einem Aufruf ohne `chat_id`/`entity_id` automatisch die erste konfigurierte zulässige Chat-ID.
8. Gibt es mehrere Telegram-Bot-Config-Entries, schlägt ein Aufruf ohne explizites Ziel geschlossen fehl.

Quellen:

- <https://www.home-assistant.io/docs/configuration/secrets/>
- <https://www.home-assistant.io/integrations/telegram_bot/>
- <https://github.com/home-assistant/core/blob/2026.10.1/homeassistant/components/telegram_bot/services.py>

## Empfehlung

### 1. Bot-Token ausschließlich in der Telegram-Integration

Der API-Token bleibt in der über **Einstellungen → Geräte & Dienste → Telegram bot** eingerichteten Config Entry. Er gehört weder in Projekt-YAML noch in eine projektlokale `secrets.yaml`.

### 2. Eingehende Befehle über die Telegram-Allowlist absichern

Die Telegram-Integration übernimmt die vorgelagerte Chat-Allowlist. `Telegram Eingang` benötigt daher keinen Vergleich mit einer im Script gespeicherten Benutzer-ID. Damit entfällt `__TELEGRAM_USER_ID__` vollständig.

Mehrere private Einzelchats werden unterstützt, sofern jeder einzeln in der Integration als **Zulässige Chat ID** konfiguriert ist. Zusätzlich begrenzt `Telegram Eingang` die Verarbeitung ohne feste IDs auf Telegram-Direktchats:

```jinja2
trigger.event.data.chat_id | string
== trigger.event.data.user_id | string
```

In privaten Telegram-Chats stimmen Chat- und Benutzer-ID überein; in Gruppen nicht. Damit können allowgelistete Gruppen als Ausgabeziele bestehen bleiben, aber keine Klima-/Heizungsbefehle an diese Automation senden.

Für reine Ausgabegruppen wird zusätzlich empfohlen, den Telegram Bot Privacy Mode aktiviert zu lassen und dem Bot keine unnötigen Administratorrechte zu geben. Dadurch erhält Home Assistant bereits möglichst wenige Gruppenupdates. Die Template-Bedingung bleibt trotzdem als zweite Schutzschicht erforderlich, weil Telegram-Bots abhängig von Privacy-Modus und Nachrichtentyp weiterhin einzelne Gruppenereignisse, etwa ausdrücklich an den Bot gerichtete Befehle, erhalten können.

Die Zulassung einer Gruppe autorisiert nicht automatisch die privaten Einzelchats ihrer Mitglieder. Ein Gruppenmitglied kann dem Bot nur dann direkt einen verarbeiteten Befehl senden, wenn auch sein privater Chat separat allowgelistet wurde.

Die Automation antwortet bereits dynamisch an `trigger.event.data.chat_id`; dafür ist kein gespeicherter Standardwert erforderlich.

### 3. Primärantwort immer an den ursprünglichen Chat

`Telegram Eingang` übernimmt `trigger.event.data.chat_id` und übergibt diesen Wert als Laufzeitziel an `Telegram Ausgang`. Damit gilt bei mehreren zulässigen Chats:

- Elternteil 1 sendet einen Befehl und erhält dessen Antwort,
- Kind 1 sendet einen Befehl und erhält dessen Antwort,
Gruppen werden als Befehlseingang abgelehnt und erzeugen daher keine Primärantwort. Das implizite Standardziel darf nicht für Request/Response-Antworten verwendet werden. Die Chat-ID wird nur zur Laufzeit verarbeitet und nicht im Repository gespeichert.

### 4. Konfigurierbare Ziele für asynchrone Systemmeldungen

Das implizite erste Allowlist-Ziel wird nicht verwendet, weil der Betreiber asynchrone Scheduler- und Fehlermeldungen gezielt beispielsweise an eine Familiengruppe senden möchte. Vorgesehen ist stattdessen eine standardmäßig deaktivierte Konfiguration:

```yaml
asynchronous_notifications_enabled: false
asynchronous_targets_csv: "__TELEGRAM_ASYNC_CHAT_IDS__"
```

Bei Aktivierung wird die kommaseparierte Liste normalisiert und als Liste von `chat_id`-Werten an `telegram_bot.send_message` übergeben. Leere Einträge werden entfernt, Dubletten werden vermieden und sämtliche Ziele müssen in der Telegram-Integration allowgelistet sein.

Ist kein Ursprungstarget vorhanden und `asynchronous_notifications.enabled` ist `false`, wird keine willkürliche Nachricht an den ersten Chat gesendet. Der Adapter liefert stattdessen ein strukturiertes Ergebnis `skipped: true` für die deaktivierte asynchrone Ausgabe. Damit ist das veröffentlichte Projekt mit Platzhalter sicher inaktiv, bis der Betreiber lokal ein Ziel konfiguriert.

Für Antworten mit explizitem, aus einem aktuellen Telegram-Event stammendem `target` bleibt `telegram_bot.send_message` mit `chat_id` verbindlich. Die Ziel-ID existiert dann nur zur Laufzeit und nicht im Repository.

### 5. Optionale Elternaufsicht mit Zielliste

Die zweite Zustellung dient fachlich der Elternaufsicht: Sendet beispielsweise Kind 1 einen Heizbefehl, erhält Kind 1 die Primärantwort und ein oder mehrere konfigurierte Elternziele erhalten zusätzlich eine nachvollziehbare Information. Diese Funktion ist optional; die Unterstützung mehrerer Aufsichtsziele ist nochmals optional, lässt sich aber mit geringem Zusatzaufwand direkt über eine kommaseparierte Liste abbilden:

```yaml
supervision_enabled: false
supervision_targets_csv: "__TELEGRAM_SUPERVISION_CHAT_IDS__"
```

Regeln:

- Standard ist `enabled: false`.
- Solange die Option deaktiviert ist, wird der Platzhalter niemals ausgewertet oder als Ziel verwendet.
- Bei Aktivierung muss der Betreiber den Platzhalter lokal durch eine kommaseparierte Liste zulässiger Chat-IDs ersetzen.
- Einzelchat, Gruppe und mehrere Ziele sind technisch gleichwertig.
- Die Aufsichtsmeldung enthält mindestens Absenderbezeichnung, normalisierten Befehl und abschließendes Ergebnis.
- Sie wird erst nach erfolgreicher Primärzustellung versendet.
- Das Ursprungstarget wird aus der Aufsichtsliste entfernt, damit keine Dublette entsteht.
- Doppelte Aufsichtsziele und leere Listenelemente werden entfernt.
- Ein Fehler der optionalen Aufsichtsmeldung darf eine bereits erfolgreiche Primärantwort nicht nachträglich als fehlgeschlagen darstellen.
- Alle Aufsichtsziele müssen ebenfalls in der Telegram-Integration allowgelistet sein.

Der veröffentlichte Platzhalter enthält keine reale vertrauliche Information und ist deshalb akzeptabel. Bei der späteren zentralen Konfiguration sollen asynchrone Ziele und Aufsicht gemeinsam in die Messenger-Konfiguration verschoben werden.

## Optionenvergleich

| Option | UI-Bearbeitung | Vertraulichkeit | Aufwand | Bewertung |
|---|---:|---:|---:|---|
| `!secret` direkt in UI-Scripts | nein | eingeschränkt; Admin/Trace kann Wert zeigen | niedrig | nicht geeignet |
| separate dateibasierte Secret-Automationen | nur lesbar | gut bei sauberem Dateizugriff | mittel | technisch möglich, widerspricht dem führenden UI-Workflow |
| `input_text` / `input_number` | ja | schlecht; Zustand und Historie können Wert zeigen | niedrig | nicht als Secret-Speicher verwenden |
| zusätzliche Custom Integration als Secret-Adapter | UI-Scripts bleiben möglich | gut bei sorgfältigem Design | hoch | für den aktuellen Telegram-Fall unnötig |
| Telegram-Config-Entry + Chat-Allowlist + deaktivierte Zielplatzhalter | ja | sehr gut; veröffentlichte YAML enthält keine reale ID | niedrig | **empfohlen** |

## Umsetzungsstand

Der minimale Veröffentlichungsschritt ist in den Repository-Kopien umgesetzt:

- feste Telegram-Benutzer-ID aus `Telegram Eingang` entfernt,
- Chat-Allowlist der Telegram-Bot-Integration als Zugangsgrenze übernommen,
- direkte Antworten verwenden weiterhin ausschließlich das Laufzeitziel,
- feste Standard-Chat-ID aus `Telegram Ausgang` entfernt,
- asynchrone Ausgabe mit kommaseparierter Zielliste ergänzt und standardmäßig deaktiviert,
- Telegram-Schlüssel aus `secrets.example.yaml` entfernt.

Noch nicht produktiv umgesetzt sind die Elternaufsicht, der messengerneutrale `reply_context` und die optionale Konfiguration der Heizungs-Zwischenbestätigung. Diese bleiben der nächste Konfigurationsschritt.

### `Automationen/Telegram Eingang.yaml`

Umgesetzt:

- feste Benutzer-ID-Bedingung entfernt,
- Chat-Allowlist der Telegram-Integration als Zugangsgrenze verwendet,
- Ereignis-`chat_id` bleibt Ziel der Primärantwort an `Telegram Ausgang`.

Dokumentiert ist außerdem, dass Gruppen nur als Ausgabeziel dienen und private Chats jeweils separat zugelassen werden müssen.

### `Skripte/Telegram Ausgang.yaml`

Umgesetzt:

- gesetztes `target` wird als verbindliche Laufzeit-`chat_id` der Primärantwort verwendet,
- ohne `target` wird ausschließlich die explizit aktivierte Liste `asynchronous_targets_csv` verwendet,
- deaktivierte asynchrone Ausgabe liefert `skipped: true`,
- aktivierte Ausgabe ohne gültiges Ziel schlägt mit verständlicher Konfigurationsmeldung fehl,
- Responses enthalten keine Chat-ID.

Aufsicht, Absenderbezeichnung und normalisierter Befehl folgen erst im nächsten Konfigurationsschritt.

### Bestehende Heizungs-Zwischenbestätigung

`Skripte/Heizung Steuerung.yaml` ruft bei unmittelbaren Befehlen derzeit `script.system_nachricht` ohne Ziel auf, bevor die Geräteaktion erfolgt. Diese Zwischenbestätigung kennt den Ursprungschat nicht und würde bei mehreren zulässigen Chats fälschlich an den Standardchat gehen.

Die Zwischenbestätigung bleibt fachlich erforderlich, weil die aktuelle Heizkörperanbindung bis zur sichtbaren Reaktion sehr lange benötigen kann. Sie wird deshalb kontexttreu umgebaut:

- `Telegram Eingang` erstellt einen messengerneutralen `reply_context` mit Messenger, Ursprungstarget, Absenderbezeichnung und normalisiertem Befehl.
- `System Message-Router`, `Klima Router` und gegebenenfalls `Klima Steuerung` reichen diesen Kontext additiv bis zur Heizungssteuerung weiter.
- `Heizung Steuerung` sendet vor der langsamen Geräteaktion eine bewusst unverbindliche Meldung an das Ursprungstarget, beispielsweise: `⏳ Heizungsbefehl angenommen. Die Übertragung an den Heizkörper läuft.`
- Die endgültige Antwort bleibt davon getrennt und bewertet den eigentlichen Scriptablauf.
- Scheduler-Aufrufe mit `acknowledge: false` erzeugen weiterhin keine Zwischenmeldung.
- Fehlt bei einem internen Aufruf ein Ursprungstarget, wird keine interaktive Zwischenbestätigung an irgendeinen Standardchat umgeleitet.

### `secrets.example.yaml`

- Telegram-Schlüssel entfernen, weil das Projekt sie nach der Migration nicht mehr benötigt.
- Datei entweder für künftig echte dateibasierte Secrets beibehalten oder löschen, solange kein solcher Anwendungsfall existiert.

### `README.md`

- Telegram-Bot-Einrichtung über Config Entry beschreiben.
- private Chats und bei Bedarf reine Ausgabegruppen als zulässige Chat-IDs hinzufügen.
- Direktchat-Prüfung, Gruppenblockade am Befehlseingang und getrennte Zulassung privater Einzelchats klar erläutern.
- lokale Aktivierung und kommaseparierte Ziele für asynchrone Meldungen und Elternaufsicht dokumentieren.
- erklären, dass deaktivierte veröffentlichte Platzhalter keine Nachricht auslösen.

## Sicherheits- und Betriebsgrenzen

- Eine Telegram-Chat-ID ist kein Bot-Token, wird in diesem Projekt aber weiterhin als vertraulicher installationsbezogener Identifikator behandelt.
- Die Integration-Allowlist schützt Chats; die zusätzliche Direktchat-Prüfung blockiert Gruppenereignisse für diese Projekt-Automation.
- Allowgelistete Gruppen dürfen als asynchrone oder Aufsichtsziele verwendet werden, nicht als Befehlseingang.
- Primärantworten verwenden immer die Laufzeit-Chat-ID des auslösenden privaten Einzelchats.
- Meldungen ohne bekannten Ursprung verwenden ausschließlich die explizit aktivierte asynchrone Zielliste; es gibt kein implizites erstes Chat-Ziel.
- Asynchrone Ausgabe und optionale Elternaufsicht sind standardmäßig deaktiviert und enthalten im veröffentlichten Code nur nichtfunktionsfähige Platzhalter.
- HA-Administratoren besitzen grundsätzlich weitreichenden Zugriff auf Integrationseinstellungen und Automations-Traces; das Projekt versucht nicht, Secrets vor voll privilegierten HA-Administratoren zu verbergen.
- Runtime-Eventdaten können in Automations-Traces auftauchen. Das ist ein lokales HA-Betriebsrisiko, aber kein Repository-Leak.

## Veröffentlichungskontrolle

Vor der ersten Veröffentlichung:

1. Migration nach separater Freigabe umsetzen und in HA testen.
2. Repositoryweit nach Token-, Chat-ID-, Benutzer-ID-, Passwort- und API-Key-Mustern suchen.
3. `secrets.yaml` und ähnliche lokale Dateien über `.gitignore` ausschließen.
4. Prüfen, dass nie eine reale Secret-Datei zum Git-Index hinzugefügt wurde.
5. Falls bereits eine Git-Historie existiert, zusätzlich deren gesamte Historie scannen; bloßes Löschen aus dem aktuellen Stand reicht nicht.
6. README-Installationsanleitung und eine Lizenz ergänzen beziehungsweise bewusst festlegen.
7. Erst danach initial committen und veröffentlichen.

## Bestätigte Entscheidungen

1. Die Automation akzeptiert nur allowgelistete private Einzelchats; Gruppen sind als Befehlseingang gesperrt und bleiben als Ausgabeziel möglich.
2. Private Direktnachrichten eines Gruppenmitglieds bleiben gesperrt, solange dessen privater Chat nicht zusätzlich allowgelistet ist.
3. Es existiert derzeit genau eine Telegram-Bot-Integration mit mehreren Chats.
4. Asynchrone Ziele werden explizit und optional konfiguriert; der erste Allowlist-Eintrag ist kein fachlicher Standard.
5. Die Heizungs-Zwischenbestätigung bleibt erhalten und wird zieltreu über einen messengerneutralen Antwortkontext versendet.
6. Die Elternaufsicht ist standardmäßig deaktiviert und unterstützt eine kommaseparierte Zielliste.

Eine externe `secrets.yaml` ist weiterhin möglich, vereinfacht diesen Fall aber nicht: `!secret` kann in den UI-gepflegten Objekten nicht verwendet werden. Die veröffentlichte Projektfassung enthält ausschließlich deaktivierte Platzhalter; reale Zielwerte werden lokal eingesetzt.