# HB-04 – Blob-Recovery (Dateien im Storage)

**SMK IKS-App** · Stand: 28.08.2026
Zweck: Gelöschte oder überschriebene Dateien im Blob Storage wiederherstellen
und nachvollziehen, wer wann was gelöscht hat.

---

## 1. Was wo liegt

Alle Blob-Daten liegen in **einem** privaten Container: `smk-iks-belege`.
Der Container wird beim ersten API-Start per `createIfNotExists()` angelegt.

| Präfix | Inhalt | Reproduzierbar? |
|---|---|---|
| `protokolle/{kategorie}/{uuid}.pdf` | Hochgeladene Protokoll-PDFs | Nein – Original |
| `wissen/{bereich}/{ts}-{name}` | Wissensdokumente | Nein – Original |
| `ki-register/{kiId}/{ts}-{name}` | Dokumente KI-Register | Nein – Original |
| `software/{swId}/{ts}-{name}` | Belege Software-Verzeichnis | Nein – Original |
| `gesellschaft/{typ}/{ts}-{name}` | Dokumente Eigentümerstruktur | Nein – Original |
| `vr-vereinbarungen/{id}/{ts}_{name}` | Anhänge VR-Vereinbarungen | Nein – Original |
| `ki-underwriting/wissen/{oid}/{ts}_{name}` | Wissensdateien Underwriting | Nein – Original |
| `protokolle/_transkripte/{uuid}.{ext}` | Quell-Transkripte (Audio/Text) | Ephemer – wird nach Verarbeitung gelöscht |
| `protokolle/_generiert/{jobId}.docx` | KI-generiertes Protokoll (DOCX) | Ja – Markdown liegt in der DB, regenerierbar |

Aus dem Präfix lässt sich also direkt ablesen, zu welchem Modul eine Datei
gehört – hilfreich, wenn man vom Blob-Pfad rückwärts zum Datensatz sucht.

---

## 2. Der aktive Schutz

**Pfad:** Portal → Storage Account → *Data management* → *Data protection*

| Einstellung | Wert |
|---|---|
| Soft delete for blobs | An, 30 Tage |
| Soft delete for containers | An, 30 Tage |
| Versioning for blobs | An, Versionen löschen nach 30 Tagen |
| Blob change feed | An |

**Wirkungsweise:** Mit aktiver Versionierung wandert ein gelöschter Blob nicht in
einen „Papierkorb", sondern wird als **frühere Version** aufbewahrt. Die App ruft
`deleteBlob(..., { deleteSnapshots: 'include' })` auf – dieser Aufruf ist mit
Versionierung verträglich, er löscht den Basis-Blob, nicht eine konkrete Version.
Deshalb war für die Härtung kein Code-Change nötig.

**Rückholfenster: 30 Tage.** Danach ist die Datei endgültig weg – abgesehen von
dem, was ein BACPAC-Import nicht abdeckt (der enthält nur die Datenbank, keine
Blobs).

---

## 3. Eine gelöschte Datei wiederherstellen

1. Portal → Storage Account → *Containers* → `smk-iks-belege`
2. Oben **Show deleted blobs** aktivieren
3. Den betroffenen Blob anklicken – Suche über den Pfad, siehe Abschnitt 1
4. Reiter **Versions**
5. Die gewünschte (in der Regel neueste) Version auswählen
6. „…" → **Make current version**
7. Verifikation: Datei lässt sich in der App öffnen

Wenn der Blob-Pfad unbekannt ist:
- Steht er im `audit_log`? (nicht in allen Modulen – siehe HB-02, Abschnitt 2)
- Sonst aus der PITR-Seiten-DB auslesen (`blob_url` bzw. `blob_name`), HB-02 Phase 4
- Sonst über den Change Feed suchen (Abschnitt 5)

---

## 4. Eine überschriebene Datei zurückholen

Gleicher Weg wie Abschnitt 3, nur ohne *Show deleted blobs*: Blob anklicken →
Reiter **Versions** → ältere Version auswählen → *Make current version*.

Relevant vor allem für `protokolle/_generiert/{jobId}.docx` – dieser Blob hat
einen festen Namen pro Job und wird bei einem Re-Run überschrieben. Alternativ
kann die Datei aus `protokoll_transkript_jobs.markdown_protokoll` neu generiert
werden.

---

## 5. Blob Change Feed: wer hat wann was gelöscht

Der Change Feed ist ein dauerhafter, chronologischer Storage-seitiger Log aller
Blob-Operationen (Erstellen, Löschen, Ändern) inklusive Blob-Pfad und Zeitstempel.

