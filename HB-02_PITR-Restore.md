# HB-02 – Point-in-Time-Restore (PITR)

**SMK IKS-App** · Stand: 28.08.2026
Zweck: Gelöschte oder fehlerhaft veränderte Datensätze über den
Point-in-Time-Restore der Azure SQL Database wiederherstellen.

---

## 1. Vorbemerkung

PITR ist bei Azure SQL dauerhaft aktiv und kann nicht deaktiviert werden.
Ein PITR erzeugt **immer eine neue Datenbank** – die Live-DB wird dabei nicht
angefasst und nicht überschrieben. Das ist der Grund, warum das Verfahren
risikoarm ist: die Seiten-Datenbank dient nur als Lesequelle.

**Rückholfenster: 35 Tage.** Liegt der Vorfall außerhalb dieses Fensters, ist
PITR nicht mehr möglich → LTR-Restore oder BACPAC (HB-03).

**Vor jedem Restore klären:**
- Auf welcher Umgebung? Auf Prod nur mit Freigabe (`<TBD: Freigabeprozess>`).
- Ist auch eine Datei betroffen? Dann Blob-Wiederherstellung parallel
  mitlaufen lassen (HB-04) – DB und Blob müssen auf denselben Stand kommen.

---

## 2. Erster Schritt in jedem Fall: `audit_log` befragen

Das `audit_log` ist die primäre Anlaufstelle für die Frage
„was wurde wann von wem gelöscht und mit welcher `id`". Jeder Lösch-Handler
schreibt dort hinein, **bevor** er die Zeile entfernt.

**Zugang A – in der App:** Seite *Audit-Log* (`/audit-log`), nach Tabelle
gefiltert. Nur für Vorstands-Rollen sichtbar.

**Zugang B – per SQL:**

```sql
SELECT a.geaendert_am, a.datensatz_id, a.alt_wert_json, u.name AS wer
FROM audit_log a
LEFT JOIN users u ON u.id = a.geaendert_von
WHERE a.tabelle = '<tabellenname>' AND a.aktion = 'DELETE'
ORDER BY a.geaendert_am DESC;
```

Benötigt werden aus dem Ergebnis: **`datensatz_id`** (die `id` der gelöschten
Zeile) und **`geaendert_am`** (der Zeitpunkt, kurz vor den restauriert wird).

**Grenzen des `audit_log`, die man kennen muss:**
- `ip_adresse` und `user_agent` werden nie befüllt.
- `alt_wert_json` ist je Modul unterschiedlich vollständig:
  `ki_register_dokumente` und `software_dokumente` speichern nur
  `{ bezeichnung }`, `protokolle` nur `{ titel }`, `gesellschaftsdokumente`
  speichert `null`. Der vollständige Datensatz inklusive Blob-Pfad kommt
  deshalb immer erst aus dem PITR-`SELECT`.
- Löschungen **außerhalb** der App (Portal, Storage Explorer, direkter
  Account-Key-Zugriff) erscheinen hier nicht. Zweite Quelle dafür ist der
  Blob Change Feed → HB-04.

---

## 3. Fall A – Soft Delete (kein PITR nötig)

**Betroffene Module:** `wissen_dokumente`, `fragebogen_wissen` (KI-Underwriting)

Der Lösch-Handler setzt hier nur `aktiv = 0`. Die DB-Zeile mit allen Metadaten
(Titel, Blob-Pfad, Fremdschlüssel) bleibt erhalten. Es wird **kein** PITR benötigt.

**Ablauf:**

1. **Blob wiederherstellen**
   Portal → Container `smk-iks-belege` → *Show deleted blobs* → Blob anklicken →
   Reiter *Versions* → neueste Version → „…" → *Make current version*
2. **DB-Zeile reaktivieren**
   ```sql
   UPDATE wissen_dokumente SET aktiv = 1 WHERE id = '<id>';
   -- bzw.
   UPDATE fragebogen_wissen SET aktiv = 1 WHERE id = '<id>';
   ```
3. **Verifikation:** Eintrag ist in der App sichtbar, die Datei lässt sich öffnen.

Geschätzte Dauer: wenige Minuten.

---

## 4. Fall B – Hard Delete (PITR-Seiten-DB)

**Betroffene Module:** `protokolle`, `vr_vereinbarungen_anhaenge`,
`ki_register_dokumente`, `software_dokumente`, `gesellschaftsdokumente`

Hier führt der Lösch-Handler ein `DELETE FROM <tabelle>` aus – die Zeile ist aus
der Datenbank entfernt. Der Blob wird ebenfalls gelöscht (Best-effort).

Geschätzte Dauer: 45–75 Minuten, davon 15–30 Minuten Wartezeit für den Restore.

### Phase 1 – Einstieg über `audit_log`

Siehe Abschnitt 2. `datensatz_id` und `geaendert_am` notieren.

### Phase 2 – PITR-Seiten-Datenbank anstoßen *(zeitkritisch, sofort starten)*

Portal → SQL-**Datenbank** → *Restore*

| Feld | Wert |
|---|---|
| Datenbankname | `iks-<umgebung>-restore` |
| Restore-Zeitpunkt | kurz **vor** `geaendert_am` aus dem `audit_log` |

Dauer ca. 15–30 Minuten. Diesen Schritt zuerst starten und die übrigen Phasen
parallel bearbeiten.

> Azure SQL erlaubt keine datenbankübergreifenden Abfragen. Werte müssen als
> Literale aus der Kopie in die Live-DB übertragen werden – ein
> `INSERT ... SELECT` über beide Datenbanken funktioniert nicht.

### Phase 3 – Blob wiederherstellen *(parallel zu Phase 2)*

