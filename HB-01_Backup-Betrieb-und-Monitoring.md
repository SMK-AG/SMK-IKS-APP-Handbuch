# HB-01 – Backup-Betrieb & Monitoring

**SMK IKS-App** · Stand: 28.08.2026
Zweck: Was der Admin regelmäßig prüft, damit im Ernstfall die Backups auch da sind.

---

## 1. Prüfkalender

| Rhythmus | Prüfung | Abschnitt |
|---|---|---|
| Wöchentlich (Mo) | BACPAC-Wochenlauf erfolgreich? | 2.1 |
| Wöchentlich | Offene Alerts gesichtet und bearbeitet? | 4 |
| Monatlich | PITR- und LTR-Retention unverändert? | 2.2 / 2.3 |
| Monatlich | Blob Data Protection am **Prod**-Storage unverändert? | 2.4 |
| Monatlich | Dateibestand im Container `bacpac` plausibel (~13 Dateien)? | 2.1 |
| Quartalsweise | Restore-Test durchführen und protokollieren | 5 |
| Quartalsweise | Empfängerliste der Action Groups aktuell? | 3 |
| Jährlich | Zugriffsrechte/Service-Principal-Rollen überprüfen | 6 |

Ergebnisse bitte in `<TBD: Ablageort Prüfprotokoll – OneNote? Repo?>` festhalten.

---

## 2. Die einzelnen Kontrollen

### 2.1 BACPAC-Wochenlauf

Der Export läuft **jeden Sonntag um 02:00 UTC** als GitHub-Actions-Workflow.

**Prüfung A – Lauf erfolgreich:**
1. GitHub → Repo `SMK-AG/SMK-IKS-App` → Tab **Actions** → Workflow *BACPAC Wochen-Export*
2. Der letzte Lauf muss grün sein. Ein roter Lauf löst zusätzlich eine Mail an
   Committer/Owner aus – diese Mail ist kein Ersatz für die eigene Sichtprüfung.
3. Bei rotem Lauf → HB-03, Abschnitt „Troubleshooting"

**Prüfung B – Datei tatsächlich angekommen:**
1. Portal → Storage Account `stsmkiksbackup` → Containers → `bacpac`
2. Es muss ein Blob `smk-iks-db-JJJJ-MM-TT.bacpac` mit dem Datum des letzten
   Sonntags existieren, Größe plausibel (BACPACs sind komprimiert, grob 20–40 %
   der DB-Größe).
3. Im eingeschwungenen Zustand liegen bei 90 Tagen Retention und wöchentlichem
   Export **ca. 13 Dateien** im Container. Deutlich weniger = es fehlen Läufe.
   Deutlich mehr = der Cleanup-Step greift nicht.

**Prüfung C – keine Altlasten:**
1. Container `bacpac-staging` sollte leer oder nahezu leer sein. Reste älter als
   7 Tage räumt der Workflow selbst auf; bleiben Dateien liegen, ist der
   Cleanup-Step gescheitert.
2. Auf dem SQL-Server `smk-iks-sql` darf **keine** Datenbank mit dem Suffix
   `-bacpac-copy` stehen. Eine verwaiste Kopie verursacht Kosten und lässt den
   nächsten Lauf mit `database already exists` scheitern → löschen.

### 2.2 SQL Point-in-Time-Restore (PITR)

PITR ist bei Azure SQL **dauerhaft aktiv** und kann nicht abgeschaltet werden;
Full-, Differential- und Log-Backups laufen ohne Konfiguration.
Einstellbar ist nur die **Retention**.

**Prüfpfad:** Portal → **logischer SQL-Server** (nicht die Datenbank) →
*Backups* → Tab *Retention policies* → Zeile der Datenbank → *Configure*
→ Abschnitt *Point-in-time restore configuration*

| Sollwert | Begründung |
|---|---|
| **35 Tage** | Maximum. Deckt das 30-Tage-Fenster des Blob-Schutzes ab. Bei nur 7 Tagen wäre ab Tag 8 keine vollständige Wiederherstellung mehr möglich, obwohl der Blob noch 30 Tage lang wiederherstellbar wäre. |

