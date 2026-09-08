# HB-00 – Übersicht & Einstieg

**SMK IKS-App – Admin-Handbücher Backup, Recovery & Betrieb**
Stand: 28.08.2026 · Zielgruppe: administrierender Betreuer der IKS-App

---

## 1. Wozu diese Handbücher

Die IKS-App läuft produktiv und geschäftskritisch. Dieses Handbuchset beschreibt,
was der Admin **regelmäßig prüfen** muss und was im **Wiederherstellungsfall** zu
tun ist. Es setzt keine Kenntnis der Projekthistorie voraus – die fachlichen
Hintergründe stehen in den Projektdokumentationen, hier stehen nur die Abläufe.

| Handbuch | Inhalt | Wann |
|---|---|---|
| **HB-01** Backup-Betrieb & Monitoring | Regelmäßige Kontrollen, Alerts, Retention-Einstellungen | Laufend (wöchentlich/monatlich/quartalsweise) |
| **HB-02** PITR-Restore (Datensatz-Wiederherstellung) | Versehentlich gelöschte Datensätze über Point-in-Time-Restore zurückholen | Anlassbezogen |
| **HB-03** BACPAC-Export – Zugriff, Betrieb, Restore | Wöchentlicher Offsite-Export: überwachen, Datei holen, importieren | Laufend + anlassbezogen |
| **HB-04** Blob-Recovery | Gelöschte Dateien im Blob Storage wiederherstellen, Change Feed lesen | Anlassbezogen |

---

## 2. Erstinbetriebnahme des Admins (einmalig)

Bevor irgendetwas anderes gemacht wird:

- [ ] Eigener Admin-Account ist in der **Action Group** als Empfänger hinterlegt
      (sonst kommen keine Alerts an) → HB-01, Abschnitt 3
- [ ] Zugriff auf das Azure-Portal mit den benötigten Rollen ist vorhanden → Abschnitt 4
- [ ] Zugriff auf das GitHub-Repo `SMK-AG/SMK-IKS-App` (Actions-Tab) ist vorhanden → HB-03
- [ ] SQL-Abfrage-Werkzeug ist eingerichtet (`<TBD: SSMS / Azure Data Studio / Portal-Query-Editor>`)
- [ ] Zugangsdaten SQL-Admin sind auffindbar (`<TBD: Ablageort / Passwortmanager>`)
- [ ] Eskalationsweg ist bekannt → Abschnitt 5

---

## 3. Entscheidungsbaum: Was ist passiert?

```
Etwas ist gelöscht / falsch verändert worden
│
├─ Eine Datei (Beleg, Dokument, Protokoll) fehlt in der App
│  │
│  ├─ Der Eintrag ist in der App noch sichtbar, nur die Datei fehlt
│  │     → HB-04 (Blob-Recovery)
│  │
│  └─ Der ganze Eintrag ist verschwunden
│        → HB-02 (PITR) + HB-04 parallel
│
├─ Viele Datensätze / ganze Tabelle betroffen, fehlerhaftes Skript
│     → HB-02, Variante "Restore auf Seiten-DB" – nicht einzeln reparieren,
│       sondern Restore-DB anlegen und mit Fachseite abstimmen
│
├─ Datenbank oder SQL-Server komplett weg
│     → HB-02, Abschnitt "Vollständiger Wiederherstellungsfall"
│       Fällt der Vorfall in ein Fenster > 35 Tage: HB-03 (BACPAC) oder LTR
│
├─ Azure-Zugang kompromittiert / Verdacht auf Manipulation
│     → HB-03: der BACPAC im WORM-Container ist die einzige Kopie
│       außerhalb der Vertrauensgrenze. Sofort eskalieren.
│
└─ Ein Alert ist eingegangen, aber nichts sichtbar kaputt
      → HB-01, Abschnitt 4 (Alert-Behandlung)
```

**Wichtigste Regel:** Datenbank und Blob Storage müssen auf **denselben Zeitpunkt**
gebracht werden. Wird nur eines von beiden wiederhergestellt, entstehen Datensätze
ohne Datei oder Dateien ohne Datensatz.

