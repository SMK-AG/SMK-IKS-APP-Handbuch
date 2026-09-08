# HB-03 – BACPAC-Export: Zugriff, Betrieb und Restore

**SMK IKS-App** · Stand: 28.08.2026
Zweck: Den wöchentlichen Offsite-Export überwachen, an die Exportdateien
herankommen und daraus eine Datenbank wiederherstellen.

---

## 1. Wozu der BACPAC-Export existiert

PITR und LTR sind Azure-eigene Mechanismen **innerhalb desselben Accounts**.
Bei kompromittiertem Account, fehlerhaftem Skript oder Azure-seitigem Fehler
können Datenbank und Backups gemeinsam verloren gehen.

Der BACPAC-Export ist die logische Kopie (Schema + Daten) in einem separaten
Storage-Account mit eigenem Zugriffsschlüssel und WORM-Policy:

- Importierbar in **jede** SQL-Server-Instanz – lokal, anderer Anbieter, Azure.
  Kein Vendor-Lock-in.
- Liegt außerhalb der Vertrauensgrenze von App und Produktions-DB.
- Innerhalb der Immutability-Frist unlöschbar, auch für einen kompromittierten Admin.

**Abgrenzung:** Die Langzeit-/Generationenaufbewahrung (10 Jahre, HGB §257)
macht LTR. Der BACPAC ist die unabhängige, aktuelle Kopie unter eigener
Kontrolle – 90 Tage rollierend, ca. 13 Dateien im eingeschwungenen Zustand.

---

## 2. Zugriff auf die Exportdateien

### 2.1 Wo die Dateien liegen

| | |
|---|---|
| Resource Group | `rg-smk-iks-backup` (bewusst getrennt von der App-RG, `CanNotDelete`-Lock) |
| Storage Account | `stsmkiksbackup` |
| Container | `bacpac` – Ziel, Immutability-Policy 90 Tage, **gelockt** |
| Container | `bacpac-staging` – Zwischenablage, ohne Immutability |
| Dateiname | `<db-name>-JJJJ-MM-TT.bacpac`, z. B. `smk-iks-db-2026-08-23.bacpac` |

### 2.2 Benötigte Rechte

Zum reinen Lesen und Herunterladen genügt **`Storage Blob Data Reader`** auf
`stsmkiksbackup`. Für das Auslesen des Account-Keys wird
`Storage Account Contributor` benötigt – das braucht ein Admin im Normalbetrieb
**nicht**, das ist die Rolle des Service Principals des Workflows.

`<TBD: Bekommt der zukünftige Admin Storage Blob Data Reader oder eine weiter
gefasste Rolle?>`

### 2.3 Weg A – Azure-Portal (Standardweg)

1. Portal → Storage Account `stsmkiksbackup` → *Data storage* → **Containers**
2. Container `bacpac` öffnen
3. Gewünschte `.bacpac`-Datei anklicken → **Download**

Die Dateien sind nach Datum benannt, die Liste ist damit chronologisch sortierbar.

### 2.4 Weg B – Azure Storage Explorer

Sinnvoll bei größeren Dateien und für den Blob Change Feed (HB-04).
Anmeldung mit dem eigenen Azure-Konto, danach
Subscription → Storage Accounts → `stsmkiksbackup` → Blob Containers → `bacpac`.

### 2.5 Weg C – Azure CLI (PowerShell)

```powershell
az login

# Verfügbare Exporte auflisten
az storage blob list `
  --account-name stsmkiksbackup `
  --container-name bacpac `
  --auth-mode login `
  --query "[].{Name:name, Groesse:properties.contentLength, Datum:properties.lastModified}" `
  -o table

# Einen Export herunterladen
az storage blob download `
  --account-name stsmkiksbackup `
  --container-name bacpac `
  --name "smk-iks-db-2026-08-23.bacpac" `
  --file "C:\temp\smk-iks-db-2026-08-23.bacpac" `
  --auth-mode login
```

### 2.6 Was **nicht** geht

Der Container `bacpac` hat eine **gelockte** Time-based-Retention-Policy über
90 Tage. Innerhalb dieser Frist kann niemand – auch kein Admin, auch nicht der
Abonnementinhaber – eine Datei löschen oder überschreiben. Ein Löschversuch im
Portal muss fehlschlagen; das ist gewollt und war Teil des Abnahmetests.

Lesen und Herunterladen ist davon nicht betroffen und jederzeit erlaubt.

> Der gelockte Zustand ist **unumkehrbar**. Die Frist lässt sich verlängern,
> aber nicht verkürzen. Der Container selbst kann nicht gelöscht werden, solange
> geschützte Blobs darin liegen.

---

## 3. Wie der Export funktioniert

**Workflow-Datei:** `.github/workflows/bacpac-export.yml`
**Repo:** `SMK-AG/SMK-IKS-App`

