# Offline-first field time-tracking app

## Zielsetzung
- **Plattformen:** Native iOS- und Android-App.
- **Nutzung ohne Netz:** Alle Datenerfassung funktioniert vollständig offline; Synchronisation erfolgt automatisch bei verfügbarer Verbindung.
- **Erfasste Daten:** GPS-Standort, Startzeit, Endzeit, Textnotizen, Material, Projektnummer.
- **Eingabe per Sprache:** Diktat in Deutsch, Englisch und Spanisch für alle Textfelder.

## Architekturüberblick
- **Client:** React Native + TypeScript für Cross-Platform UI und native Module.
- **Lokale Persistenz:** SQLite (über `react-native-sqlite-storage`) oder WatermelonDB für robuste Offline-Queries und Konfliktauflösung.
- **Synchronisation:** Background sync (WorkManager / BGTaskScheduler) mit REST-API; Upload in Batches, Backoff bei Fehlern, Retry bei Netzrückkehr.
- **Standort:** Native Geolocation-API mit präziser Genauigkeit (z. B. `react-native-geolocation-service`); Einholen von Berechtigungen (Foreground) und optional Hintergrundtracking für offene Sessions.
- **Spracheingabe:** Native Speech-to-Text (`SpeechRecognizer` auf Android, `SFSpeechRecognizer` auf iOS) mit Fallback auf OS-Spracherkennung; UI zur Sprachwahl (DE/EN/ES).
- **Sicherheit & Datenschutz:** Alle Daten lokal verschlüsselt (SQLCipher oder EncryptedStorage). Transportverschlüsselung via HTTPS/TLS. Optionales App-PIN/BIOMETRICS zum Öffnen.

## Datenmodell (vereinfacht)
- **projects**: `id`, `name`, `number`, `created_at`, `updated_at`.
- **materials**: `id`, `name`, `unit`, `created_at`, `updated_at`.
- **entries**: `id`, `project_id`, `material_id`, `note`, `language`, `start_time`, `end_time`, `gps_lat`, `gps_lon`, `gps_accuracy`, `synced_at`, `created_at`, `updated_at`.
- **sync_queue**: lokale Tabelle mit Pending-Änderungen (operation, payload, version).

## Kern-Workflows
- **Neue Zeiterfassung starten**
  - Standort abrufen (mit Anzeige von Genauigkeit).
  - Startzeit automatisch auf „jetzt“ setzen; projektnummer/material via Auswahl oder Freitext.
  - Notizfeld per Spracheingabe (DE/EN/ES) oder Tastatur.
  - Eintrag wird lokal persistiert und in `sync_queue` markiert.
- **Ende erfassen**
  - Endzeit setzen (automatisch „jetzt“ oder manuell).
  - Optional weitere Notiz via Spracheingabe.
  - Eintrag aktualisieren; Änderung landet in `sync_queue`.
- **Offline/Online Handling**
  - Alle Aktionen funktionieren offline.
  - Sync-Service prüft Netzwerkstatus; bei Verbindung werden `sync_queue`-Batches gesendet.
  - Konflikte: „Last write wins“ per Zeitstempel oder serverseitige Merge-Regeln.
- **Spracheingabe**
  - Sprachumschaltung per Dropdown.
  - UI-Indikator für aktives Diktat; Fehler-Feedback bei fehlender Berechtigung/keine Sprache erkannt.

## API-Skizze (Server)
- `POST /v1/entries/batch` – nimmt Batch (create/update) inkl. lokaler IDs und Zeitstempel entgegen.
- `GET /v1/entries/changes?since=<timestamp>` – liefert Server-Änderungen zur Reconciliation.
- Authentifizierung über kurzlebige Tokens (z. B. OAuth2 Client Credentials oder JWT-Service-Accounts pro Gerät).

## UI-Skizze (Screen-Flow)
1. **Dashboard** – Start/Stop-Button, laufende Session, GPS-Status.
2. **Eintrag bearbeiten** – Projektnummer, Material, Notiz (mit Spracheingabe), Start/Ende anpassen.
3. **Verlauf** – Liste der letzten Einträge mit Sync-Status („ausstehend/gesendet“).
4. **Einstellungen** – Sprachpräferenz, Berechtigungen, Datenschutz, App-Sperre.

## Qualität & Tests
- **Unit/Integration:** Redux-Slices/Services, Offline-DB-Adapter, Sync-Queue.
- **E2E:** Detox-Tests für Start/Stop, Spracheingabe-Fallbacks, Offline-Workflows (Flugmodus).
- **Field Tests:** GPS-Genauigkeit, Akkuverbrauch, Verhalten bei intermittierendem Netz.

## Erweiterungen (optional)
- Fotos/Anhänge pro Eintrag (Materialbelege).
- Geofencing für projektspezifische Orte.
- Export als PDF/CSV bei vorhandener Verbindung.
