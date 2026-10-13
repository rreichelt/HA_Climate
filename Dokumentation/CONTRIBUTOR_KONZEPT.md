# Konzept – Beiträge und kommerzielles Dual-Licensing

Stand: 2026-10-13

## Ziel

Externe Beiträge sollen von Beginn an möglich sein. Beitragende behalten ihr Urheberrecht und werden namentlich genannt. Gleichzeitig muss der Projektinhaber das Gesamtprojekt weiterhin

- nichtkommerziell unter `CC BY-NC-SA 4.0` veröffentlichen und
- aufgrund separater Vereinbarungen kommerziell lizenzieren

können.

Eine Vergütung für Beiträge ist grundsätzlich nicht geschuldet. Eine freiwillige oder projektbezogen vereinbarte Beteiligung bei kommerzieller Nutzung bleibt möglich, entsteht aber nur durch eine separate schriftliche Vereinbarung.

## Empfohlenes Modell

### Contributor License Agreement statt Copyright-Übertragung

Beitragende behalten ihr Copyright. Mit einem schlanken Contributor License Agreement, kurz CLA, räumen sie dem Projektinhaber eine dauerhafte, weltweite, nicht-exklusive und grundsätzlich unwiderrufliche Lizenz ein, den Beitrag:

- zu verwenden,
- zu vervielfältigen,
- zu verändern,
- mit anderen Beiträgen zu verbinden,
- öffentlich bereitzustellen,
- unter der öffentlichen Projektlizenz weiterzugeben,
- unter abweichenden Bedingungen einschließlich kommerzieller Lizenzen zu lizenzieren und unterzulizenzieren.

Die Lizenz muss ausdrücklich für kommerzielle Nutzung und Relizenzierung gelten. Eine reine Zustimmung zu `CC BY-NC-SA 4.0` oder ein Developer Certificate of Origin genügt dafür nicht zuverlässig.

### Rechte der Beitragenden

Beitragende:

- behalten ihr Urheberrecht,
- dürfen ihren eigenen Beitrag anderweitig verwenden und lizenzieren,
- werden entsprechend ihrer gewünschten öffentlichen Bezeichnung genannt,
- garantieren nur, dass sie den Beitrag einreichen dürfen,
- erhalten keine automatische Vergütungspflicht oder Beteiligung an späteren Einnahmen.

Eine mögliche Vergütung oder Umsatzbeteiligung kann im Einzelfall separat schriftlich vereinbart werden. Das CLA darf eine solche freiwillige Vereinbarung nicht ausschließen, begründet aber selbst keinen Anspruch darauf.

## Öffentliche Dateien

Nach erfolgreichem Live-Test sollte die öffentliche Repository-Fassung enthalten:

### `CONTRIBUTING.md`

Kurze, verständliche Anleitung:

1. Issue für größere Änderungen eröffnen.
2. Keine Secrets, Tokens, realen Chat-/User-IDs oder privaten Daten einreichen.
3. Tests und Dokumentation aktualisieren.
4. CLA einmalig akzeptieren.
5. Gewünschten Namen oder Alias für die Namensnennung angeben.

### `CLA.md`

Der vollständige Rechtstext. Er sollte mindestens regeln:

- Identität beziehungsweise GitHub-Konto des Beitragenden,
- Eigentum bleibt beim Beitragenden,
- nicht-exklusive Rechteübertragung,
- kommerzielle Nutzung, Unterlizenzierung und Relizenzierung,
- Recht zur Veröffentlichung unter `CC BY-NC-SA 4.0` oder späteren kompatiblen öffentlichen Bedingungen,
- Zusicherung der Beitragsberechtigung,
- Umgang mit Beiträgen aus einem Arbeitsverhältnis,
- keine Vergütungspflicht,
- separate Vergütungsvereinbarungen bleiben möglich,
- keine Pflicht zur Annahme oder Nutzung eines Beitrags,
- Haftungs- und Gewährleistungsausschluss,
- anwendbares Recht und Gerichtsstand nach bewusster Entscheidung.

Der finale Rechtstext sollte vor dem ersten kommerziellen Vertrag juristisch geprüft werden.