| Trigger | Beschreibung |
|---|---|
| `schedule` (Cron `0 2 * * 0`) | jeden Sonntag 02:00 UTC |
| `workflow_dispatch` | manueller Start über den Actions-Tab |

> Der Workflow läuft zwingend von `main`, weil das Entra Federated Credential
> auf diesen Branch beschränkt ist. Ein Lauf von einem anderen Branch scheitert
> bereits beim Login.

### 3.1 Zweistufiges Export-Muster

Ein BACPAC-Export kann **nicht direkt in einen WORM-Container schreiben**:
Der Export schreibt den Blob in mehreren Operationen, was eine
Immutability-Policy verbietet. Deshalb der Umweg über einen Staging-Container.

```
Live-DB ──copy──▶ DB-Kopie ──export──▶ bacpac-staging ──copy──▶ bacpac (WORM)
                     │                        │                     │
                  (gelöscht)              (gelöscht)          (90 Tage, unlöschbar)
```

Die server-seitige Kopie von `bacpac-staging` nach `bacpac` ist eine einmalige
Neuanlage eines frischen Blobs und unter WORM erlaubt. Anschließend wird
`copy.status` gepollt, bis `success` gemeldet wird.

### 3.2 Die Schritte des Workflows

| # | Step | Zweck |
|---|---|---|
| 1 | Azure Login per OIDC | Anmeldung ohne gespeichertes Azure-Secret über GitHubs OIDC-Token |
| 2 | Set names | Setzt `STAMP` (Datum), `COPY_DB` (DB-Kopie), `BLOB` (`<db>-<datum>.bacpac`) |
| 3 | Create transactionally consistent DB copy | `az sql db copy` – der Export einer laufenden DB wäre nicht transaktionskonsistent |
| 4 | Read storage key | Account-Key des Backup-Storage-Accounts, maskiert an die späteren Steps übergeben |
| 5 | Export BACPAC to staging | `az sql db export` in den Container `bacpac-staging` |
| 6 | Copy BACPAC into immutable container | Server-seitige Kopie `bacpac-staging` → `bacpac`, danach Polling auf `success` |
| 7 | Remove staging copy | Löscht den Staging-Blob |
| 8 | Delete DB copy (`if: always()`) | Entfernt die temporäre DB-Kopie auch im Fehlerfall – keine verwaiste Kopie |
| 9 | Clean up old exports | Löscht in `bacpac` Exporte älter als 90 Tage, in `bacpac-staging` Reste fehlgeschlagener Läufe älter als 7 Tage |

### 3.3 Konfiguration in GitHub

**Secrets** (Repo → Settings → Secrets and variables → Actions):
`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`,
`SQL_ADMIN_USER` (= `AZURE_SQL_USER`), `SQL_ADMIN_PASSWORD` (= `AZURE_SQL_PASSWORD`)

**Variables:**

| Variable | Beispiel | Stolperstelle |
|---|---|---|
| `SQL_RESOURCE_GROUP` | `rg-smk-iks` | RG des SQL-Servers, **nicht** die Backup-RG |
| `SQL_SERVER` | `smk-iks-sql` | kurzer Name, **ohne** `.database.windows.net` |
| `SQL_DATABASE` | `smk-iks-db` | reiner DB-Name, **ohne** `server/`-Präfix |
| `BACKUP_STORAGE_ACCOUNT` | `stsmkiksbackup` | – |

### 3.4 Authentifizierung (Federated Credential)

App Registration `github-bacpac-export` in Entra ID – kein Redirect URI,
kein Client Secret. Das Federated Credential wurde im Modus *Other issuer*
angelegt:

| Feld | Wert |
|---|---|
| Issuer | `https://token.actions.githubusercontent.com` |
| Subject | `repo:SMK-AG/SMK-IKS-App:ref:refs/heads/main` |
| Audience | `api://AzureADTokenExchange` |

> Der Subject ist **case-sensitive** und muss zeichengenau dem entsprechen, was
> GitHub sendet. Nur SQL-Admin-User und -Passwort bleiben als GitHub-Secret,
> da die Export-API sie verlangt.

**Was das für den Admin heißt:** Es gibt kein Azure-Passwort, das abläuft. Wird
das Repo umbenannt, verschoben oder der Default-Branch geändert, muss das
Federated Credential angepasst werden – sonst schlägt der Login fehl.
Das SQL-Admin-Passwort muss bei jeder Rotation **an beiden Stellen** aktualisiert
werden: in den GitHub-Secrets und in den App-Settings.

---

## 4. Manuellen Lauf auslösen

Nötig z. B. vor einem größeren Deployment, nach einem fehlgeschlagenen Lauf oder
für einen Test.

1. GitHub → Repo `SMK-AG/SMK-IKS-App` → Tab **Actions**
2. Links Workflow *BACPAC Wochen-Export* wählen
3. Rechts **Run workflow** → Branch `main` → *Run workflow*
4. Lauf beobachten. Läuft er durch, liegt danach eine neue Datei im Container
   `bacpac` (Abschnitt 2.3).

