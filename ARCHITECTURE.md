# WakaTime Extension Architecture

> **[Deutsche Version unten](#deutsche-version)**

## English Version

### Overview

The WakaTime VSCode extension is a time tracking and productivity analytics tool that automatically monitors your coding activity. It uses an event-driven architecture to capture your programming behavior while filtering out automated and bot activity.

---

## 1. How the Extension Works

### Architecture Overview

The extension follows a **modular, event-driven architecture**:

```
VSCode Events → Event Listeners → Debouncer → Time Filters → 
Heartbeat Queue → Batch Processor → WakaTime CLI → API
```

### Core Components

1. **extension.ts**: Main entry point that initializes the extension
2. **wakatime.ts**: Core logic for tracking and heartbeat management
3. **dependencies.ts**: Manages the wakatime-cli binary
4. **utils.ts**: Helper functions for time calculations and filtering
5. **logger.ts**: Debug logging system
6. **options.ts**: Configuration management

### Event Listeners

The extension monitors these VSCode events:

- **Text Selection Changes**: When you move your cursor
- **Document Changes**: When you type or modify code
- **File Saves**: When you save a file
- **Active Editor Changes**: When you switch between files/tabs
- **Debug Sessions**: When you start/stop debugging
- **Task Execution**: When you run build tasks
- **Notebook Changes**: For Jupyter notebooks

Each event triggers the `onEvent()` handler, which:
1. Debounces rapid-fire events (50ms delay)
2. Checks if enough time has passed since last heartbeat (120 seconds)
3. Creates a heartbeat object with current activity data
4. Queues the heartbeat for sending

---

## 2. How Time Tracking Works

### Heartbeat System

The extension uses a **heartbeat-based tracking system**:

**What is a Heartbeat?**
- A data packet capturing a single moment of coding activity
- Sent periodically to the WakaTime servers
- Contains metadata about what you're working on

**Heartbeat Data Structure:**
```typescript
{
  entity: "/path/to/file.ts",           // File path
  time: 1642342800.0,                   // UNIX timestamp
  is_write: true,                       // Write operation?
  lineno: 42,                           // Cursor line number
  cursorpos: 15,                        // Cursor column position
  lines_in_file: 300,                   // Total lines in file
  category: "coding",                   // Activity type
  ai_line_changes: 0,                   // AI-generated lines
  human_line_changes: 5,                // Human-written lines
  alternate_project: "MyWorkspace",     // Workspace name
  project_folder: "/path/to/project",   // Project root
  is_unsaved_entity: false              // Unsaved file?
}
```

### Time Threshold Logic

**Minimum Time Between Heartbeats: 120 seconds (2 minutes)**

The extension enforces a time threshold to:
- Prevent excessive API calls
- Accurately represent active coding time
- Filter out brief context switches

**How it Works:**
```typescript
function enoughTimePassed(lastHeartbeat: number): boolean {
  const currentTime = Date.now() / 1000;  // Current UNIX time
  return currentTime - lastHeartbeat >= 120;  // 2 minute threshold
}
```

### Activity Categories

The extension automatically categorizes your activity:

1. **Coding** (default): Normal programming
2. **Debugging**: When debugger is active
3. **Building**: When running build/compile tasks
4. **Code Reviewing**: When viewing pull request files
5. **AI Coding**: When AI assistants generate code

### Batch Processing

**Send Buffer System:**
- Heartbeats are queued in memory
- Sent in batches every **30 seconds**
- Multiple heartbeats can be sent in a single API call
- On extension shutdown, all pending heartbeats are flushed

---

## 3. How Bot Activity Detection Works

The extension employs multiple layers of filtering to exclude automated activity:

### Layer 1: Duplicate Heartbeat Detection

**Purpose**: Prevent sending identical events from automated scripts

**Method**: `isDuplicateHeartbeat()`
- Compares current heartbeat with last sent heartbeat
- Checks: same file + same cursor position + minimal time elapsed
- **Duplicate Window**: 30 minutes
- Only applies to **write events** (not selections)

**Example of Filtered Activity:**
```typescript
// Script that repeatedly saves the same file
for (let i = 0; i < 100; i++) {
  document.save();  // ❌ Only first save tracked
}
```

### Layer 2: AI Code Generation Detection

**Purpose**: Distinguish human-written code from AI-generated code

**AI Paste Detection Logic:**
```typescript
// Tracks recent paste operations
const AI_RECENT_PASTES_TIME_MS = 500;  // 500ms window
const AI_RECENT_PASTES_COUNT = 4;      // 4+ pastes = AI coding

// What qualifies as an "AI paste"?
function isAiPaste(change: DocumentChange): boolean {
  const hasMultipleLineBreaks = change.text.includes('\n\n');
  const isSingleLineBurst = !change.text.includes('\n') && 
                            change.text.length >= 50;
  
  return hasMultipleLineBreaks || isSingleLineBurst;
}
```

**AI Activity Tracking:**
- Monitors paste frequency and patterns
- Detects multi-line or large single-line insertions
- Automatically sets `category: "ai coding"` when threshold met
- Tracks `ai_line_changes` separately from `human_line_changes`
- Resets AI flag on manual edits

**Supported AI Providers:**
- GitHub Copilot
- Claude (Anthropic)
- Codeium
- Continue
- OpenAI
- Sourcegraph Cody
- SuperMaven
- Tabnine

### Layer 3: Command-Based Filtering

**Purpose**: Exclude automated VS Code operations

**Filtered Activities:**
- Command palette operations
- Macro executions
- Background/watch tasks
- Extension-triggered changes

**Implementation:**
```typescript
function isCommandTriggered(event: SelectionChangeEvent): boolean {
  // Check if selection change came from a VS Code command
  // rather than user interaction
  return event.kind === TextEditorSelectionChangeKind.Command;
}
```

### Layer 4: Time-Based Deduplication

**Purpose**: Prevent flooding from rapid events

**Debouncing Strategy:**
- **50ms delay** on all events
- Coalesces rapid-fire keystrokes into single heartbeat
- Reduces API load during active typing

---

## 4. API Communication Architecture

### CLI-Based Design

The extension doesn't communicate directly with WakaTime servers. Instead:

**VSCode Extension → WakaTime CLI → WakaTime API**

**Why Use CLI?**
- **Security**: API key never exposed to JavaScript context
- **Offline Support**: CLI buffers heartbeats when offline
- **Cross-Platform**: Single CLI binary works everywhere
- **Shared Logic**: Same CLI used by all WakaTime editor plugins

### CLI Execution Flow

**1. Binary Management** (dependencies.ts):
```typescript
// Auto-download CLI from GitHub releases
async function installCli(): Promise<void> {
  const release = await fetchLatestRelease();
  const binary = await downloadBinary(release);
  await extractAndInstall(binary);
}

// Check for updates every 4 hours
setInterval(checkForUpdates, 4 * 60 * 60 * 1000);
```

**2. Heartbeat Sending** (wakatime.ts):
```typescript
// Execute CLI with heartbeat data
execFile(
  cliPath,                    // Path to wakatime-cli
  [
    '--entity', filePath,
    '--time', timestamp,
    '--lineno', lineNumber,
    '--cursorpos', columnNumber,
    '--lines-in-file', totalLines,
    '--category', category,
    '--key', apiKey,
    '--plugin', 'vscode/1.89.0',
    '--extra-heartbeats'      // Batch mode flag
  ],
  { stdin: batchHeartbeats }  // Additional heartbeats via stdin
);
```

**3. Response Handling**:

Exit codes from CLI:
- `0`: Success
- `102` / `112`: Offline mode (no internet)
- `103`: Configuration error
- `104`: Invalid API key
- Other: Unknown error

### Batch API Communication

**Optimization Strategy:**

Instead of sending one heartbeat per API call:
```
Send 1 heartbeat → 1 API call
Send 10 heartbeats → 1 API call (batch)
```

**Implementation:**
```typescript
// Primary heartbeat via command-line args
const primaryHeartbeat = createHeartbeatArgs(currentActivity);

// Additional heartbeats via stdin (JSON array)
const extraHeartbeats = bufferedHeartbeats.map(h => ({
  entity: h.file,
  time: h.timestamp,
  // ... other fields
}));

// Single CLI call with all heartbeats
execFile(cli, primaryHeartbeat, {
  stdin: JSON.stringify(extraHeartbeats)
});
```

### API Endpoints

The CLI communicates with these endpoints:

1. **POST /api/v1/users/current/heartbeats**
   - Sends activity data
   - Batch upload supported

2. **GET /api/v1/users/current/summaries**
   - Fetches daily coding statistics
   - Used for status bar display

3. **GET /api/v1/users/current/file_experts**
   - Retrieves per-file developer expertise
   - Team feature only

### Configuration Management

**API Settings:**
- **API Key**: Stored in `~/.wakatime.cfg` or VS Code settings
- **API URL**: Defaults to `https://api.wakatime.com/api/v1`
- **Proxy**: Supports HTTP/HTTPS proxies
- **Timeout**: 60 seconds per request
- **Retry Logic**: Automatic retries on network failures

**Environment Variables:**
- `WAKATIME_API_KEY`: API key for cloud IDEs (Gitpod, etc.)
- `HTTPS_PROXY` / `HTTP_PROXY`: Proxy configuration

---

## 5. Offline Support

The CLI automatically handles offline scenarios:

1. **Detection**: CLI detects network failures (exit code 102/112)
2. **Buffering**: Heartbeats stored in `~/.wakatime/wakatime-internal.cfg`
3. **Retry**: On reconnection, buffered heartbeats are sent
4. **Backoff**: Exponential backoff prevents excessive retries

---

## 6. Privacy & Security

### Data Minimization

The extension only tracks:
- ✅ File paths (can be obfuscated)
- ✅ Programming language
- ✅ Project names
- ✅ Line counts
- ✅ Timestamps

Does NOT track:
- ❌ File contents
- ❌ Variable names
- ❌ Code logic
- ❌ Keyboard input

### Security Measures

1. **API Key Protection**: Never logged or exposed
2. **HTTPS Only**: All API communication encrypted
3. **Local Processing**: All filtering happens client-side
4. **No Third-Party**: Direct communication with WakaTime only

---

## 7. Performance Optimizations

### Memory Efficiency

- **Bounded Buffers**: Maximum 10 heartbeats buffered
- **Lazy Loading**: CLI downloaded only when needed
- **Minimal Footprint**: ~1-2 MB memory usage

### CPU Efficiency

- **Debouncing**: Reduces event processing by 90%+
- **Time Thresholds**: Prevents unnecessary API calls
- **Asynchronous**: Non-blocking CLI execution

### Network Efficiency

- **Batch Sending**: 30-second intervals
- **Compression**: Gzip compression on API calls
- **Conditional Updates**: Only download new CLI versions

---

## 8. Troubleshooting Bot Detection

### If Legitimate Activity is Filtered

1. **Check Duplicate Window**: Activity might be too frequent
2. **Review AI Detection**: Large pastes might trigger AI flag
3. **Enable Debug Mode**: `Cmd+Shift+P` → "WakaTime: Debug"
4. **Check Logs**: `~/.wakatime/wakatime.log`

### If Bot Activity is Not Filtered

1. **Report Issue**: Open issue on GitHub with anonymized logs
2. **Custom Filters**: Add exclude patterns in `~/.wakatime.cfg`

---

## 9. Key Algorithms

### Time Gap Calculation

```typescript
/**
 * Determines if enough time has passed to send next heartbeat
 * @param lastTime - Timestamp of last heartbeat (seconds)
 * @returns true if >= 120 seconds elapsed
 */
function enoughTimePassed(lastTime: number): boolean {
  const now = Date.now() / 1000;
  const threshold = 120;  // 2 minutes
  return (now - lastTime) >= threshold;
}
```

### Duplicate Detection

```typescript
/**
 * Checks if current heartbeat is duplicate of last
 * @param current - Current heartbeat
 * @param previous - Previous heartbeat
 * @returns true if same file + position within 30 min
 */
function isDuplicateHeartbeat(
  current: Heartbeat, 
  previous: Heartbeat
): boolean {
  const DUPLICATE_WINDOW = 30 * 60;  // 30 minutes
  
  return (
    current.entity === previous.entity &&
    current.lineno === previous.lineno &&
    current.cursorpos === previous.cursorpos &&
    current.time - previous.time < DUPLICATE_WINDOW
  );
}
```

### AI Paste Detection

```typescript
/**
 * Detects if recent activity is AI-generated code
 * @param recentPastes - Array of recent paste events
 * @returns true if 4+ AI pastes in 500ms window
 */
function isAiCodingActivity(recentPastes: Paste[]): boolean {
  const TIME_WINDOW = 500;  // milliseconds
  const PASTE_THRESHOLD = 4;
  
  const now = Date.now();
  const recentAiPastes = recentPastes.filter(paste => 
    (now - paste.timestamp) < TIME_WINDOW &&
    isAiPaste(paste.text)
  );
  
  return recentAiPastes.length >= PASTE_THRESHOLD;
}

function isAiPaste(text: string): boolean {
  // Multi-line with blank lines (AI formatting)
  if (text.includes('\n\n')) return true;
  
  // Single long line (AI completion)
  if (!text.includes('\n') && text.length >= 50) return true;
  
  return false;
}
```

---

# Deutsche Version

## Überblick

Die WakaTime VSCode-Erweiterung ist ein Zeiterfassungs- und Produktivitätsanalyse-Tool, das Ihre Programmieraktivität automatisch überwacht. Es verwendet eine ereignisgesteuerte Architektur, um Ihr Programmierverhalten zu erfassen und gleichzeitig automatisierte und Bot-Aktivitäten herauszufiltern.

---

## 1. Wie die Erweiterung funktioniert

### Architektur-Überblick

Die Erweiterung folgt einer **modularen, ereignisgesteuerten Architektur**:

```
VSCode-Ereignisse → Event-Listener → Debouncer → Zeitfilter → 
Heartbeat-Warteschlange → Batch-Prozessor → WakaTime CLI → API
```

### Kernkomponenten

1. **extension.ts**: Haupteinstiegspunkt, der die Erweiterung initialisiert
2. **wakatime.ts**: Kernlogik für Tracking und Heartbeat-Verwaltung
3. **dependencies.ts**: Verwaltet die wakatime-cli-Binärdatei
4. **utils.ts**: Hilfsfunktionen für Zeitberechnungen und Filterung
5. **logger.ts**: Debug-Logging-System
6. **options.ts**: Konfigurationsverwaltung

### Event-Listener

Die Erweiterung überwacht diese VSCode-Ereignisse:

- **Textauswahländerungen**: Wenn Sie den Cursor bewegen
- **Dokumentänderungen**: Wenn Sie tippen oder Code ändern
- **Dateispeicherungen**: Wenn Sie eine Datei speichern
- **Aktive Editor-Wechsel**: Wenn Sie zwischen Dateien/Tabs wechseln
- **Debug-Sitzungen**: Wenn Sie das Debuggen starten/stoppen
- **Task-Ausführung**: Wenn Sie Build-Tasks ausführen
- **Notebook-Änderungen**: Für Jupyter-Notebooks

Jedes Ereignis löst den `onEvent()`-Handler aus, der:
1. Ereignisse mit hoher Frequenz entprellt (50ms Verzögerung)
2. Prüft, ob genug Zeit seit dem letzten Heartbeat vergangen ist (120 Sekunden)
3. Ein Heartbeat-Objekt mit aktuellen Aktivitätsdaten erstellt
4. Den Heartbeat zum Senden einreiht

---

## 2. Wie die Zeiterfassung funktioniert

### Heartbeat-System

Die Erweiterung verwendet ein **Heartbeat-basiertes Tracking-System**:

**Was ist ein Heartbeat?**
- Ein Datenpaket, das einen einzelnen Moment der Programmieraktivität erfasst
- Wird periodisch an die WakaTime-Server gesendet
- Enthält Metadaten darüber, woran Sie arbeiten

**Heartbeat-Datenstruktur:**
```typescript
{
  entity: "/pfad/zur/datei.ts",         // Dateipfad
  time: 1642342800.0,                   // UNIX-Zeitstempel
  is_write: true,                       // Schreiboperation?
  lineno: 42,                           // Cursor-Zeilennummer
  cursorpos: 15,                        // Cursor-Spaltenposition
  lines_in_file: 300,                   // Gesamtzahl der Zeilen
  category: "coding",                   // Aktivitätstyp
  ai_line_changes: 0,                   // KI-generierte Zeilen
  human_line_changes: 5,                // Manuell geschriebene Zeilen
  alternate_project: "MeinWorkspace",   // Workspace-Name
  project_folder: "/pfad/zum/projekt",  // Projekt-Root
  is_unsaved_entity: false              // Ungespeicherte Datei?
}
```

### Zeitschwellen-Logik

**Mindestzeit zwischen Heartbeats: 120 Sekunden (2 Minuten)**

Die Erweiterung erzwingt eine Zeitschwelle, um:
- Übermäßige API-Aufrufe zu verhindern
- Aktive Programmierzeit genau darzustellen
- Kurze Kontextwechsel herauszufiltern

**Wie es funktioniert:**
```typescript
function enoughTimePassed(lastHeartbeat: number): boolean {
  const currentTime = Date.now() / 1000;  // Aktuelle UNIX-Zeit
  return currentTime - lastHeartbeat >= 120;  // 2-Minuten-Schwelle
}
```

### Aktivitätskategorien

Die Erweiterung kategorisiert Ihre Aktivität automatisch:

1. **Coding** (Standard): Normale Programmierung
2. **Debugging**: Wenn der Debugger aktiv ist
3. **Building**: Beim Ausführen von Build/Compile-Tasks
4. **Code Reviewing**: Beim Betrachten von Pull-Request-Dateien
5. **AI Coding**: Wenn KI-Assistenten Code generieren

### Batch-Verarbeitung

**Send-Buffer-System:**
- Heartbeats werden im Speicher in eine Warteschlange gestellt
- In Batches alle **30 Sekunden** gesendet
- Mehrere Heartbeats können in einem einzigen API-Aufruf gesendet werden
- Beim Herunterfahren der Erweiterung werden alle ausstehenden Heartbeats geleert

---

## 3. Wie Bot-Aktivitätserkennung funktioniert

Die Erweiterung verwendet mehrere Filterebenen, um automatisierte Aktivitäten auszuschließen:

### Ebene 1: Doppelte Heartbeat-Erkennung

**Zweck**: Verhindern, dass identische Ereignisse von automatisierten Skripten gesendet werden

**Methode**: `isDuplicateHeartbeat()`
- Vergleicht aktuellen Heartbeat mit zuletzt gesendetem Heartbeat
- Prüft: gleiche Datei + gleiche Cursor-Position + minimale verstrichene Zeit
- **Duplikat-Fenster**: 30 Minuten
- Gilt nur für **Schreibereignisse** (nicht für Auswahlen)

**Beispiel für gefilterte Aktivität:**
```typescript
// Skript, das wiederholt dieselbe Datei speichert
for (let i = 0; i < 100; i++) {
  document.save();  // ❌ Nur erstes Speichern wird getrackt
}
```

### Ebene 2: KI-Code-Generierungserkennung

**Zweck**: Von Menschen geschriebenen Code von KI-generiertem Code unterscheiden

**KI-Einfüge-Erkennungslogik:**
```typescript
// Verfolgt kürzliche Einfügevorgänge
const AI_RECENT_PASTES_TIME_MS = 500;  // 500ms-Fenster
const AI_RECENT_PASTES_COUNT = 4;      // 4+ Einfügevorgänge = KI-Coding

// Was qualifiziert als "KI-Einfügung"?
function isAiPaste(change: DocumentChange): boolean {
  const hasMultipleLineBreaks = change.text.includes('\n\n');
  const isSingleLineBurst = !change.text.includes('\n') && 
                            change.text.length >= 50;
  
  return hasMultipleLineBreaks || isSingleLineBurst;
}
```

**KI-Aktivitätsverfolgung:**
- Überwacht Einfügungshäufigkeit und -muster
- Erkennt mehrzeilige oder große einzeilige Einfügungen
- Setzt automatisch `category: "ai coding"`, wenn Schwellenwert erreicht wird
- Verfolgt `ai_line_changes` getrennt von `human_line_changes`
- Setzt KI-Flag bei manuellen Änderungen zurück

**Unterstützte KI-Anbieter:**
- GitHub Copilot
- Claude (Anthropic)
- Codeium
- Continue
- OpenAI
- Sourcegraph Cody
- SuperMaven
- Tabnine

### Ebene 3: Befehlsbasierte Filterung

**Zweck**: Automatisierte VS Code-Operationen ausschließen

**Gefilterte Aktivitäten:**
- Command-Palette-Operationen
- Makro-Ausführungen
- Hintergrund-/Watch-Tasks
- Durch Erweiterungen ausgelöste Änderungen

**Implementierung:**
```typescript
function isCommandTriggered(event: SelectionChangeEvent): boolean {
  // Prüfen, ob Auswahländerung von einem VS Code-Befehl kam
  // statt von Benutzerinteraktion
  return event.kind === TextEditorSelectionChangeKind.Command;
}
```

### Ebene 4: Zeitbasierte Deduplikation

**Zweck**: Überflutung durch schnelle Ereignisse verhindern

**Debouncing-Strategie:**
- **50ms Verzögerung** bei allen Ereignissen
- Fasst schnelle Tastenanschläge zu einem einzigen Heartbeat zusammen
- Reduziert API-Last während aktiver Eingabe

---

## 4. API-Kommunikationsarchitektur

### CLI-basiertes Design

Die Erweiterung kommuniziert nicht direkt mit WakaTime-Servern. Stattdessen:

**VSCode-Erweiterung → WakaTime CLI → WakaTime API**

**Warum CLI verwenden?**
- **Sicherheit**: API-Key wird nie im JavaScript-Kontext offengelegt
- **Offline-Unterstützung**: CLI puffert Heartbeats, wenn offline
- **Plattformübergreifend**: Einzelne CLI-Binärdatei funktioniert überall
- **Geteilte Logik**: Dieselbe CLI wird von allen WakaTime-Editor-Plugins verwendet

### CLI-Ausführungsablauf

**1. Binärverwaltung** (dependencies.ts):
```typescript
// CLI automatisch von GitHub-Releases herunterladen
async function installCli(): Promise<void> {
  const release = await fetchLatestRelease();
  const binary = await downloadBinary(release);
  await extractAndInstall(binary);
}

// Alle 4 Stunden nach Updates suchen
setInterval(checkForUpdates, 4 * 60 * 60 * 1000);
```

**2. Heartbeat-Versand** (wakatime.ts):
```typescript
// CLI mit Heartbeat-Daten ausführen
execFile(
  cliPath,                    // Pfad zu wakatime-cli
  [
    '--entity', filePath,
    '--time', timestamp,
    '--lineno', lineNumber,
    '--cursorpos', columnNumber,
    '--lines-in-file', totalLines,
    '--category', category,
    '--key', apiKey,
    '--plugin', 'vscode/1.89.0',
    '--extra-heartbeats'      // Batch-Modus-Flag
  ],
  { stdin: batchHeartbeats }  // Zusätzliche Heartbeats über stdin
);
```

**3. Antwortbehandlung**:

Exit-Codes von der CLI:
- `0`: Erfolg
- `102` / `112`: Offline-Modus (kein Internet)
- `103`: Konfigurationsfehler
- `104`: Ungültiger API-Key
- Andere: Unbekannter Fehler

### Batch-API-Kommunikation

**Optimierungsstrategie:**

Statt einen Heartbeat pro API-Aufruf zu senden:
```
1 Heartbeat senden → 1 API-Aufruf
10 Heartbeats senden → 1 API-Aufruf (Batch)
```

**Implementierung:**
```typescript
// Primärer Heartbeat über Kommandozeilenargumente
const primaryHeartbeat = createHeartbeatArgs(currentActivity);

// Zusätzliche Heartbeats über stdin (JSON-Array)
const extraHeartbeats = bufferedHeartbeats.map(h => ({
  entity: h.file,
  time: h.timestamp,
  // ... andere Felder
}));

// Einzelner CLI-Aufruf mit allen Heartbeats
execFile(cli, primaryHeartbeat, {
  stdin: JSON.stringify(extraHeartbeats)
});
```

### API-Endpunkte

Die CLI kommuniziert mit diesen Endpunkten:

1. **POST /api/v1/users/current/heartbeats**
   - Sendet Aktivitätsdaten
   - Batch-Upload unterstützt

2. **GET /api/v1/users/current/summaries**
   - Ruft tägliche Programmierstatistiken ab
   - Wird für Statusleisten-Anzeige verwendet

3. **GET /api/v1/users/current/file_experts**
   - Ruft Entwickler-Expertise pro Datei ab
   - Nur Team-Funktion

### Konfigurationsverwaltung

**API-Einstellungen:**
- **API-Key**: Gespeichert in `~/.wakatime.cfg` oder VS Code-Einstellungen
- **API-URL**: Standard ist `https://api.wakatime.com/api/v1`
- **Proxy**: Unterstützt HTTP/HTTPS-Proxys
- **Timeout**: 60 Sekunden pro Anfrage
- **Retry-Logik**: Automatische Wiederholungsversuche bei Netzwerkfehlern

**Umgebungsvariablen:**
- `WAKATIME_API_KEY`: API-Key für Cloud-IDEs (Gitpod, etc.)
- `HTTPS_PROXY` / `HTTP_PROXY`: Proxy-Konfiguration

---

## 5. Offline-Unterstützung

Die CLI behandelt Offline-Szenarien automatisch:

1. **Erkennung**: CLI erkennt Netzwerkausfälle (Exit-Code 102/112)
2. **Pufferung**: Heartbeats werden in `~/.wakatime/wakatime-internal.cfg` gespeichert
3. **Wiederholung**: Bei Wiederverbindung werden gepufferte Heartbeats gesendet
4. **Backoff**: Exponentieller Backoff verhindert übermäßige Wiederholungsversuche

---

## 6. Datenschutz & Sicherheit

### Datenminimierung

Die Erweiterung verfolgt nur:
- ✅ Dateipfade (können verschleiert werden)
- ✅ Programmiersprache
- ✅ Projektnamen
- ✅ Zeilenanzahlen
- ✅ Zeitstempel

Verfolgt NICHT:
- ❌ Dateiinhalte
- ❌ Variablennamen
- ❌ Code-Logik
- ❌ Tastatureingaben

### Sicherheitsmaßnahmen

1. **API-Key-Schutz**: Wird niemals protokolliert oder offengelegt
2. **Nur HTTPS**: Alle API-Kommunikation verschlüsselt
3. **Lokale Verarbeitung**: Alle Filterung erfolgt clientseitig
4. **Keine Drittanbieter**: Direkte Kommunikation nur mit WakaTime

---

## 7. Leistungsoptimierungen

### Speichereffizienz

- **Begrenzte Puffer**: Maximum 10 Heartbeats gepuffert
- **Lazy Loading**: CLI wird nur bei Bedarf heruntergeladen
- **Minimaler Footprint**: ~1-2 MB Speichernutzung

### CPU-Effizienz

- **Debouncing**: Reduziert Ereignisverarbeitung um 90%+
- **Zeitschwellen**: Verhindert unnötige API-Aufrufe
- **Asynchron**: Nicht-blockierende CLI-Ausführung

### Netzwerkeffizienz

- **Batch-Versand**: 30-Sekunden-Intervalle
- **Kompression**: Gzip-Kompression bei API-Aufrufen
- **Bedingte Updates**: Lädt nur neue CLI-Versionen herunter

---

## 8. Fehlerbehebung bei Bot-Erkennung

### Wenn legitime Aktivität gefiltert wird

1. **Duplikat-Fenster prüfen**: Aktivität könnte zu häufig sein
2. **KI-Erkennung überprüfen**: Große Einfügungen könnten KI-Flag auslösen
3. **Debug-Modus aktivieren**: `Cmd+Shift+P` → "WakaTime: Debug"
4. **Logs überprüfen**: `~/.wakatime/wakatime.log`

### Wenn Bot-Aktivität nicht gefiltert wird

1. **Problem melden**: Issue auf GitHub mit anonymisierten Logs öffnen
2. **Benutzerdefinierte Filter**: Ausschlussmuster in `~/.wakatime.cfg` hinzufügen

---

## 9. Schlüssel-Algorithmen

### Zeitlücken-Berechnung

```typescript
/**
 * Bestimmt, ob genug Zeit vergangen ist, um nächsten Heartbeat zu senden
 * @param lastTime - Zeitstempel des letzten Heartbeats (Sekunden)
 * @returns true wenn >= 120 Sekunden vergangen
 */
function enoughTimePassed(lastTime: number): boolean {
  const now = Date.now() / 1000;
  const threshold = 120;  // 2 Minuten
  return (now - lastTime) >= threshold;
}
```

### Duplikatserkennung

```typescript
/**
 * Prüft, ob aktueller Heartbeat ein Duplikat des letzten ist
 * @param current - Aktueller Heartbeat
 * @param previous - Vorheriger Heartbeat
 * @returns true wenn gleiche Datei + Position innerhalb von 30 Min
 */
function isDuplicateHeartbeat(
  current: Heartbeat, 
  previous: Heartbeat
): boolean {
  const DUPLICATE_WINDOW = 30 * 60;  // 30 Minuten
  
  return (
    current.entity === previous.entity &&
    current.lineno === previous.lineno &&
    current.cursorpos === previous.cursorpos &&
    current.time - previous.time < DUPLICATE_WINDOW
  );
}
```

### KI-Einfüge-Erkennung

```typescript
/**
 * Erkennt, ob kürzliche Aktivität KI-generierter Code ist
 * @param recentPastes - Array kürzlicher Einfüge-Ereignisse
 * @returns true wenn 4+ KI-Einfügungen in 500ms-Fenster
 */
function isAiCodingActivity(recentPastes: Paste[]): boolean {
  const TIME_WINDOW = 500;  // Millisekunden
  const PASTE_THRESHOLD = 4;
  
  const now = Date.now();
  const recentAiPastes = recentPastes.filter(paste => 
    (now - paste.timestamp) < TIME_WINDOW &&
    isAiPaste(paste.text)
  );
  
  return recentAiPastes.length >= PASTE_THRESHOLD;
}

function isAiPaste(text: string): boolean {
  // Mehrzeilig mit Leerzeilen (KI-Formatierung)
  if (text.includes('\n\n')) return true;
  
  // Einzelne lange Zeile (KI-Vervollständigung)
  if (!text.includes('\n') && text.length >= 50) return true;
  
  return false;
}
```

---

## Zusammenfassung

Die WakaTime VSCode-Erweiterung ist ein ausgeklügeltes Zeiterfassungssystem, das:

1. **Ereignisgesteuert arbeitet**: Erfasst alle relevanten VSCode-Aktivitäten
2. **Intelligente Filterung verwendet**: Unterscheidet menschliche von automatisierter/KI-Aktivität
3. **Effizient kommuniziert**: Batch-Verarbeitung und CLI-basierte API-Aufrufe
4. **Datenschutz respektiert**: Erfasst nur Metadaten, keine Code-Inhalte
5. **Offline funktioniert**: Puffert Daten und synchronisiert bei Wiederverbindung

Die mehrstufige Bot-Erkennung stellt sicher, dass nur echte menschliche Programmieraktivität verfolgt wird, während Skripte, Makros und KI-generierter Code entsprechend gekennzeichnet oder gefiltert werden.
