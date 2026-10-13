# Fachdesign – Konfigurierbare Kollisionsprüfung und Überschreiben

Stand: 2026-10-12

Status: Fachdesign und Analyse. Keine der beschriebenen Änderungen ist in den produktiven Scripts oder Automationen umgesetzt.

## 1. Ziel und Systemgrenzen

Das Fachmodell unterscheidet herstellerunabhängig zwei Gerätesysteme:

- **Raumklima:** Befehle `klima ...`; aktuell Split-Klimaanlagen mit den Modi `heat`, `cool`, `fan_only`, `dry`, `auto` und `off`.
- **Heizkörper:** Befehle `heizung ...`; aktuell separat steuerbare Heizkörperthermostate.

Hersteller und Modelle sind Eigenschaften einer konkreten Installation, aber keine Bestandteile der Kollisionslogik. Verständliche Begriffe wie `raumklima`, `heizkoerper`, `innengeraet` und `aussengeraet` sollen in Konfiguration und Code verwendet werden.

Eine gemeinsame HA-Domain oder Raumzuordnung bedeutet nicht, dass zwei Entities dasselbe Gerät oder dieselbe technische Ressource darstellen. Sowohl Raumklima- als auch Heizkörpergeräte sind derzeit HA-`climate`-Entities. Der Systemtyp darf daher niemals allein aus der Entity-Domain abgeleitet werden.

Die zentrale Prüfung liefert genau eine der Klassifikationen:

- `allowed`: zulässig, keine Meldung erforderlich.
- `warning`: zulässig; Warnung wird mit `⚠️` am Anfang der Benutzerantwort ausgegeben.
- `conflict`: blockiert, sofern keine freigegebene Überschreibregel greift.

Kompatibilitätsbewertungen funktionieren in beiden Auftragsrichtungen. Gerichtete Prioritäten sind davon getrennt und werden erst nach einer festgestellten Kollision geprüft.

Die Prüfung soll gelten für:

- geplante Aufträge gegen bestehende Kalenderaufträge,
- mehrere neue Aufträge desselben Batches,
- unmittelbare Befehle gegen relevante geplante Reservierungen,
- unmittelbare Befehle gegen den aus HA gelesenen aktiven Gerätezustand.

Ein kollidierender Livezustand blockiert einen unmittelbaren Befehl ohne Force. Die Antwort nennt das betroffene Gerät, dessen Modus und den Grund der Kollision und weist auf `-überschreiben` hin, sofern manuelles Überschreiben aktiviert ist. Mit Force gilt der neue Befehl als aktueller Wille: kollidierende Raumklimageräte dürfen aktiv auf `off` gesetzt und eigene überlappende Kalenderjobs entsprechend ersetzt werden.

## 2. Analyse des bestehenden Codes

### 2.1 Aktueller Jobvertrag

`Klima - Steuerung` und `Heizung - Steuerung` übergeben an `Klima - Kalender` jeweils eine Liste mit:

```yaml
room: AZ
resource: climate # oder heating
start_at: "2026-10-12 18:00:00"
end_at: "2026-10-12 22:00:00"
summary: Klima Kühlen 24 °C
description: klima kühlen az 24 superhigh
```

Der Kalender speichert `summary`, `description`, `location`, Start und Ende. System und Modus bestehender Ereignisse werden aus `description` rekonstruiert.

### 2.2 Reservierungen und Folgeaktionen

Bei einer sofort beginnenden, zeitlich begrenzten Planung wird die Primäraktion direkt ausgeführt und zusätzlich als nicht ausführbare Reservierung gespeichert:

```text
reservation: klima ...
```

Eine Folgeaktion wird als separates einminütiges, ausführbares Kalenderereignis angelegt. Primärjob beziehungsweise Reservierung und Folgeaktion besitzen keine gemeinsame stabile logische Auftrags-ID.

Folgen:

- Zusammengehörige Ereignisse können nicht sicher gruppiert werden.
- Ein gelöschter Primärjob kann eine verwaiste Folgeaktion hinterlassen.
- Anzeige-IDs wie `AZ_1` sind temporäre Listennummern, keine Calendar-Event-UIDs.

### 2.3 Aktuelle Kollisionsprüfung

`Skripte/Klima Kalender.yaml` enthält zwei duplizierte Prüfblöcke:

1. neue Jobs gegen bestehende Kalendereinträge,
2. neue Jobs desselben Batches gegeneinander.

Die Implementierung betrachtet nur:

- gleichen Raum,
- echte zeitliche Überlappung,
- `resource: climate|heating`,
- eine fest codierte Liste blockierender Klimamodi.

Nicht vorhanden sind:

- Außengerätegruppen,
- raumübergreifende Raumklima-Konflikte,
- Warnungen,
- drei Klassifikationsstufen,
- Prioritäten und Überschreiben,
- stabile logische Auftragsgruppen,
- Sammlung mehrerer Konflikte.

Der erste Konflikt stoppt den Batch. Der zwischenzeitliche Vorschlag, `fan_only` pauschal in alte `blocked_modes`-Listen aufzunehmen, ist verworfen: Heizkörper-Heizen plus Raumklima-`fan_only` ist ausdrücklich erlaubt.

Die vorhandenen Felder `config.climate.calendar` und `config.heating.calendar` werden von Kalender und Scheduler nicht genutzt; dort ist `calendar.klima` fest codiert.

### 2.4 Nicht atomare Verarbeitung

Der Ablauf lautet: Bestand lesen, prüfen, Ereignisse einzeln anlegen. Zwischen Prüfung und Schreiben sind parallele Änderungen möglich. Schlägt eine spätere Erstellung fehl, existiert kein Rollback für bereits erstellte Ereignisse.

Auch Sofortaktion und Planung sind nicht atomar: Reservierung und Folgeaktion werden vor der Geräteaktion angelegt. Bei einem Gerätefehler können Kalenderereignisse bestehen bleiben.