Er ist der **einzige** Log, der Löschungen **außerhalb der App** erfasst – also
per Storage Explorer, Portal oder direktem Account-Key-Zugriff. Solche
Löschungen erscheinen nicht im `audit_log` der Anwendung.

| | |
|---|---|
| Ablageort | versteckter Container `$blobchangefeed` |
| Format | Avro |
| Zugriff | Azure Storage Explorer oder Azure SDK |

**Vorgehen mit dem Storage Explorer:** Anmelden → Storage Account →
Blob Containers → versteckte Container einblenden → `$blobchangefeed` →
die Avro-Segmente nach Datum herunterladen und auswerten.

**Ergänzend:** Am Prod-Storage ist ein Diagnostic Setting mit `StorageWrite` +
`StorageDelete` gegen den Log Analytics Workspace Prod eingerichtet.
`StorageRead` wurde bewusst weggelassen (hohes Volumen, geringer
Erkenntnisgewinn). Über Log Analytics lassen sich Löschvorgänge damit auch per
KQL abfragen – bequemer als über den Change Feed, dafür nur innerhalb der
Workspace-Retention von 90 Tagen.

---

## 6. Zusammenspiel mit der Datenbank

**Die wichtigste Regel:** Ein Restore muss Datenbank und Blob Storage auf
**denselben Zeitpunkt** bringen. Werden die Fenster unterschiedlich konfiguriert
oder wird nur eine Seite wiederhergestellt, entstehen

- Datensätze ohne zugehörigen Blob (Eintrag in der App, Datei öffnet nicht), oder
- Blobs ohne zugehörigen Datensatz (Datei liegt im Storage, ist aber unerreichbar).

Deshalb ist die PITR-Retention auf 35 Tage auszurichten – passend zum
30-Tage-Fenster des Blob-Schutzes (HB-01, Abschnitt 2.2).

Welches Modul welchen Recovery-Weg braucht, steht in HB-02:
Klasse A (Soft Delete, DB-Zeile bleibt) versus Klasse B (Hard Delete,
PITR erforderlich).

---

## 7. Azure Backup for Blobs (Vaulted Backup)

Zusätzlich zum Soft Delete und zur Versionierung im Storage Account selbst ist
ein **vaulted Backup** eingerichtet. Der Unterschied ist wesentlich:
Soft Delete und Versionierung liegen im selben Storage Account wie die Daten und
fallen mit ihm gemeinsam aus. Das vaulted Backup legt eine Kopie in einem
separaten Backup Vault ab – also außerhalb des Storage Accounts und damit
außerhalb der Reichweite eines kompromittierten oder gelöschten Accounts.

| | |
|---|---|
| Backup Vault | `<TBD: Name>` |
| Backup Policy | `<TBD: Name>` |
| Backup-Typ | Vaulted Backup (nicht nur Operational) |
| Retention | **90 Tage** |
| Geschützter Storage Account | `smkiksstorage` |
| Geschützter Container | `smk-iks-belege` |

**Prüfpfad:** Portal → *Backup center* bzw. Backup Vault `<TBD: Name>` →
*Backup instances* → Instanz des Storage Accounts.
Prüfen: Status *Protected*, letzter erfolgreicher Backup-Lauf, keine
fehlgeschlagenen Jobs unter *Backup jobs*.

**Wiederherstellung:** Backup Vault → *Backup instances* → Instanz auswählen →
*Restore*. Wiederherstellung auf einen **anderen** Zielcontainer oder
Storage Account durchführen, nicht direkt über den Live-Container –
sonst wird ein bereits korrigierter Stand wieder überschrieben.

> **Reihenfolge im Ernstfall:** Für einzelne Dateien innerhalb von 30 Tagen ist
> der Weg über Soft Delete und Versionierung (Abschnitt 3) schneller und
> präziser. Das vaulted Backup ist die Rückfallebene für den Fall, dass der
> Storage Account selbst betroffen ist oder viele Dateien gleichzeitig
> wiederhergestellt werden müssen.

> **Hinweis zur Aufbewahrungsfrist:** Die 90 Tage decken den operativen
> Wiederherstellungsbedarf ab, nicht die handelsrechtliche Aufbewahrung von
> 10 Jahren (HGB §257). Für die Datenbank übernimmt das die SQL-LTR
> (HB-01, Abschnitt 2.3). `<TBD: Wie ist die Langzeitaufbewahrung für die
> Belege im Blob Storage abgedeckt?>`