> **Status:** Umgesetzt – die Retention steht auf **35 Tagen**.
> Weicht der Wert bei einer Prüfung davon ab, wurde er nachträglich verändert:
> auf 35 Tage zurücksetzen und der Ursache nachgehen (Activity Log).

### 2.3 Long-Term Retention (LTR)

**Prüfpfad:** Portal → logischer SQL-Server → *Backups* → Tab *Retention policies*
→ *Configure* → Abschnitt *Long-term retention*

| Backup-Typ | Sollwert |
|---|---|
| Wochen-Backup | 8 Wochen |
| Monats-Backup | 12 Monate |
| Jahres-Backup | 10 Jahre (Woche des Jahres: KW 1) |

Die 10 Jahre bilden die handelsrechtliche Aufbewahrung (HGB §257) ab. LTR ist
nicht für schnelle Restores gedacht – ein LTR-Restore erzeugt immer eine neue
Datenbank und dauert entsprechend.

### 2.4 Blob Data Protection

**Prüfpfad:** Portal → Storage Account `smkiksstorage` → *Data management* →
*Data protection*

**Sollwerte Produktion:**

| Einstellung | Sollwert |
|---|---|
| Soft delete for blobs | An, 30 Tage |
| Soft delete for containers | An, 30 Tage |
| Versioning for blobs | An, Versionen löschen nach 30 Tagen |
| Blob change feed | An |

> **Stage weicht bewusst ab.** Auf `smkiksstoragestage` liegen keine
> schützenswerten Produktivdaten, ein gleichwertiger Schutz wäre dort nur
> zusätzlicher Speicherverbrauch ohne Nutzen. Die Stage-Einstellungen sind
> deshalb **kein Prüfpunkt** – eine Abweichung dort ist kein Befund.
> Prüfpflichtig ist ausschließlich der Prod-Storage.

**Ebenfalls prüfen:** Backup-Storage-Redundanz der Datenbank.
Pfad: **Datenbank** (nicht Server) → *Compute + storage* → unten
*Backup storage redundancy*. Nur mit **geo-redundantem** Backup-Storage ist ein
Geo-Restore bei Regionsausfall überhaupt möglich.
`<TBD: Ist die Entscheidung geo vs. lokal inzwischen getroffen – Stichwort
Datenlokation?>`

### 2.5 Logging

| Prüfung | Sollwert |
|---|---|
| Log Analytics Workspace, Data Retention | 90 Tage |
| Application Insights | je eine Instanz pro Umgebung (Prod/Stage) |
| Diagnostic Setting am Prod-Storage | `StorageWrite` + `StorageDelete` → Log Analytics Workspace Prod (`StorageRead` bewusst weggelassen: hohes Volumen, geringer Nutzen) |

---

## 3. Alerting: Action Groups pflegen

**Pfad:** Portal → *Monitor* → *Alerts* → *Action groups*

Aktuell existiert eine Action Group mit E-Mail-Benachrichtigung an **M.G.** und den
**Aithoria-Admin-Account**.

> **Beim Onboarding eines neuen Admins zwingend:** eigenen Account als Empfänger
> ergänzen. Ohne diesen Schritt laufen alle Alerts ins Leere.
> Ablauf: Action Group öffnen → *Notifications* → Zeile hinzufügen
> (Typ *Email/SMS/Push/Voice*) → Name und Adresse → *Save*.

**Geplante Aufteilung nach Eskalationsstufe – noch nicht umgesetzt, muss mit
SMK und Aithoria abgestimmt werden.** Bis dahin gilt die eine bestehende
Action Group für alle Alerts.

| Action Group | Kanal | Zuständig für |
|---|---|---|
| `ag-iks-kritisch` | SMS | `Microsoft.Sql/servers/databases/delete`, `Microsoft.Sql/servers/delete` |
| `ag-iks-info` | E-Mail | `Microsoft.Sql/servers/firewallRules/write` |