### `CONTRIBUTORS.md`

Öffentliche Namensnennung mit freiwilligen Angaben:

```text
- Anzeigename oder Alias – GitHub-Profil – Kurzbeschreibung der Beiträge
```

Klarnamen sind nicht erforderlich. Beitragende bestimmen, ob GitHub-Name, Alias oder Klarname genannt wird. Die Git-Historie bleibt zusätzlich als technische Urheberzuordnung erhalten.

### `.github/pull_request_template.md`

Kurze Checkliste:

- keine vertraulichen Werte,
- Tests ausgeführt,
- Dokumentation angepasst,
- Beitrag ist selbst erstellt oder rechtmäßig einreichbar,
- CLA akzeptiert,
- gewünschte Attribution angegeben.

## Technische CLA-Erfassung

### Bevorzugt: CLA-Bot beziehungsweise CLA-Assistant

Ein GitHub-CLA-Dienst kann die Zustimmung pro GitHub-Konto und CLA-Version protokollieren und Pull Requests bis zur Zustimmung markieren.

Vorteile:

- niedrige Einstiegshürde,
- nachvollziehbare Zustimmung,
- erneute Zustimmung bei wesentlicher CLA-Änderung,
- automatischer Pull-Request-Status.

Nachteile:

- externer Dienst erhält GitHub-Metadaten,
- zusätzliche Dienstabhängigkeit,
- Einrichtung und Datenschutzprüfung erforderlich.

### Anfangs möglich: manuelles Register

Bei wenigen Beiträgen kann die Zustimmung zunächst manuell über einen eindeutig formulierten Pull-Request-Kommentar und ein nichtöffentliches Register aus GitHub-Konto, Datum, Pull Request und CLA-Version dokumentiert werden. Eine bloße Checkbox ohne dauerhafte Protokollierung ist schwächer und nicht die bevorzugte Dauerlösung.

## Warum DCO allein nicht genügt

Ein `Signed-off-by` nach Developer Certificate of Origin bestätigt im Wesentlichen, dass der Beitrag rechtmäßig eingereicht werden darf und unter der Projektlizenz beigesteuert wird. Das schafft nicht automatisch die weitergehenden Rechte, den Beitrag unabhängig davon kommerziell zu relizenzieren. Ein DCO kann zusätzlich verwendet werden, ersetzt das CLA in diesem Geschäftsmodell aber nicht.

## Unternehmen und Minderjährige

- Beschäftigte müssen bestätigen, dass sie den Beitrag außerhalb entgegenstehender Arbeitgeberrechte einreichen dürfen, oder eine Freigabe des Arbeitgebers vorlegen.
- Für regelmäßige Unternehmensbeiträge kann später ein Corporate CLA ergänzt werden.
- Minderjährige benötigen abhängig von Rechtsordnung und Vertragsfähigkeit die Zustimmung einer erziehungsberechtigten Person. Der CLA-Prozess muss solche Beiträge im Zweifel manuell behandeln.

## Vergütung und kommerzielle Nutzung

Empfohlene klare Formulierung:

- Beiträge erfolgen grundsätzlich unentgeltlich.
- Es besteht kein automatischer Anspruch auf Erlöse, Lizenzgebühren oder Umsatzbeteiligung.
- Der Projektinhaber darf kommerzielle Lizenzen vergeben.
- Eine Vergütung, Prämie oder Beteiligung kann freiwillig angeboten oder für einen konkreten Beitrag separat schriftlich vereinbart werden.
- Eine spätere freiwillige Vergütung ändert nicht automatisch die Bedingungen anderer Beiträge.

So bleibt eine faire Beteiligung möglich, ohne bei jeder kommerziellen Lizenz eine kaum administrierbare automatische Abrechnungspflicht gegenüber sämtlichen Beitragenden auszulösen.

## Empfohlener Annahmeprozess

1. Pull Request wird eröffnet.
2. Automatischer Secret-Scan und Tests laufen.
3. CLA-Status wird geprüft.
4. Maintainer prüft Rechtehinweise, technische Qualität und Attribution.
5. Beitragender wird in `CONTRIBUTORS.md` ergänzt beziehungsweise bestätigt seine vorhandene Zeile.
6. Erst danach wird gemergt.
7. Für separat vergütete Arbeit wird vorab eine eigene schriftliche Vereinbarung geschlossen.