Ein zusätzlicher Lauf ist unkritisch: Der Dateiname enthält das Datum, ein
zweiter Lauf am selben Tag würde denselben Namen erzeugen und wegen der
WORM-Policy scheitern. `<TBD: bestätigen – oder wird der Stamp feiner aufgelöst?>`

---

## 5. Troubleshooting

| Fehlerbild | Ursache | Vorgehen |
|---|---|---|
| `AADSTS700213` beim Login | Federated-Credential-Subject stimmt nicht: Case-Sensitivity, `@`-IDs statt Namen, falscher Branch | Entra ID → App Registration `github-bacpac-export` → Federated credentials → Subject zeichengenau gegen `repo:SMK-AG/SMK-IKS-App:ref:refs/heads/main` prüfen |
| `Resource not found` bei `az sql db copy` | Falsche `SQL_*`-Variables | RG/Server/DB prüfen: Server **ohne** Domain-Suffix, DB **ohne** `server/`-Präfix, RG des SQL-Servers (nicht die Backup-RG) |
| `blob is immutable due to a policy` beim Export | Es wurde direkt in den WORM-Container exportiert | Export muss nach `bacpac-staging` gehen – Workflow-Datei gegen den Soll-Ablauf in Abschnitt 3.2 prüfen |
| `database already exists` bei der DB-Kopie | Verwaiste `*-bacpac-copy` aus einem früheren Lauf | Auf dem SQL-Server die Datenbank mit Suffix `-bacpac-copy` löschen, danach Lauf wiederholen |
| Export scheitert mit Firewall-Fehler | „Allow Azure services and resources to access this server" ist ausgeschaltet | SQL-Server → *Networking* → Einstellung auf **On** setzen |
| Cleanup löscht nichts | Retention im Cleanup-Step und Immutability-Policy des Containers weichen voneinander ab | Beide müssen auf **90 Tage** stehen |

**Nach jedem gescheiterten Lauf prüfen:** Ist die temporäre DB-Kopie
(`*-bacpac-copy`) wirklich weg? Der Löschschritt steht auf `if: always()`,
aber bei einem Abbruch des Runners kann sie stehenbleiben.

---

## 6. Restore aus einem BACPAC

### 6.1 Import über das Azure-Portal

1. Portal → SQL-Server `smk-iks-sql` → **Import database**
2. `.bacpac` aus dem Container `bacpac` wählen – Lesen aus dem WORM-Container
   ist erlaubt
3. **Neuen** DB-Namen vergeben, z. B. `smk-iks-db-restore-test`.
   Niemals auf die bestehende Produktions-DB importieren.
4. Tier und SQL-Admin-Credentials angeben
5. Fortschritt verfolgen unter SQL-Server → *Import/Export history*
6. Verifikation per `SELECT COUNT(*)` auf den Kern-Tabellen, Abgleich gegen den
   erwarteten Stand
7. Für den Produktivfall: Connection-String der App auf die neue Datenbank
   umstellen. Erst nach erfolgreicher Verifikation aus Schritt 6 und nur mit
   Freigabe (HB-00, Abschnitt 5).

### 6.2 Import lokal oder außerhalb von Azure

```powershell
SqlPackage /Action:Import `
  /SourceFile:"C:\temp\smk-iks-db-2026-08-23.bacpac" `
  /TargetServerName:"<server>" `
  /TargetDatabaseName:"<neue-db>" `
  /TargetUser:"<user>" `
  /TargetPassword:"<passwort>"
```

Das ist der eigentliche Portabilitätsvorteil des BACPAC-Formats: die Datei
lässt sich in jede SQL-Server-Instanz importieren, auch ohne Azure.

### 6.3 Wichtige Einschränkung

Ein BACPAC ist ein **Wochenstand**. Zwischen dem letzten Export und dem
Schadereignis liegen bis zu sieben Tage Datenverlust. Der BACPAC ist deshalb die
letzte Rückfallebene, nicht das Mittel erster Wahl – für alles innerhalb des
PITR-Fensters ist HB-02 der richtige Weg.

Ebenso zu beachten: Der BACPAC enthält **nur die Datenbank**, nicht den Blob
Storage. Ein Restore aus BACPAC muss immer mit einer passenden
Blob-Wiederherstellung kombiniert werden (HB-04).

---

## 7. Kosten und Aufbewahrung

- Retention **90 Tage**. Cleanup-Step und Container-Immutability-Policy müssen
  denselben Wert haben, sonst kann der Cleanup nicht löschen.
- Im eingeschwungenen Zustand ca. **13 Dateien** im `bacpac`-Container.
- BACPACs sind komprimiert (~20–40 % der DB-Größe); der Speicher liegt im
  niedrigen einstelligen Euro-Bereich pro Jahr (LRS, Cool-Tier).
- Die temporäre DB-Kopie kostet bei der Serverless-DB nur die Compute-Minuten
  ihrer kurzen Lebensdauer.