Abzustimmen sind insbesondere: wer im kritischen Kanal per SMS erreichbar sein
soll, ob es Erreichbarkeitszeiten gibt und wie die Zuordnung der übrigen
Operationen erfolgt.

---

## 4. Alert-Behandlung

Eingerichtete Activity-Log-Alerts:

| Operation | Bedeutung | Sofortmaßnahme |
|---|---|---|
| `Microsoft.Sql/servers/databases/delete` | Eine Datenbank wurde gelöscht | **Kritisch.** Prüfen, ob es die Produktions-DB war. Wenn ja: sofort eskalieren und HB-02 „Vollständiger Wiederherstellungsfall". Häufigster harmloser Grund: die temporäre `-bacpac-copy` des Wochenlaufs. |
| `Microsoft.Sql/servers/delete` | Ein SQL-Server wurde gelöscht | **Kritisch.** Sofort eskalieren. |
| `Microsoft.Sql/servers/firewallRules/write` | Firewall-Regel geändert | Prüfen, wer die Regel gesetzt hat und ob sie beabsichtigt war. Nicht mehr benötigte IP-Freigaben entfernen. |
| `Microsoft.Sql/servers/databases/write` | Datenbank angelegt oder geändert | Meist unkritisch (Restore-DBs, `-bacpac-copy`). Bei unerklärlichen Einträgen nachgehen. |

**Bei jedem Alert prüfen:** Portal → *Monitor* → *Activity log* → betroffene
Ressource → Spalte *Initiated by*. Der Auslöser sagt in der Regel sofort, ob es
ein automatisierter Lauf oder eine Person war.

**Alert-Test:** Die Alerts wurden getestet, dabei waren zunächst die Conditions
nicht korrekt gesetzt und mussten überarbeitet werden. Der Fall
`firewallRules/write` wurde anschließend verifiziert (lokale IP zur Whitelist
hinzugefügt → Alert löste aus, Mail kam an). Nach jeder Änderung an einer Action
Group oder einem Alert sollte ein solcher Test wiederholt werden.

---

## 5. Restore-Tests

Ein Backup, das nie zurückgespielt wurde, ist kein Backup. Quartalsweise
mindestens eines der folgenden Szenarien durchspielen und Ergebnis protokollieren:

| Szenario | Anleitung | Aufwand |
|---|---|---|
| Klasse-A-Wiederherstellung (Soft Delete) | HB-02, Abschnitt 3 | gering |
| Klasse-B-Wiederherstellung über PITR-Seiten-DB | HB-02, Abschnitt 4 | mittel, ca. 1 h |
| BACPAC-Import in eine neue DB | HB-03, Abschnitt 6 | mittel |

Tests grundsätzlich **auf Stage** durchführen. Ein Trockenlauf beider Klassen
wurde am 11.08.2026 auf Stage erfolgreich abgeschlossen.

---

## 6. Zugriffsrechte jährlich prüfen

| Prinzipal | Scope | Rolle | Soll |
|---|---|---|---|
| Service Principal `github-bacpac-export` | logischer SQL-Server | `SQL DB Contributor` | unverändert |
| Service Principal `github-bacpac-export` | `stsmkiksbackup` | `Storage Account Contributor` | unverändert |

Der Service Principal hat bewusst **keine** Rechte auf dem App-Storage-Account
und kann außerhalb dieser zwei Scopes nichts anfassen (kleiner Blast Radius).
Kommen dort Rollen hinzu, ist das begründungspflichtig.

Ebenfalls prüfen: Auf dem SQL-Server muss unter *Networking* die Einstellung
**„Allow Azure services and resources to access this server" = On** stehen –
der Azure Import/Export-Dienst greift darüber auf die DB zu. Ohne diese
Einstellung scheitert der BACPAC-Export mit einem Firewall-Fehler.

Der Key-Zugriff (Access Keys) auf `stsmkiksbackup` muss **aktiv** bleiben:
Export und Cleanup des Workflows benötigen den Account-Key.