## Festgelegte Angaben

- Öffentlicher Rechteinhaber: **René Reichelt**
- Kontakt für kommerzielle Lizenzierung: `lizenz@rrsb.de`
- Rolle: natürliche Person, freiberuflich tätig; keine Gesellschaft oder andere juristische Person als Rechteinhaber benannt
- CLA-Erfassung: manuelles Register, da höchstens eine einstellige Zahl externer Beiträge erwartet wird
- Erste Veröffentlichungsphase: Issues sind willkommen; Pull Requests werden nur nach vorheriger Abstimmung beziehungsweise Einladung angenommen

## Empfohlenes Attributionsformat

`CONTRIBUTORS.md` verwendet:

```text
- Gewünschter Name oder Alias (@github-konto) – kurze Beschreibung des Beitrags
```

Regeln:

- Beitragende wählen Klarname, Alias oder GitHub-Anzeigename.
- Ohne gesonderten Wunsch wird das GitHub-Konto verwendet.
- E-Mail-Adressen werden nicht öffentlich eingetragen.
- Die Git-Historie bleibt zusätzliche technische Attribution.
- Die Hauptlizenz nennt René Reichelt als ursprünglichen Rechteinhaber; externe Urheber werden ergänzend in `CONTRIBUTORS.md` geführt.

## Empfohlene Rechtswahl

Für den endgültigen CLA-Entwurf bietet sich folgende vorsichtige Grundformulierung an:

```text
Es gilt das Recht der Bundesrepublik Deutschland unter Ausschluss des
UN-Kaufrechts. Gerichtsstand ist, soweit eine Gerichtsstandsvereinbarung
gesetzlich zulässig ist, der Wohn- beziehungsweise Geschäftssitz des
Projektinhabers.
```

Der Zusatz „soweit gesetzlich zulässig“ ist wichtig, weil ein Gerichtsstand nicht gegenüber jeder natürlichen Person wirksam frei vereinbart werden kann. Die freiberufliche Tätigkeit ändert nicht automatisch den urheberrechtlichen Rechteinhaber: Solange keine Gesellschaft als Rechteinhaber benannt wird, handelt René Reichelt als natürliche Person.

Vor dem ersten kommerziellen Vertrag oder einem konfliktträchtigen internationalen Beitrag sollte der finale CLA-Text juristisch geprüft werden.

## Manuelles CLA-Register

Das Register bleibt außerhalb des öffentlichen Repositorys und enthält pro Zustimmung mindestens:

```text
GitHub-Konto
gewünschte öffentliche Attribution
CLA-Version und Prüfsumme beziehungsweise Commit-ID
Datum und Uhrzeit in UTC
Issue und Pull Request
Link auf den Zustimmungs-Kommentar
Bestätigung eigener Rechte
gegebenenfalls Arbeitgeberfreigabe
Status: akzeptiert / widerrufene noch nicht gemergte Einreichung / abgelehnt
```

Es sollte verschlüsselt beziehungsweise zugriffsgeschützt gespeichert und gesichert werden. Keine Ausweiskopien oder unnötigen personenbezogenen Daten erfassen.

Empfohlener Zustimmungs-Kommentar vor dem ersten Merge:

```text
Ich habe das Contributor License Agreement in CLA.md, Version <Version oder
Commit-ID>, gelesen und stimme ihm zu. Ich bestätige, dass ich den Beitrag
rechtmäßig einreichen darf. Meine gewünschte öffentliche Attribution lautet:
<Name oder Alias>.
```

Die Zustimmung wird vor dem Merge im manuellen Register dokumentiert.

## Noch festzulegen

1. endgültiger CLA-Wortlaut nach dem Live-Test,
2. konkrete Versionskonvention für das CLA,
3. Aufbewahrungsort und Sicherung des manuellen Registers,
4. Umgang mit Unternehmensbeiträgen, sobald ein solcher Fall erstmals auftritt.