## 3. Zentral konfigurierbares Datenmodell

Die Konfiguration soll additiv unter `mapping.config.conflicts` in `System Raum Mapping` liegen. Die Verträge `mapping.rooms`, `mapping.aliases` und `mapping.config` bleiben erhalten. Bestehende Kompatibilitätsfelder wie `room.systems.climate.entity` und `room.systems.heating.entity` können zunächst weitergeführt werden.

Der folgende Entwurf ist logisch und noch nicht implementiert:

```yaml
config:
  conflicts:
    schema_version: 1

    overwrite:
      automatic: false
      manual: false

    device_systems:
      raumklima:
        aussengeraete:
          aussengeraet_az_wz:
            innengeraete:
              - innengeraet_az
              - innengeraet_wz
          aussengeraet_sz_ku:
            innengeraete:
              - innengeraet_sz
              - innengeraet_ku

        same_innengeraet:
          default: conflict

        same_aussengeraet:
          mode_matrix:
            heat:
              heat: allowed
              cool: conflict
              dry: conflict
              fan_only: conflict
              auto: conflict
              "off": allowed
            cool:
              heat: conflict
              cool: allowed
              dry: allowed
              fan_only: allowed
              auto: conflict
              "off": allowed
            dry:
              heat: conflict
              cool: allowed
              dry: allowed
              fan_only: allowed
              auto: conflict
              "off": allowed
            fan_only:
              heat: conflict
              cool: allowed
              dry: allowed
              fan_only: allowed
              auto: conflict
              "off": allowed
            auto:
              heat: conflict
              cool: conflict
              dry: conflict
              fan_only: conflict
              auto: conflict
              "off": allowed
            "off":
              heat: allowed
              cool: allowed
              dry: allowed
              fan_only: allowed
              auto: allowed
              "off": allowed

        different_aussengeraet:
          default: allowed

      heizkoerper:
        same_room:
          default: allowed
        different_room:
          default: allowed

    cross_system:
      heizkoerper_raumklima:
        same_room:
          raumklima_modes:
            heat: allowed
            cool: conflict
            dry: conflict
            fan_only: allowed
            auto: conflict
            "off": allowed
        different_room:
          raumklima_modes:
            heat: allowed
            cool: warning
            dry: warning
            fan_only: allowed
            auto: warning
            "off": allowed

    priorities:
      automatic:
        - id: raumklima_heat_replaces_fan_only
          incoming:
            system: raumklima
            mode: heat
          existing:
            system: raumklima
            mode: fan_only
          scopes:
            - same_innengeraet
            - same_aussengeraet
          action: replace_overlap
```

Die Räume erhalten dazu eindeutige, lesbare Komponentenreferenzen. Beispielhaft:

```yaml
rooms:
  az:
    systems:
      climate:
        type: raumklima
        entity: climate.climacontrol_arbeitszimmer_climate
        innengeraet: innengeraet_az
        aussengeraet: aussengeraet_az_wz
      heating:
        type: heizkoerper
        entity: climate.arbeitszimmer_heizung_az
```

Die englischen Bestandsfelder `climate` und `heating` werden aus Kompatibilitätsgründen nicht sofort umbenannt. Neue Typ- und Gruppennamen sind herstellerneutral und beschreibend.

### 3.1 Bedeutung der Bereiche

- `same_innengeraet`: zwei überlappende eigenständige Raumklima-Aufträge für dasselbe Innengerät; immer `conflict`, unabhängig vom Modus.
- `same_aussengeraet`: verschiedene Innengeräte am selben Außengerät; Bewertung über die Modusmatrix.
- `different_aussengeraet`: Innengeräte verschiedener Außengeräte; standardmäßig technisch unabhängig.
- `heizkoerper`: Heizkörperaufträge kollidieren untereinander nicht.
- `cross_system.same_room`: Heizkörper und Raumklima im selben Raum.
- `cross_system.different_room`: raumübergreifende Plausibilitätswarnung; die Außengerätezuordnung ist dabei unerheblich.

Die aktuelle Matrix für Raumklimageräte derselben Außengerätegruppe basiert auf der Referenzinstallation mit MXZ-2F53VF und MSZ-AY25VGK. Sie ist ein konfigurierbares Installationsprofil, keine globale Herstelleraussage. `auto + auto` ist bewusst konservativ als `conflict` festgelegt.

Die Matrix ist symmetrisch. Die Implementierung soll widersprüchliche Werte beim Laden erkennen. Nur Einträge unter `priorities` sind gerichtet. Ein unbekannter oder nicht konfigurierter aktiver Raumklimamodus am selben Außengerät wird konservativ als `conflict` behandelt und mit einer klaren Diagnose gemeldet.

### 3.2 Normalisierte interne Darstellung

Jeder geplante oder aus dem Livezustand abgeleitete Job wird vor der Bewertung normalisiert:

```yaml
room: az
system: raumklima
resource: climate # altes Kompatibilitätsfeld
mode: heat
entity: climate.climacontrol_arbeitszimmer_climate
innengeraet: innengeraet_az
aussengeraet: aussengeraet_az_wz
start_at: ...
end_at: ...
source: calendar # batch oder live_state
project_owned: true
logical_job_id: ...
role: primary # reservation oder followup
calendar_uid: ...
```

Ein live aktiver Zustand hat keine Calendar-UID und ist nicht automatisch projektbesessen. Für ihn gelten deshalb andere Mutationsmöglichkeiten als für einen eigenen Kalenderjob.

### 3.3 Bewertungsreihenfolge

