# Kostenanalyse im Azure Portal – Klickanleitung

Ziel: von der Gesamtübersicht der Subscription bis auf die Kosten einer einzelnen Datei herunterklicken. Beispiel am Ende: die bacpac-Datei in `rg-smk-iks` mit rund 0,50 €.

## Abkürzungen

| Kürzel | Bedeutung |
|---|---|
| Subscription | Abrechnungseinheit in Azure; alles darin landet auf einer Rechnung. |
| Scope | Der Bereich, für den die Kosten gerade ausgewertet werden (z. B. eine bestimmte Subscription). |
| RG (Resource Group) | Logischer Container, in dem zusammengehörige Azure-Ressourcen liegen. |
| Ressource | Ein einzelner Azure-Dienst, z. B. ein Storage Account oder eine SQL-Datenbank. |
| Meter | Der einzelne Abrechnungsposten einer Ressource (z. B. "Blob Storage – Hot LRS"). Azure rechnet nicht die Ressource als Ganzes ab, sondern ihre Meter. |
| bacpac | Exportformat einer Azure-SQL-Datenbank: Schema und Daten in einer einzigen Datei. Wird als Datei in einem Storage Account abgelegt. |

## Klickweg

1. **Cost Management öffnen**
   Obere Suchleiste im Azure Portal → `Cost Management` eingeben → Treffer öffnen.

2. **Scope einstellen**
   Oben mittig über **Scope** die gewünschte Subscription auswählen.

3. **Cost analysis aufrufen**
   Linkes Menü → **Reporting + analytics** → **Cost analysis**.

4. **Auf "All views" wechseln**
   Im Reiterfeld oben auf **All views**. Hier liegen die vorgefertigten Auswertungen. Relevant sind **Resource groups** und **Resources**.

5. **Resource groups öffnen**
   Zeigt alle Resource Groups der Subscription mit ihrer jeweiligen Kostenaufteilung. Gute Übersicht darüber, welches Projekt wie viel verursacht.

6. **In `rg-smk-iks` hineinklicken**
   Klappt die einzelnen Ressourcen dieser Resource Group auf.

7. **Einzelne Ressource aufklappen**
   Ein weiterer Klick auf eine Ressource zeigt deren Instanzen bzw. Meter – also die tatsächlichen Abrechnungsposten.

## Beispiel: die bacpac-Datei (~0,50 €)

In `rg-smk-iks` bis zum **Storage Account** durchklicken und dessen Meter aufklappen.

Wichtig für die Erwartungshaltung im Meeting: Es gibt keine eigene Ressource namens "bacpac". Die Datei liegt als Blob im Storage Account, deshalb erscheinen ihre Kosten dort als Speicherkosten – im Beispiel rund 0,50 € für den Monat.

Das ist gleichzeitig die eigentliche Aussage: Ein vollständiger Datenbank-Export kostet im Monat den Gegenwert eines Kaugummis. Backup-Kosten sind in dieser Größenordnung kein Argument gegen ein sauberes Backup-Konzept.

## Hinweis: Alternativweg über die Subscription

Statt über Cost Management kann man auch direkt über die **Subscription** gehen:

Suchleiste → `Subscriptions` → gewünschte Subscription auswählen → linkes Menü → **Kostenanalyse** (Cost analysis).

Der Unterschied ist die Darstellung: Dieser Einstieg liefert von Haus aus die visuell aufbereitete Sicht – Balken- und Flächendiagramme über den Zeitverlauf, Tortendiagramme nach Dienst, Standort oder Resource Group. Für eine Präsentation ist dieser Weg oft der dankbarere Einstieg, weil man ohne weitere Klicks ein Bild auf dem Schirm hat. Der Weg über Cost Management ist dafür besser zum Durchklicken in die Tiefe.

Praktisch: mit dem Diagramm einsteigen, um das Gesamtbild zu zeigen, und dann für das bacpac-Beispiel in die Tabellenansicht wechseln.

## Sonstiges

- **Zeitraum**: Oben rechts den Zeitraum so setzen, dass der Monat abgedeckt ist, in dem der Export tatsächlich lief – sonst steht dort 0,00 €.
- **Aktualisierung**: Kostendaten werden mit bis zu 24 Stunden Verzögerung fortgeschrieben. Ein Export von heute Morgen ist unter Umständen noch nicht sichtbar.
- **Berechtigung**: Für Cost Management wird mindestens die Rolle "Cost Management Reader" auf dem gewählten Scope benötigt. Falls jemand aus der Runde mitklicken soll, vorher klären.