---

## 4. Ressourcenübersicht

| Ressource | Prod | Stage |
|---|---|---|
| Static Web App | `smk-iks-app` | `smk-iks-app-stage` |
| Logischer SQL-Server | `smk-iks-sql` | `smk-iks-sql-stage` |
| Datenbank | `smk-iks-db` | `smk-iks-db-stage` |
| Resource Group (App) | `rg-smk-iks` | `rg-smk-iks-stage` |
| Storage Account (App-Daten) | `smkiksstorage` | `smkiksstoragestage` |
| Blob-Container (Belege) | `smk-iks-belege` | `smk-iks-belege` |
| Backup-Resource-Group | `rg-smk-iks-backup` | – |
| Backup-Storage-Account | `stsmkiksbackup` | – |
| Container BACPAC (WORM, 90 Tage) | `bacpac` | – |
| Container BACPAC-Staging | `bacpac-staging` | – |
| Log Analytics Workspace | `DefaultWorkspace-76e781a4-3c0d-46dc-b063-b2d0759975c0-WEU` (Retention 90 Tage) | `<TBD>` |
| Application Insights | `smk-iks-app` | `<TBD>` |
| GitHub-Repo | `SMK-AG/SMK-IKS-App` | – |

> **Zur Orientierung beim Suchen im Portal:**
> Die Application-Insights-Instanz für Prod heißt genauso wie die Static Web App
> (`smk-iks-app`) – bei der Suche im Portal erscheinen beide Treffer, auf den
> Ressourcentyp achten.
> Der Log Analytics Workspace ist ein von Azure automatisch angelegter
> Default-Workspace (Region West Europe); der GUID-Anteil im Namen ist die
> Subscription-ID und lässt sich nicht ändern.

**Benötigte Rollen des Admin-Accounts:** `<TBD – Vorschlag:>`
Reader auf der Subscription, Contributor auf `rg-smk-iks`,
`Storage Blob Data Reader` auf `stsmkiksbackup`, SQL-Zugriff über
`<TBD: Entra-Gruppe / SQL-Admin-Login>`.

**Resource Locks:** Auf `rg-smk-iks-backup` liegt ein `CanNotDelete`-Lock.
Der Container `bacpac` hat eine **gelockte** Immutability-Policy (90 Tage) –
Dateien darin können auch von einem Admin weder gelöscht noch überschrieben werden.
Lesen und Herunterladen ist erlaubt.

---

## 5. Eskalation & Kontakte

| Fall | Kontakt |
|---|---|
| Fachliche Rückfrage zu Daten/Modulen | `<TBD>` |
| App-Owner SMK | Marco Gerth |
| Technische Eskalation Aithoria | `<TBD>` |
| Freigabe für einen Restore auf der **Produktions-DB** | `<TBD – wird Vier-Augen-Prinzip verlangt?>` |

**Zielwerte (Arbeitshypothese, noch zu bestätigen):**
RPO < 1 Stunde · RTO 4 Stunden innerhalb der Geschäftszeiten.

---

## 6. Schutzschichten im Überblick

| Schicht | Deckt ab | Aufbewahrung | Handbuch |
|---|---|---|---|
| SQL PITR | Versehen, fehlerhafte Skripte, Datensatzverlust | 35 Tage | HB-02 |
| SQL LTR | Gesetzliche Aufbewahrung (HGB §257) | 8 Wochen / 12 Monate / 10 Jahre | HB-01 |
| Blob Soft Delete + Versionierung | Gelöschte/überschriebene Dateien | 30 Tage | HB-04 |
| Blob Change Feed | Nachweis von Löschungen außerhalb der App | dauerhaft | HB-04 |
| BACPAC-Export (WORM, separater Account) | Kompromittierter Account, Azure-seitiger Ausfall | 90 Tage rollierend | HB-03 |

> Merksatz aus der Projektdoku: PITR/LTR schützen vor eigenen Fehlern.
> Der BACPAC schützt vor dem Verlust des Azure-Accounts selbst.