1. Keine echte Zeitüberlappung beziehungsweise kein aktuell aktiver Livezustand → `allowed`.
2. Systemtyp, Modus und Gerätebezug normalisieren.
3. Projektfremde oder unvollständig erkennbare Kalendereinträge niemals automatisch mutieren.
4. Gleiches Raumklima-Innengerät → `same_innengeraet`.
5. Verschiedene Raumklima-Innengeräte am selben Außengerät → `same_aussengeraet.mode_matrix`.
6. Raumklima-Innengeräte verschiedener Außengeräte → `different_aussengeraet`.
7. Zwei Heizkörperaufträge → `allowed`.
8. Heizkörper und Raumklima → Cross-System-Matrix nach gleichem oder unterschiedlichem Raum.
9. Bei `conflict` erst danach automatische beziehungsweise manuelle Überschreibung prüfen.

Warnungen werden gesammelt und blockieren weder Auftrag noch Batch. Rückgaben werden kompatibel erweitert:

```yaml
ok: true
collision: false
classification: warning
warnings:
  - code: cross_system_different_room
    message: "⚠️ Gleichzeitig wird in einem anderen Raum geheizt und gekühlt."
created: 2
message: ...
```

Bestehende Aufrufer können weiter `ok` verwenden. Fachsteuerungen müssen Warnungen in die Benutzerantwort übernehmen.

## 4. Überschreibregeln

### 4.1 Globale Schalter

```yaml
conflicts:
  overwrite:
    automatic: false
    manual: false
```

- `automatic`: ausschließlich explizit konfigurierte, gerichtete Prioritäten dürfen verdrängen.
- `manual`: ein expliziter Befehlsparameter darf jede technisch mutierbare Kollision zur Verdrängung freigeben.

Die Schalter sind unabhängig. `warning` benötigt nie Überschreiben; `allowed` löst keine Prioritätsprüfung aus.

### 4.2 Verbindliche automatische Priorität

Ein neuer Raumklima-`heat`-Job darf einen bestehenden Raumklima-`fan_only`-Job ersetzen, wenn:

- beide dasselbe Innengerät betreffen oder
- beide zu verschiedenen Innengeräten desselben Außengeräts gehören,
- eine echte zeitliche Kollision vorliegt,
- `overwrite.automatic` aktiviert ist.

Die Regel ist gerichtet. Neues `fan_only` verdrängt vorhandenes `heat` nicht automatisch. Zwischen verschiedenen Außengeräten greift sie nicht. Heizkörper-Heizen plus Raumklima-`fan_only` bleibt `allowed` und löst die Priorität nicht aus.

### 4.3 Manuelles Überschreiben und Syntax

Empfehlung:

- kanonisch und in Hilfetexten: `-überschreiben`
- technischer Alias: `-force`
- optional ebenfalls akzeptiert: `--überschreiben`, `--force`
- ausschließlich als eigenständiges Token, nicht als Teil eines Worts
- wirkt auf den gesamten eingehenden Auftrag, nicht auf die Folgeaktion allein

Beispiel:

```text
klima heizen wz 22 morgen von 19 bis 21 -überschreiben
```

`System Zeitplanung` sollte die Kontrolltokens vor der Zerlegung in Aktion, Zeitplanung und Folgeaktion zentral entfernen und additiv liefern:

```yaml
overwrite_requested: true
```

Damit bleiben Fachparser frei von Syntaxduplikaten. Bei `manual: false` wird eine Anforderung klar abgelehnt. Mehrfach angegebene gleichwertige Tokens können idempotent als ein `true` behandelt werden.

`-force` ist kurz und für technisch erfahrene Nutzer verständlich, aber weniger selbsterklärend und sprachlich uneinheitlich. Daher ist es als Alias geeignet, nicht als primäre Nutzersyntax.

## 5. Kalender-API und technische Optionen

### 5.1 Gegen Home Assistant Core 2026.10.1 verifiziert

Aktualisierte produktive Installation laut Nutzer:

- Home Assistant Container
- Core `2026.10.1`
- Frontend `20260930.3`

Der Quellcode des Tags `2026.10.1` bestätigt:

- Local Calendar unterstützt die Featureflags Create, Delete und Update.
- WebSocket-Kommandos sind vorhanden:
  - `calendar/event/create`
  - `calendar/event/delete`
  - `calendar/event/update`
  - `calendar/event/subscribe`
- Update und Delete arbeiten mit Event-UIDs.
- Als reguläre, aus UI-Scripts aufrufbare Calendar-Actions werden weiterhin nur Create und Get registriert.
- `calendar.get_events` filtert die Antwort weiterhin auf Listenfelder ohne UID.
- REST- und WebSocket-Eventmodelle enthalten die UID.

In `2026.10.1` wurden die regulären Services in `homeassistant/components/calendar/services.py` ausgelagert. Dort werden ausdrücklich nur `create_event` und `get_events` registriert. Das Update hat die Script-Lücke für echtes Update/Delete daher nicht geschlossen.

Zu verifizieren bleibt das reale Laufzeitverhalten von `calendar.klima` mit einem Testkalender, insbesondere `supported_features`, UID-Ausgabe über REST/WebSocket, Berechtigungen und Fehlerantworten.

Quellen:

- <https://www.home-assistant.io/integrations/local_calendar/>
- <https://github.com/home-assistant/core/tree/2026.10.1/homeassistant/components/calendar>
- <https://github.com/home-assistant/core/tree/2026.10.1/homeassistant/components/local_calendar>

Direkte `.ics`-Bearbeitung ist ausgeschlossen.

### 5.2 Nachweisbarer Zugriffspfad und Stabilitätsgrenze

Eine Custom Integration erhält nicht automatisch Zugriff auf die Implementierung anderer Integrationen. Der Adapter darf deshalb weder `LocalCalendarEntity`, `LocalCalendarStore`, `EventStore` noch andere Objekte aus `homeassistant.components.local_calendar` importieren.

Der nachgewiesene generische Zugriffspfad ist:

```python
from homeassistant.components.calendar import (
    DATA_COMPONENT,
    CalendarEntity,
    CalendarEntityFeature,
)

component = hass.data[DATA_COMPONENT]
entity = component.get_entity("calendar.klima")

assert isinstance(entity, CalendarEntity)
events = await entity.async_get_events(hass, start, end)
await entity.async_update_event(uid, event)
await entity.async_delete_event(uid)
```

Dieser Weg ist aus folgenden Gründen deutlich stabiler als der Zugriff auf Local-Calendar-Interna:

- `CalendarEntity` definiert generisch `async_get_events`, `async_create_event`, `async_update_event` und `async_delete_event`.
- `CalendarEvent` enthält das Feld `uid`.
- `CalendarEntityFeature` deklariert Create, Update und Delete.
- Home Assistants eigene WebSocket-Handler suchen die Entity ebenfalls über `hass.data[DATA_COMPONENT].get_entity(entity_id)` und rufen anschließend genau `async_update_event` beziehungsweise `async_delete_event` auf.
- Local Calendar implementiert diese generischen Methoden und kapselt Speicherung, Sperre und `.ics`-Serialisierung selbst.
- Die relevanten Methoden, Featureflags und der Lookup-Pfad sind in Core `2026.7.2`, Core `2026.10.1` und dem am 2026-10-12 geprüften `dev`-Stand vorhanden.

Damit benötigt der Adapter keine Kenntnis der Local-Calendar-Datei oder ihrer privaten Speicherimplementierung.

**Stabilitätsgrenze:** `CalendarEntity` und die Featureflags sind ein klarer Core-Plattformvertrag. Der Zugriff auf das laufende Entity-Objekt über `hass.data[DATA_COMPONENT]` wird von Core selbst verwendet, ist aber keine ausdrücklich als dauerhaft stabil dokumentierte Custom-Integration-API. Ein HA-Update kann daher weiterhin eine Anpassung erfordern. Der Adapter ist vertretbar, aber nicht wartungsfrei.

Verbindliche Schutzmaßnahmen:

1. Manifest-Abhängigkeit von `calendar`, damit die Plattform vorher geladen ist.
2. Keine Imports aus `homeassistant.components.local_calendar`.
3. Beim Setup und vor jeder Mutation Typ, Methoden, Featureflags und Ziel-Entity prüfen.
4. Bei fehlender Fähigkeit geschlossen fehlschlagen; niemals auf direkte `.ics`-Bearbeitung ausweichen.
5. Nur `calendar.klima` beziehungsweise explizit konfigurierte Zielkalender zulassen.
6. Projektbesitz und Metadatenschema vor jeder Mutation prüfen.
7. Adapterversion und getestete minimale/maximale HA-Version dokumentieren.
8. Vor jedem HA-Update einen Kompatibilitätstest ausführen; nach dem Update Create/List/Update/Delete auf einem Testkalender prüfen.
9. Bei inkompatibler API keine Mutations-Actions registrieren und einen klaren Reparatur-/Systemhinweis erzeugen.

Die endgültige technische Freigabe erfolgt erst durch einen isolierten Laufzeitnachweis auf einem Testkalender:

- Termin über die normale Calendar-Action erstellen,
- über den Adapter einschließlich UID lesen,
- über die generische Entity-Methode ändern und Ergebnis erneut lesen,
- löschen und Abwesenheit verifizieren,
- Neustartpersistenz prüfen,
- fehlende Featureflags und falsche Projektmetadaten als negative Tests prüfen.

### 5.3 Managementbewertung der Umsetzungsoptionen

| Option | Nutzen | Aufwand/Risiko | Bewertung |
|---|---|---|---|
| Nur klassifizieren, nicht ersetzen | Schnell, vollständig in UI-Scripts, geringstes Betriebsrisiko | Konflikte können nur blockiert werden | Geeignete erste Ausbaustufe |
| Home Assistant aktualisieren | Wartung, Sicherheit und Fehlerkorrekturen | Update bereits auf `2026.10.1` erfolgt; weiterhin keine normalen Delete-/Update-Actions | Sinnvoll durchgeführt, Script-Lücke bleibt bestehen |
| Schmaler Kalender-Mutationsadapter als Custom Integration | Verwendet die generische `CalendarEntity`-Schnittstelle; restliche Scripts bleiben UI-gepflegt | Kleine Python-Komponente unter `custom_components`; interner Entity-Lookup kann nach HA-Updates Anpassung verlangen | Bevorzugte Option, sofern isolierter Laufzeittest und Update-Prüfung bestehen |
| Externer Dienst, AppDaemon oder Node-RED | Flexible WebSocket-Nutzung | Zusätzliche Laufzeit, Authentifizierung und Betriebsüberwachung | Möglich, aber für dieses Projekt unnötig breit |
| HA-Selbstaufruf per REST/WebSocket und Long-Lived Token | Ohne Custom Integration prinzipiell möglich | Secret-, Berechtigungs- und Wartungsrisiko | Nicht empfohlen |
| Direkte `.ics`-Bearbeitung | Kein API-Adapter nötig | Datenkorruption, Race Conditions, Umgehung von HA | Ausgeschlossen |

Das Update auf `2026.10.1` ist erfolgt. Die Quellcodeprüfung zeigt weiterhin keine nativen Script-Actions für UID-Lesen, Update oder Delete.

Mit „Custom Integration“ ist hier **kein neues Kalenderplugin und kein Ersatz für Local Calendar** gemeint. Gemeint ist eine kleine lokale HA-Komponente, die ausschließlich wenige kontrollierte Actions wie `ha_climate_calendar.list_events_with_uid`, `update_event` und `delete_event` bereitstellt und dafür die vorhandenen Calendar-Entity-Funktionen nutzt. Die Fachlogik, Auswahl der zu ändernden Events und Benutzerkommunikation bleiben in den bestehenden Scripts.