Portal → Container `smk-iks-belege` → *Show deleted blobs* → Version →
*Make current version*.
Den Blob-Pfad entweder aus dem `audit_log` ableiten (falls dort vorhanden) oder
aus der Restore-DB in Phase 4.

Details und Sonderfälle → HB-04.

### Phase 4 – Zeile aus der Kopie auslesen

In der Seiten-DB `iks-<umgebung>-restore`:

```sql
SELECT * FROM <tabelle> WHERE id = '<datensatz_id>';
```

### Phase 5 – Zeile in die Live-DB einfügen

**Vor dem Insert prüfen**, dass alle Fremdschlüssel-Elternsätze in der Live-DB
noch existieren: `ki_register_id`, `software_eintrag_id`, `vereinbarung_id`,
`kategorie_id`. Fehlt ein Elternsatz, muss er zuerst wiederhergestellt werden.

```sql
INSERT INTO <tabelle> (<spalten>) VALUES (<werte>);
```

Alle GUID- und Datums-/Zeitwerte in einfache Anführungszeichen setzen;
numerische Werte und `NULL` ohne Anführungszeichen.

**Pflichtfelder je Tabelle:**

| Tabelle | Spalten für den INSERT |
|---|---|
| `ki_register_dokumente` | `id`, `ki_register_id`, `bezeichnung`, `dateiname`, `blob_url`, `mime_type`, `dateigroesse_kb`, `hochgeladen_von`, `created_at` |
| `software_dokumente` | `id`, `software_eintrag_id`, `bezeichnung`, `dateiname`, `blob_url`, `mime_type`, `dateigroesse_kb`, `hochgeladen_von`, `created_at` |
| `gesellschaftsdokumente` | `id`, `typ`, `titel`, `beschreibung`, `gueltig_ab`, `version_nr`, `blob_url`, `dateiname`, `mime_type`, `dateigroesse_kb`, `hochgeladen_von`, `hochgeladen_am` |
| `protokolle` | `id`, `kategorie_id`, `titel`, `datum`, `kommentar`, `dateiname`, `blob_name`, `dateigroesse_kb`, `hochgeladen_von`, `hochgeladen_am` |
| `vr_vereinbarungen_anhaenge` | `id`, `vereinbarung_id`, `dateiname`, `blob_name`, `content_type`, `dateigroesse_kb`, `hochgeladen_von`, `created_at` |

### Phase 6 – Verifikation

Eintrag in der App sichtbar, Datei lässt sich öffnen.

### Phase 7 – Seiten-DB löschen

Portal → SQL-Datenbank `iks-<umgebung>-restore` → *Delete*.

**Nicht vergessen.** Eine liegengebliebene Restore-DB kostet Geld und wird
irgendwann für die Live-DB gehalten.

---

## 5. Sonderfall Protokoll-Transkripte

| Blob-Pfad | Verhalten | Recovery |
|---|---|---|
| `protokolle/_transkripte/{uuid}.{ext}` | Nach erfolgreicher Verarbeitung absichtlich gelöscht | **Nicht vorgesehen** – ephemer, keine Wiederherstellung |
| `protokolle/_generiert/{jobId}.docx` | Fester Name pro Job, bei Re-Run überschrieben | Regenerierbar aus `protokoll_transkript_jobs.markdown_protokoll`; ältere Fassungen über die Blob-Versionierung |

---

## 6. Vollständiger Wiederherstellungsfall

Wenn die gesamte Datenbank betroffen ist (Löschung, fehlerhaftes Massenupdate,
Ransomware-Verdacht):

1. **Nicht auf der Live-DB arbeiten.** Erst Ist-Zustand sichern, dann handeln.
2. Eskalieren (HB-00, Abschnitt 5) und die Entscheidung dokumentieren.
3. PITR-Restore auf eine **neue** Datenbank ziehen (Phase 2 oben), Zeitpunkt
   kurz vor dem Schadereignis.
4. Inhalte gegen die Fachseite verifizieren.
5. Erst dann den Umschwenk: Connection-String der App auf die wiederhergestellte
   Datenbank umstellen (Case-2-Runbook in `recovery.md`).
6. Blob-Stand auf denselben Zeitpunkt bringen (HB-04).
7. Alte DB nicht sofort löschen – als Beweismittel und Rückfallebene behalten.

**Wenn auch die Azure-Ebene selbst kompromittiert ist** (PITR/LTR liegen im
selben Account und sind dann mitbetroffen): Einstieg über den BACPAC-Export im
WORM-Container → HB-03, Abschnitt 6.

**Wenn eine ganze Region ausgefallen ist:** Geo-Restore. Setzt geo-redundanten
Backup-Storage der Datenbank voraus (HB-01, Abschnitt 2.4).

---

## 7. Referenz-Trockenlauf (Stage, 11.08.2026)

Beide Klassen wurden erfolgreich durchgespielt – das Verfahren ist erprobt:

**Klasse A (`wissen_dokumente`):** Dokument in der App gelöscht (`aktiv = 0`),
Blob-Version im Portal wiederhergestellt, `UPDATE aktiv = 1` ausgeführt,
Dokument in der App wieder sichtbar und Datei geöffnet – alle Schritte ✓.

**Klasse B (`ki_register_dokumente`):** Dokument gelöscht und 0 Zeilen im SELECT
bestätigt, Restore-DB `iks-stage-restore` mit Zeitpunkt 14:13 deployed,
vollständige Zeile aus der Restore-DB gelesen, Blob-Version wiederhergestellt,
`INSERT` in die Live-DB erfolgreich, Dokument in der App sichtbar und PDF
geöffnet, Restore-DB anschließend gelöscht – alle Schritte ✓.