Reine Script-Anpassungen können mit den in `2026.10.1` angebotenen Actions bestehende Termine nicht physisch aktualisieren oder löschen. Als script-only Ausweichlösung wären nur logische Sperr-/Ersetzungsmarker möglich, die der Scheduler beachten müsste; alte Termine blieben im Kalender sichtbar und die Lösung wäre komplexer sowie fehleranfälliger. Sie wird deshalb nicht empfohlen.

## 6. Vorschlag für zeitliche Ersetzung

### 6.1 Varianten

| Variante | Bewertung |
|---|---|
| Bestehenden Auftrag vollständig löschen | Einfach, verliert jedoch nicht kollidierende Zeitanteile |
| Bestehenden Auftrag kürzen oder aufteilen | Fachlich bevorzugt; benötigt UID, Update/Create/Delete und stabile Metadaten |
| Auftrag nur zur Laufzeit unterbrechen | Nicht empfohlen; erfordert zusätzlichen Laufzeitzustand und lässt konkurrierende Jobs bestehen |

Zielbild ist Variante 2.

### 6.2 Intervallsubtraktion

Für einen überschreibbaren Zeitraum `old` und den neuen Zeitraum `new`:

- `new` deckt `old` vollständig ab → alten Primärjob löschen.
- Überlappung am Ende von `old` → `old.end = new.start`.
- Überlappung am Anfang von `old` → `old.start = new.end`.
- `new` liegt vollständig in `old` → linken Rest behalten und rechten Rest als verknüpften Job erzeugen.
- Rand-an-Rand → keine Kollision.

```text
Alt:    18–22 Uhr Raumklima Lüften AZ
Neu:    19–21 Uhr Raumklima Heizen WZ

Links:  18–19 Uhr Raumklima Lüften AZ
Neu:    19–21 Uhr Raumklima Heizen WZ
Rechts: 21–22 Uhr Raumklima Lüften AZ
```

### 6.3 Projektmetadaten und Menschenlesbarkeit

Neue Projekttermine benötigen mindestens:

```yaml
project: ha_climate
schema_version: 1
logical_job_id: <stabile ID>
role: primary | reservation | followup
parent_job_id: <ID oder leer>
```

Vorschlag:

- `summary` bleibt menschenlesbar, zum Beispiel `2026-10-12 22:00 · Klima Heizen · AZ · 26 °C · Auto`.
- Erste `description`-Zeile bleibt das ausführbare beziehungsweise mit `reservation:` markierte Kommando.
- Zweite Zeile enthält eindeutig markierte maschinenlesbare Metadaten, vorzugsweise kompaktes JSON.

Beispiel:

```text
klima heizen az 26 auto
ha_climate_meta: {"schema_version":1,"logical_job_id":"20261012T220000-az-001","role":"primary","parent_job_id":""}
```

Scheduler und Parser dürfen nur die erste Zeile als Befehl interpretieren. Bestehende Ereignisse ohne Metadaten bleiben les- und ausführbar, aber vor automatischer Mutation geschützt. Projektfremde Ereignisse werden niemals automatisch verändert.

### 6.4 Folgeaktionen – Entscheidungsvorschlag

Beispiel:

```text
Alt: Lüften 18–22, danach aus um 22
Neu: Heizen 20–23
```

Wird der alte Job auf 18–20 gekürzt, sind beide naheliegenden Verschiebungen falsch:

- Folgeaktion auf 20 verschieben: Sie würde den gerade beginnenden Heizjob ausschalten.
- Folgeaktion bei 22 belassen: Sie würde den neuen Heizjob mitten in seiner Laufzeit ausschalten.

Empfohlene Regel:

- Primärjob vollständig entfernt → zugehörige alte Folgeaktion entfernen.
- Alter Job ohne rechten Rest gekürzt → alte Folgeaktion entfernen; der neue Auftrag bestimmt den späteren Zustand.
- Alter Job geteilt und rechter Rest läuft bis zum ursprünglichen Ende → ursprüngliche Folgeaktion genau einmal am ursprünglichen Ende behalten und dem rechten Rest zuordnen.
- Alter Job nur am Anfang gekürzt und läuft danach bis zum ursprünglichen Ende → Folgeaktion am ursprünglichen Ende behalten.
- Fachliche Folgeaktion ungleich `aus` → in der Ersetzungsantwort ausdrücklich nennen, weil ihre Entfernung eine bewusste Verhaltensänderung ist.

Diese Regel verhindert, dass eine alte Folgeaktion den verdrängenden Auftrag ungewollt beendet.

### 6.5 Konsistenz und nicht kompensierbare Teilmutation

Die Kalender-API bietet keine nachgewiesene Batchtransaktion. Sichere Annäherung:

1. Bestand inklusive UIDs und Auftragsgruppen lesen.
2. Klassifikationen und vollständigen Mutationsplan ohne Schreibzugriff berechnen.
3. Vor dem Schreiben unveränderten Vorzustand prüfen.
4. Projektmutationen zentral serialisieren.
5. Vorherzustände und eine eindeutige Operations-ID speichern.
6. Mutationen einzeln ausführen und jeweils prüfen.
7. Bei Fehlern bestmögliche Kompensation durchführen.
8. Kalender erneut lesen und Sollzustand verifizieren.

Problemfall:

1. Alter Job wird erfolgreich gekürzt.
2. Neuer Job oder rechter Rest kann nicht erstellt werden.
3. Auch die Wiederherstellung des alten Jobs schlägt fehl.

Dann ist der Kalender trotz Kompensationsversuch inkonsistent. Vorgeschriebenes Verhalten:

- keine Erfolgsmeldung,
- neue Geräteaktion nicht starten,
- harte Fehlermeldung mit Operations-ID,
- Administrator-/Systemnachricht senden,
- Kalenderzustand erneut einlesen,
- verbliebenen Istzustand und fehlgeschlagene Kompensation sichtbar protokollieren,
- keine weiteren abhängigen Mutationen dieser Auftragsgruppe ausführen.

Das ist keine echte Atomarität. Der Adapter muss Mutationen serialisieren und Vorherzustände halten. Projektfremde oder unvollständig identifizierte Ereignisse dürfen Warnungen oder Konflikte auslösen, aber nie automatisch gelöscht, geändert oder geteilt werden.

## 7. Minimalinvasiver Implementierungsplan

### Phase 0 – Laufzeit verifizieren

Keine produktive Codeänderung:

- `supported_features` von `calendar.klima` prüfen.
- UID-, Update- und Delete-Verhalten auf einem Testkalender prüfen.
- Update auf Core `2026.10.1` als erfolgt dokumentieren.
- Reales Verhalten der neuen Version auf einem Testkalender prüfen.
- Entscheidung treffen, ob der schmale Mutationsadapter als lokale HA-Komponente akzeptiert wird.

### Phase 1 – Zentrale Konfiguration

`Skripte/System Raum Mapping.yaml` additiv ergänzen:

- Komponentenrollen `raumklima` und `heizkoerper`,
- Innengeräte und Außengerätegruppen AZ/WZ sowie SZ/KU,
- vollständige Modus- und Cross-System-Matrix,
- Überschreibschalter,
- gerichtete automatische Priorität.

Bestehende Mapping-Felder bleiben erhalten.

### Phase 2 – Eine zentrale Klassifikation ohne Mutation

Primär `Skripte/Klima Kalender.yaml` minimalinvasiv so strukturieren, dass beide bisherigen Prüfpfade dieselbe Klassifikationslogik verwenden:

- normalisierte Jobs,
- `allowed`/`warning`/`conflict`,
- Gruppen- und Cross-System-Regeln,
- mehrere Warnungen und Konflikte sammeln,
- kompatible Rückgaben.

`Klima Steuerung` und `Heizung Steuerung` übernehmen Warnungen in ihre Antworten. Zunächst keine Ersetzung.

### Phase 3 – Unmittelbare Befehle

Vor Geräteaktionen aktive Modi und relevante Reservierungen normalisieren und mit derselben zentralen Logik bewerten. Keine getrennte zweite Fachmatrix in den Steuerungsskripten anlegen.

Bei einem Livekonflikt ohne Force wird blockiert und mit betroffenem Raum/Gerät, aktivem Modus und Konfliktgrund geantwortet. Bei aktivierter manueller Überschreibung und Force werden kollidierende Raumklimageräte kontrolliert auf `off` gesetzt, der Zustand wird soweit möglich bestätigt und erst danach wird der neue Befehl ausgeführt. Eigene kollidierende Kalenderjobs müssen zusätzlich ersetzt werden, damit sie den alten Zustand nicht erneut herstellen.

Scheitert der neue Gerätebefehl nach dem Ausschalten der Konfliktgeräte, bleiben die Konfliktgeräte ausgeschaltet. Zusätzlich wird auch das Zielgerät sicherheitshalber auf `off` gesetzt. Eine Wiederherstellung des alten Modus erfolgt nicht. Der Nutzer erhält eine deutliche Fehlermeldung, welche Abschaltungen erfolgreich waren und ob auch das Sicherheitsabschalten des Zielgeräts bestätigt werden konnte.

### Phase 4 – Manueller Parameter

`System Zeitplanung` extrahiert die Force-Tokens und liefert `overwrite_requested`. Fachsteuerungen reichen das Feld weiter. Bei deaktiviertem manuellem Überschreiben erfolgt eine klare Ablehnung.

### Phase 5 – Stabile Projektmetadaten

Joberzeugung, Kalenderparser und Scheduler um `logical_job_id`, Rollen und Metadatenzeile erweitern. Alttermine bleiben ausführbar, aber nicht automatisch mutierbar.

### Phase 6 – Kalender-Mutationsadapter

Nicht Bestandteil der aktuellen produktiven Veröffentlichung. Vor einer späteren Freigabe ist ein isolierter technischer Nachweis auf einem separaten Testkalender erforderlich. Minimalumfang:

- ausschließlich die generische `CalendarEntity`-Schnittstelle verwenden,
- Events mit UID lesen,
- Event per UID erstellen, aktualisieren und löschen,
- Featureflags und API-Kompatibilität vor jeder Mutation prüfen,
- Projektbesitz und Schema validieren,
- Mutationen serialisieren,
- Operations-ID, Vorherzustand und maschinenlesbare Fehler liefern,
- bei API-Abweichung geschlossen fehlschlagen und keine Mutation anbieten.

### Phase 7 – Ersetzung

Zuerst automatische `heat`-über-`fan_only`-Priorität implementieren. Danach manuelles Überschreiben und Intervallteilung aktivieren. Jede Stufe separat in HA testen und freigeben.

## 8. Regressionstestmatrix

Kein Fall gilt als bestanden, bevor er in der aktuellen HA-Installation ausgeführt und dokumentiert wurde.

### 8.1 Raumklima innerhalb eines Außengeräts

Für verschiedene Innengeräte derselben Außengerätegruppe ist die vollständige symmetrische Matrix aus Abschnitt 3 zu testen. Mindestfälle:

| ID | Bestehend | Neu | Räume | Erwartung |
|---|---|---|---|---|
| RK-01 | `heat` | `heat` | AZ/WZ | `allowed` |
| RK-02 | `heat` | `cool` | AZ/WZ | `conflict` |
| RK-03 | `cool` | `heat` | AZ/WZ | `conflict` |
| RK-04 | `heat` | `fan_only` | AZ/WZ | `conflict` |
| RK-05 | `fan_only` | `heat` | AZ/WZ | `conflict` |
| RK-06 | `cool` | `dry` | AZ/WZ | `allowed` |
| RK-07 | `dry` | `fan_only` | AZ/WZ | `allowed` |
| RK-08 | `auto` | `auto` | AZ/WZ | `conflict` |
| RK-09 | `off` | jeder aktive Modus | AZ/WZ | `allowed` |
| RK-10 | `heat` | `cool` | SZ/KU | `conflict` |
| RK-11 | `heat` | `cool` | AZ/SZ | `allowed` |
| RK-12 | beliebige Modi | beliebige Modi | gleiches Innengerät | `conflict` |
| RK-13 | unbekannter Modus | aktiver Modus | gleiches Außengerät | konservativ `conflict` mit Diagnose |

Alle Matrixpaare sind in beiden Auftragsrichtungen und mit vertauschten Räumen zu prüfen.

### 8.2 Heizkörper untereinander

| ID | Bestehend | Neu | Räume | Erwartung |
|---|---|---|---|---|
| HK-01 | Heizkörper heizen | Heizkörper heizen | gleicher Raum | `allowed` |
| HK-02 | Heizkörper heizen | Heizkörper heizen | verschiedene Räume | `allowed` |

### 8.3 Heizkörper und Raumklima im selben Raum

| ID | Bestehend | Neu | Erwartung |
|---|---|---|---|
| CROSS-01 | Heizkörper heizen | Raumklima `heat` | `allowed` |
| CROSS-02 | Raumklima `heat` | Heizkörper heizen | `allowed` |
| CROSS-03 | Heizkörper heizen | Raumklima `fan_only` | `allowed` |
| CROSS-04 | Raumklima `fan_only` | Heizkörper heizen | `allowed` |
| CROSS-05 | Heizkörper heizen | Raumklima `cool` | `conflict` |
| CROSS-06 | Raumklima `cool` | Heizkörper heizen | `conflict` |
| CROSS-07 | Heizkörper heizen | Raumklima `dry` | `conflict` |
| CROSS-08 | Raumklima `dry` | Heizkörper heizen | `conflict` |
| CROSS-09 | Heizkörper heizen | Raumklima `auto` | `conflict` |
| CROSS-10 | Raumklima `auto` | Heizkörper heizen | `conflict` |
| CROSS-11 | Heizkörper heizen | Raumklima `off` | `allowed` |

### 8.4 Heizkörper und Raumklima in unterschiedlichen Räumen

| ID | Bestehend | Neu | Räume | Erwartung |
|---|---|---|---|---|
| WARN-01 | Heizkörper heizen | Raumklima `cool` | WZ/SZ | `warning`, Auftrag zulässig |
| WARN-02 | Raumklima `cool` | Heizkörper heizen | SZ/WZ | `warning`, Auftrag zulässig |
| WARN-03 | Heizkörper heizen | Raumklima `dry` | AZ/WZ | `warning`, Auftrag zulässig |
| WARN-04 | Raumklima `dry` | Heizkörper heizen | WZ/AZ | `warning`, Auftrag zulässig |
| WARN-05 | Heizkörper heizen | Raumklima `auto` | WZ/SZ | `warning`, Auftrag zulässig |
| WARN-06 | Raumklima `auto` | Heizkörper heizen | SZ/WZ | `warning`, Auftrag zulässig |
| WARN-07 | Heizkörper heizen | Raumklima `fan_only` | WZ/AZ | `allowed` |
| WARN-08 | Heizkörper heizen | Raumklima `heat` | WZ/SZ | `allowed` |

Warnfälle sowohl innerhalb derselben als auch zwischen verschiedenen Außengerätegruppen testen. Jede Warnantwort beginnt mit `⚠️`.

### 8.5 Geplant, Batch und unmittelbar

| ID | Quelle bestehender Zustand | Neuer Auftrag | Erwartung |
|---|---|---|---|
| SRC-01 | Kalenderjob | Kalenderjob | Matrixentscheidung |
| SRC-02 | neuer Job desselben Batches | Kalenderjob | identische Matrixentscheidung |
| SRC-03 | Reservierung | unmittelbarer Befehl | identische Matrixentscheidung |
| SRC-04 | live aktiver Modus | unmittelbarer Befehl ohne Force | blockiert mit Problemhinweis |
| SRC-05 | live aktiver Konfliktmodus | unmittelbarer Befehl mit Force | Konfliktgerät auf `off`, danach neuer Befehl |
| SRC-06 | mehrere Warnquellen | beliebig | alle Warnungen gesammelt, Auftrag zulässig |
| SRC-07 | mehrere Konflikte | beliebig | alle Konflikte nachvollziehbar gemeldet, nichts teilweise angelegt |

### 8.6 Automatisches Überschreiben

| ID | Schalter | Bestehend | Neu | Scope | Erwartung |
|---|---:|---|---|---|---|
| AUTO-01 | aus | `fan_only` | `heat` | gleiches Innengerät | `conflict`, keine Mutation |
| AUTO-02 | an | `fan_only` | `heat` | gleiches Innengerät | Ersetzung |
| AUTO-03 | an | `fan_only` | `heat` | gleiches Außengerät | Ersetzung |
| AUTO-04 | an | `fan_only` | `heat` | verschiedene Außengeräte | keine automatische Ersetzung erforderlich/zulässig |
| AUTO-05 | an | `heat` | `fan_only` | gleiches Innengerät | `conflict`, keine Ersetzung |
| AUTO-06 | an | Heizkörper heizen | Raumklima `fan_only` | gleicher Raum | `allowed`, keine Ersetzung |

### 8.7 Manuelles Überschreiben

| ID | `manual` | Parameter | Erwartung |
|---|---:|---:|---|
| MAN-01 | aus | nicht gesetzt | normale Matrixentscheidung |
| MAN-02 | aus | `-überschreiben` | klare Ablehnung, keine Mutation |
| MAN-03 | an | nicht gesetzt | `conflict` bleibt blockiert |
| MAN-04 | an | `-überschreiben` | technisch mutierbare Konfliktjobs werden ersetzt |
| MAN-05 | an | `-force` | identisch zu `-überschreiben` |
| MAN-06 | an | gesetzt | `warning` bleibt zulässig, keine unnötige Mutation |
| MAN-07 | an | gesetzt | projektfremdes Event bleibt geschützt |
| MAN-08 | an | Token als Teil eines Worts | nicht als Force-Parameter erkennen |
| MAN-09 | an | Force gegen live aktives Konfliktgerät | Konfliktgerät auf `off`, neuer Auftrag wird ausgeführt |
| MAN-10 | an | Force gegen Livezustand und eigenen Kalenderjob | Livegerät ausschalten und Kalenderjob konsistent ersetzen |
| MAN-11 | an | Konfliktgerät aus, neuer Befehl schlägt fehl | Konfliktgerät bleibt aus; Zielgerät wird auf `off` gesetzt; deutliche Fehlermeldung |

### 8.8 Zeitliche Ersetzung

| ID | Alt | Neu | Erwartung |
|---|---|---|---|
| TIME-01 | 18–22 | 18–22 | Alt vollständig ersetzt |
| TIME-02 | 18–22 | 17–23 | Alt vollständig ersetzt |
| TIME-03 | 18–22 | 19–23 | linker Rest 18–19 |
| TIME-04 | 18–22 | 17–20 | rechter Rest 20–22 |
| TIME-05 | 18–22 | 19–21 | Reste 18–19 und 21–22 |
| TIME-06 | 18–22 | 22–23 | keine Überlappung |
| TIME-07 | mehrere ersetzbare Konflikte | ein neuer Job | konsistenter Gesamtplan |
| TIME-08 | Fehler beim Update | beliebig | Kompensation und harte Fehlermeldung |
| TIME-09 | Fehler beim rechten Rest | Teilung | kein stiller Datenverlust |

### 8.9 Auftragsgruppen und Fremdereignisse

| ID | Fall | Erwartung |
|---|---|---|
| META-01 | Primärjob vollständig entfernt | keine verwaiste Folgeaktion |
| META-02 | Kürzung ohne rechten Rest | alte Folgeaktion entfernt |
| META-03 | Teilung mit rechtem Rest | Folgeaktion genau einmal am ursprünglichen Ende |
| META-04 | fachliche Folgeaktion ungleich `aus` wird entfernt | Änderung in Antwort genannt |
| META-05 | altes Ereignis ohne Projektmetadaten | niemals automatisch mutieren |
| META-06 | manueller/projektfremder Termin | geschützt |
| META-07 | mehrere Befehle nahezu gleichzeitig | Serialisierung verhindert Überholen |
| META-08 | Neustart zwischen Teilmutationen | inkonsistenter Zustand erkannt und gemeldet |
| META-09 | Kompensation schlägt fehl | keine Geräteaktion; Operations-ID und Administratorhinweis |

## 9. Historische Evidenz

Historische Testergebnisse sind in `Dokumentation/TESTMATRIX.md` getrennt dokumentiert. Insbesondere gilt die frühere Aussage „Heizung + Klima Lüften erfolgreich getestet“ nicht als verlässlicher Testnachweis. Die heutige Sollregel erlaubt Heizkörper-Heizen plus Raumklima-`fan_only`; dies muss dennoch aktuell neu getestet werden.

Die frühere pauschale `fan_only`-Sperre ist verworfen. Sie darf weder in die alte Implementierung noch in die neue Matrix übernommen werden.

## 10. Offene Entscheidungen und benötigter Input

Zusätzlich entschieden:

- Ein unmittelbarer Livekonflikt wird ohne Force blockiert und verständlich erklärt.
- Force repräsentiert den aktuellen Benutzerwillen und darf kollidierende Raumklimageräte aktiv auf `off` setzen.
- Eigene kollidierende Kalenderjobs müssen dabei ebenfalls angepasst werden, damit sie den verdrängten Zustand nicht erneut aktivieren.
- Home Assistant wurde auf Core `2026.10.1` und Frontend `20260930.3` aktualisiert.

Weitere Entscheidungen:

- Ein unbekannter oder nicht konfigurierter Raumklimamodus am selben Außengerät wird konservativ als `conflict` behandelt und diagnostiziert.
- Scheitert nach einer Force-Abschaltung der neue Gerätebefehl, bleiben die zuvor ausgeschalteten Konfliktgeräte aus.
- Das Zielgerät wird in diesem Fehlerfall zusätzlich sicherheitshalber auf `off` gesetzt.
- Der Nutzer erhält eine deutliche Meldung über den fehlgeschlagenen Befehl und den bestätigten beziehungsweise nicht bestätigten Abschaltzustand aller betroffenen Geräte.

Bedingt entschieden:

- Eine spätere kleine Custom Integration im HA-Container ist nur vertretbar, wenn sie die generische `CalendarEntity`-Schnittstelle verwendet und zuvor isoliert auf einem separaten Testkalender nachgewiesen wird.
- Direkte Zugriffe auf private Local-Calendar-Implementierung, Speicher oder `.ics`-Dateien bleiben ausgeschlossen.
- Das verbleibende Update-Risiko des Entity-Lookups ist durch Featureprüfung, Fail-closed-Verhalten, Versionsdokumentation und verpflichtende Tests vor und nach HA-Updates zu begrenzen.

Ein solcher Adapter ist nicht Bestandteil dieser Veröffentlichung und wird von den produktiven YAML-Skripten nicht vorausgesetzt. Seine mögliche spätere Implementierung benötigt eine getrennte Freigabe.