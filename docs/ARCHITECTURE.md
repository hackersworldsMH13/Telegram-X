# TELEGRAM MEDIA OS — UNIFIED ARCHITECTURE BLUEPRINT

*Merging the visionary "Media OS" with the production‑ready "Archiver Pro" foundation*

---

## 1. Vision & Philosophy

**Telegram Media OS** is the ultimate Android application that behaves as:

- A **reliable offline Telegram mirror**  
- A **smart, AI‑powered media archive**  
- A **local media server / NAS**  
- A **streaming hub** (watch before full download)  
- A **fully automated sync & backup engine**

We achieve this not by rewriting everything from scratch but by **forking the official Telegram‑FOSS Android client** and extending it with modular engines. The forked base gives us:

- Battle‑tested TDLib integration (JNI + Java/Kotlin wrapper)  
- Stable login / session persistence / 2FA  
- Efficient caching and file management  
- High‑performance UI components (RecyclerView, chat list, photo viewer)  
- Background connection handling

On top of this foundation we build **15 specialised engines** that transform it into a Media OS.

---

## 2. Technology Stack & Module Map

```
┌──────────────────────────────────────────────────┐
│                :app (Application)                 │
│   DI (Hilt), Navigation, Startup, Permissions    │
└────┬─────────────────────────────────────────────┘
     │
     ├── :telegram-base  (forked Telegram-FOSS, stripped)
     │    ├── TDLib JNI + native libs
     │    ├── ConnectionsManager, NotificationCenter
     │    ├── Utilities, FileLoader, ImageLoader
     │    └── Minimal UI: LoginActivity, ChatListFragment
     │
     ├── :archiver-core   (Domain layer)
     │    ├── Use Cases (interactors)
     │    ├── Repository interfaces
     │    ├── Domain models & enums
     │    └── Extension functions
     │
     ├── :archiver-data   (Data layer)
     │    ├── TDLibWrapper (Kotlin coroutine API)
     │    ├── Room database + DAOs
     │    ├── FileStorageManager
     │    ├── MetadataIndexer
     │    └── CacheReader (reuse Telegram cache)
     │
     ├── :archiver-ui     (Compose screens)
     │    ├── ViewModels
     │    ├── Screens (Dashboard, Downloader, Browser…)
     │    └── Reusable Compose components
     │
     ├── :archiver-sync   (Background work)
     │    ├── Workers (PeriodicSync, DriveUpload, Forward)
     │    ├── ForegroundDownloadService
     │    └── SyncScheduler
     │
     ├── :archiver-media  (Processing engines)
     │    ├── FFmpegProcessor
     │    ├── ThumbnailGenerator
     │    ├── APKAnalyzer
     │    └── MediaCategorizer
     │
     ├── :archiver-ai     (On‑device intelligence)
     │    ├── ImageTagger (TFLite)
     │    ├── OCRProcessor
     │    ├── DuplicateDetector (pHashing)
     │    └── NSFWClassifier (ML Kit)
     │
     ├── :archiver-streaming (ExoPlayer integration)
     │    ├── PartialFileDataSource
     │    └── StreamingViewModel
     │
     └── :archiver-nas    (Local server)
          ├── NanoHTTPD wrapper
          └── DLNA / Chromecast bridge
```

**Key data pipes:**  
- TDLib events → `SharedFlow` → Domain Use Cases → `StateFlow` → Compose UI.  
- Room `Flow<List<T>>` → ViewModel → UI.  
- Download progress: `ForegroundService` broadcasts → `Flow` updates.

---

## 3. Architecture Layers – Deep Dive

### 3.1 UI Layer (`:archiver-ui`)

**Technology:** Jetpack Compose + Material 3 + existing Telegram Views (reused for login & chat list).  
**Screens:**  
- Dashboard (active downloads, storage stats, quick actions)  
- Chat Picker (multi‑select, filters, limit)  
- Download Manager (queue, progress, retry)  
- Media Browser (grid with search, filter, preview)  
- Video Player (streaming & local)  
- AI Search (full‑text with smart filters)  
- Analytics Dashboard  
- Settings (path, sync schedule, cloud accounts, encryption)  

**Data Flow:**  
```text
ViewModel (holds StateFlow<UiState>)
   ↑ collects
Repository / UseCase (returns Flow)
   ↑
Data layer (Room DAO, TDLib wrapper)
```

### 3.2 Domain Layer (`:archiver-core`)

Contains **pure business logic** – no Android dependencies.

**Key Interfaces (repositories):**
```kotlin
interface ChatRepository {
    suspend fun getDialogs(filter: ChatTypeFilter): List<ArchiveChat>
    suspend fun updateLastSyncMessageId(chatId: Long, msgId: Long)
    fun observeChats(): Flow<List<ArchiveChat>>
}

interface MediaRepository {
    suspend fun recordDownload(media: MediaItem): Long
    suspend fun isDuplicate(chatId: Long, msgId: Long): Boolean
    suspend fun isDuplicateByHash(hash: String): Boolean
    fun search(query: String, filters: MediaFilter): Flow<List<MediaItem>>
}

interface DownloadQueueRepository {
    suspend fun enqueue(task: DownloadTask)
    suspend fun dequeue(limit: Int): List<DownloadTask>
    suspend fun markCompleted(taskId: String)
    suspend fun markFailed(taskId: String, error: String)
    fun observeQueue(): Flow<List<DownloadTask>>
}

interface SyncHistoryRepository { … }
interface AiTagRepository { … }
```

**Use Cases (Interactors):**  
- `LoginUseCase` – delegates to TDLib wrapper  
- `FetchChatsUseCase` – paging, search  
- `StartDownloadUseCase` – builds tasks from chat/message range  
- `ResumeDownloadUseCase` – reads last synced ID  
- `ProcessDownloadUseCase` – runs download + post‑processing pipeline  
- `SyncChannelsUseCase` – orchestrates periodic scans  
- `SearchMediaUseCase` – FTS + filters  
- `GetStatisticsUseCase` – aggregations  
- `StreamMediaUseCase` – prepares ExoPlayer source  

### 3.3 Data Layer (`:archiver-data`)

#### 3.3.1 TDLib Wrapper (Kotlin coroutines)
We **do not** call TDLib's JNI directly. Instead we build a thin layer over Telegram's `ConnectionsManager`:

```kotlin
class TdLibClient @Inject constructor(
    private val connectionsManager: ConnectionsManager
) {
    // Convert callback-style to suspend functions
    suspend fun getChatHistory(chatId: Long, fromMessageId: Long, limit: Int): List<TdApi.Message> =
        suspendCoroutine { cont ->
            connectionsManager.sendRequest(
                TdApi.GetChatHistory(chatId, fromMessageId, 0, limit, false),
                { response -> cont.resume((response as TdApi.Messages).messages.asList()) },
                { error -> cont.resumeWithException(Exception(error.message)) }
            )
        }

    // Or use flow to emit progress
    fun downloadFile(fileId: Int, priority: Int = 1): Flow<DownloadStatus> = callbackFlow {
        connectionsManager.sendRequest(
            TdApi.DownloadFile(fileId, priority, 0, 0, true),
            { /* onProgress */ },
            { /* onComplete/Error */ }
        )
        awaitClose { /* cancel if needed */ }
    }
}
```

**Data pipe:** `updateFile` events from TDLib are observed via `NotificationCenter` and published to a `SharedFlow` that our wrapper transforms into `DownloadStatus` objects.

#### 3.3.2 Room Database (Schemas)
We fully define every table here, with indices and FTS.

**Table: `archive_chats`**
| Column               | Type    | Description                     |
|----------------------|---------|---------------------------------|
| chat_id              | Long PK | Telegram chat ID                |
| title                | String  |                                 |
| username             | String? |                                 |
| chat_type            | Int     | Enum: 0=private,1=group,…       |
| last_synced_msg_id   | Long    | Resume point (default 0)        |
| is_sync_enabled      | Boolean | For scheduled sync              |
| private_encrypted    | Boolean | Whether files are encrypted     |

**Table: `media_items`**
| Column               | Type    | Notes                           |
|----------------------|---------|---------------------------------|
| id                   | Long PK | auto‑generated                  |
| message_id           | Long    |                                 |
| chat_id              | Long FK | → archive_chats.chat_id         |
| media_type           | Int     | Enum: PHOTO, VIDEO, AUDIO, …    |
| original_file_name   | String? |                                 |
| local_path           | String  | Full path on device             |
| size_bytes           | Long    |                                 |
| md5_hash             | String? | For duplicate check             |
| phash                | String? | Perceptual hash (for images)    |
| ocr_text             | String? | Extracted text                  |
| download_timestamp   | Long    | Epoch millis                    |
| is_private           | Boolean |                                 |

**Index:** `UNIQUE (chat_id, message_id)`  
**FTS table:** `media_items_fts`  
- Content: `original_file_name`, `ocr_text`, `caption` (from TDLib, stored separately if needed), `ai_tags`  
- Trigger to keep FTS updated automatically.

**Table: `download_queue`**
| Column        | Type    |
|---------------|---------|
| task_id       | String PK (chat_id + message_id) |
| chat_id       | Long    |
| message_id    | Long    |
| file_id       | Int     | TDLib file ID                    |
| media_type    | Int     |
| target_path   | String  |
| priority      | Int     |
| retry_count   | Int     |
| status        | Int     | 0=PENDING,1=DOWNLOADING,2=DONE,3=FAILED |

**Table: `ai_tags`**
| Column    | Type    |
|-----------|---------|
| media_id  | Long FK |
| tag       | String  |
| confidence| Float   |

**Table: `sync_history`**  
… etc.

#### 3.3.3 File Storage Manager
- Uses **Storage Access Framework (SAF)** to let user choose base directory (works with internal, SD card, USB OTG).  
- `FileStorageManager` provides methods: `buildLocalPath(chat, message, ext)`, `moveFromCache(tempFile, finalPath)`, `ensureFolder()`.  
- Folder structure:
```
BaseDir/
 ├── Channels/<ChannelName>/
 │    ├── Photos/
 │    ├── Videos/
 │    ├── Audio/
 │    ├── Documents/
 │    ├── APKs/
 │    ├── Archives/
 │    └── Metadata/
 ├── PrivateChats/<ContactName>/
 │    └── … (same subfolders, optionally encrypted)
 ├── _Temp/            (in‑progress downloads)
 └── _CacheIndex/      (symlinks or database records for Telegram cache reuse)
```

#### 3.3.4 Metadata Indexer
- After file is saved, extracts EXIF, video codec info, duration, etc., using `MediaMetadataRetriever` or FFmpeg.  
- Stores structured data in `media_items` columns or separate JSON.

#### 3.3.5 Cache Reuse Engine
- Telegram already caches files in its internal directory (`/data/data/org.telegram.messenger/cache`).  
- Our `CacheReader` scans that directory using TDLib's `getFile` and `file.local.path`.  
- If a file is already cached and not expired, we **copy** (or hardlink if on same filesystem) to our organized folder instead of re‑downloading.  
- This dramatically saves bandwidth.

### 3.4 Native Layer (`:archiver-media`, `:archiver-ai`)

- **FFmpeg Kit**: executed via `FFmpegKit.executeAsync()` with progress callback.  
- **Thumbnail Generator**: uses `MediaMetadataRetriever` for local files and `FFmpeg` for streaming/partial.  
- **AI Models**: TensorFlow Lite for image classification; ML Kit for OCR and face detection (optional).  
- **Perceptual Hashing**: custom JNI or Java library (e.g., `pHash`) for image duplicate detection.  
- **APK Analyzer**: uses `PackageManager` and `ApkParser` library to extract manifest, permissions, icon.

---

## 4. Core Engine: Download & Sync Pipeline

This is the heart of the system. It must be **resilient**, **thread‑safe**, and **observable**.

### 4.1 Flowchart: End‑to‑End Message Processing

```
User selects chats + media types + limit
        │
        ▼
StartDownloadUseCase
  └─ for each chat:
       load last_synced_msg_id (resume point)
       fetch batch from TDLib (getChatHistory)
        │
        ▼
For each message:
   ├─ Check duplicate (chat_id + message_id in Room) → skip
   ├─ Does message contain selected media type? → no → update last_synced_msg_id (skip)
   └─ [media found] → enqueue DownloadTask
        │
        ▼
DownloadQueue (Room table) – priority FIFO
        │
        ▼
DownloadWorker (ForegroundService)
   ├─ Dequeue next task
   ├─ TDLib downloadFile(fileId)
   │   ├─ progress → update notification
   │   └─ completion → file.local.path (in Telegram cache)
   ├─ Copy from cache → _Temp/target
   ├─ Optional: FFmpeg conversion (if needed for compatibility)
   ├─ Generate thumbnail
   ├─ Calculate MD5 & pHash (in background)
   ├─ Move to final organized path
   ├─ Insert media_item record (or update cache if reused)
   ├─ Insert/update ai_tags (if AI enabled)
   ├─ Update last_synced_msg_id for chat
   └─ Mark task completed
        │
        ▼
Optional post‑processing:
   ├─ Auto‑forward (sendCopy)
   └─ Upload to cloud (Google Drive worker)
```

### 4.2 Resume Logic (Detailed)
- For each chat, we store `last_synced_msg_id` (initially 0).  
- When fetching history: `getChatHistory(chatId, fromMessageId = last_synced_msg_id, offset = 0, limit = LIMIT)`.  
- **Crucial:** TDLib returns messages **in descending order** (newest first). If we process from oldest to newest (recommended), we must reverse the list.  
- After processing a message, we update `last_synced_msg_id` to that message's ID **only if it is greater than the previous** (i.e., we are moving forward in time).  
- This ensures that if the app is killed mid‑batch, next launch will resume from the last successfully recorded message.

**Data flow for resume:**
```
Room (archive_chats) → last_synced_msg_id → TDLib getChatHistory(fromMessageId) → process → update last_synced_msg_id
```

### 4.3 Duplicate Prevention
Three‑tier check:
1. **Before download:** `SELECT EXISTS(media_items WHERE chat_id=? AND message_id=?)` – fast, 100% accurate.  
2. **After download (optional):** compute MD5, query `SELECT id FROM media_items WHERE md5_hash=?` → if found, delete file and log.  
3. **File system:** if final path exists, either rename or skip.

---

## 5. Media Processing Pipeline

### 5.1 Categorization Engine
- Uses MIME type from TDLib (`message.content.mime_type`) and file extension.  
- Special rules: if extension `.apk` → APKs folder; if `.zip`, `.rar` → Archives folder; `.pdf` → Documents/PDFs.  
- AI can add further refinement (e.g., an image of a cat → tagged, but still in Photos).

### 5.2 FFmpeg Integration
- `FFmpegProcessor` has method `convertForCompatibility(input: File, output: File): Boolean`.  
- Configuration from settings: target codec (H.264), preset, CRF.  
- Progress emitted via callback → UI update.  
- Only triggered if original codec is not playable (or if user forces conversion for Google Drive).  
- Use temporary file, then replace original on success.

### 5.3 Thumbnail Generation
- For images: use Coil/Glide to downscale and save as JPEG.  
- For videos: use `MediaMetadataRetriever.getFrameAtTime()` or FFmpeg to extract a frame.  
- Thumbnails stored in `BaseDir/Thumbnails/` with a naming convention derived from media ID.

---

## 6. AI & Search Engine

### 6.1 On‑Device AI Tagging
- **Image classification:** TensorFlow Lite model (MobileNetV3) runs after download.  
- **OCR:** ML Kit Text Recognition (on‑device). Process screenshots and PDFs.  
- **NSFW detection:** ML Kit or custom TFLite model.  
- **Face grouping:** using ML Kit Face Detection, optional clustering.

All AI tasks are enqueued to a separate low‑priority `Executor` (e.g., `WorkManager` one‑time worker for post‑download processing). Results stored in `ai_tags` table and concatenated into FTS index.

### 6.2 Search Engine
- Uses Room FTS5.  
- Query builder supports:  
  - Free text (matches file name, OCR text, tags)  
  - Filter by media type, chat, date range, file size, private flag.  
- Exposed as `searchMedia(query: String, filters: MediaFilter): Flow<List<MediaItem>>`.  
- Data pipe: UI text field → ViewModel → UseCase → DAO FTS query → Flow → UI list.

---

## 7. Advanced Feature Integration

### 7.1 Streaming Engine (`:archiver-streaming`)
- Uses ExoPlayer with a custom `DataSource` that reads from a partially downloaded file.  
- When user taps "Play" while download is in progress, we start `DownloadService` with maximum priority for that file.  
- The `PartialFileDataSource` tracks the write position and serves data as it arrives.  
- We pre‑buffer the first segments for instant start.  
- **Flow:** Download task → file write (our pipe) → ExoPlayer read same file → seamless playback.

### 7.2 Local Media Server / NAS (`:archiver-nas`)
- Embed NanoHTTPD or a lightweight HTTP server.  
- Serve files from the organised folder structure.  
- Optional: UPnP/DLNA using `Cling` library or Chromecast via Cast SDK.  
- Toggle in settings; runs as foreground service when enabled.  
- **Data flow:** HTTP request → FileStorageManager lookup → stream file.

### 7.3 APK Intelligence Engine
- After downloading an APK, call `ApkAnalyzer` which uses `PackageManager.getPackageArchiveInfo()` (without installing) to extract:  
  - App name, version, target SDK, permissions, icon.  
- Store extracted metadata in a separate `apk_info` table.  
- Display a dedicated UI card with "Permissions" and "Install" button.

### 7.4 Auto Forwarding / Cloud Upload
- **Telegram Forward:** after successful download, call `sendCopy(chatId, fromChatId, messageId)` to forward the original message to a designated "backup" channel.  
- **Google Drive Upload:** a `WorkManager` worker picks new files (marked `upload_status=0`) and uploads them using Drive REST API; updates status to 1 on success.  
- Both are configurable per chat.

---

## 8. Full Database Architecture (Room)

Here is the complete schema with indices and triggers:

```sql
-- archive_chats
CREATE TABLE archive_chats (
    chat_id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    username TEXT,
    chat_type INTEGER NOT NULL,
    last_synced_msg_id INTEGER NOT NULL DEFAULT 0,
    is_sync_enabled INTEGER NOT NULL DEFAULT 1,
    private_encrypted INTEGER NOT NULL DEFAULT 0
);

-- media_items
CREATE TABLE media_items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    message_id INTEGER NOT NULL,
    chat_id INTEGER NOT NULL REFERENCES archive_chats(chat_id),
    media_type INTEGER NOT NULL,
    original_file_name TEXT,
    local_path TEXT NOT NULL,
    size_bytes INTEGER NOT NULL,
    md5_hash TEXT,
    phash TEXT,
    ocr_text TEXT,
    download_timestamp INTEGER NOT NULL,
    is_private INTEGER NOT NULL DEFAULT 0
);
CREATE UNIQUE INDEX idx_media_unique ON media_items(chat_id, message_id);

-- FTS virtual table
CREATE VIRTUAL TABLE media_items_fts USING fts5(
    original_file_name,
    ocr_text,
    ai_tags,
    content=media_items,
    content_rowid=id
);

-- Triggers to keep FTS in sync
CREATE TRIGGER media_ai AFTER INSERT ON media_items BEGIN
    INSERT INTO media_items_fts(rowid, original_file_name, ocr_text) VALUES (new.id, new.original_file_name, new.ocr_text);
END;
-- (similar triggers for DELETE, UPDATE)

-- download_queue
CREATE TABLE download_queue (
    task_id TEXT PRIMARY KEY,
    chat_id INTEGER NOT NULL,
    message_id INTEGER NOT NULL,
    file_id INTEGER NOT NULL,
    media_type INTEGER NOT NULL,
    target_path TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 0,
    retry_count INTEGER NOT NULL DEFAULT 0,
    status INTEGER NOT NULL DEFAULT 0
);

-- ai_tags
CREATE TABLE ai_tags (
    media_id INTEGER NOT NULL REFERENCES media_items(id),
    tag TEXT NOT NULL,
    confidence REAL,
    PRIMARY KEY (media_id, tag)
);

-- sync_history
CREATE TABLE sync_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    chat_id INTEGER,
    start_time INTEGER NOT NULL,
    end_time INTEGER,
    messages_scanned INTEGER,
    downloaded_count INTEGER,
    status TEXT
);
```

**Data pipes:**  
- Room DAO methods return `Flow` or `suspend` functions.  
- FTS queries use `MATCH` with the Room `@Query`.  
- The UI collects these flows and updates automatically.

---

## 9. File Storage Layout & Encryption

**Base directory:** user picks via SAF.  
**Private chat encryption:** when `private_encrypted` is true for a chat, files are stored inside an **encrypted virtual file system** (e.g., using `EncryptedFile` from `AndroidX Security` or a custom `FileLocker` that encrypts/decrypts on the fly using AES‑GCM with a key from Android Keystore). The `local_path` in DB still points to the decrypted location (which is a temporary decrypted version), but the actual stored file is encrypted. Alternatively, we can keep encrypted files with `.enc` extension and decrypt only when accessed.

**Cache reuse:** we keep a table `cache_index` that maps TDLib file IDs to local cache paths and their expiration. Before downloading, we check if the file is already in the cache and valid.

---

## 10. Security & Permissions

- **Session:** TDLib database is already encrypted (Telegram's own encryption). We don't store extra credentials.  
- **Our Room DB:** optionally encrypted using SQLCipher (passphrase derived from user password or biometric).  
- **Private media:** AES‑GCM with key stored in Android Keystore, bound to user authentication.  
- **Permissions:** only `READ_MEDIA_*`, `FOREGROUND_SERVICE`, `POST_NOTIFICATIONS`, `INTERNET`. SAF for file access – no `MANAGE_EXTERNAL_STORAGE` needed.  
- **Network security:** TDLib uses MTProto, already secure.

---

## 11. UI/UX Design & Screen Flows

All screens are built with **Jetpack Compose** and follow Material 3 guidelines. Navigation uses a `NavHost` with bottom navigation bar: **Home, Downloads, Browser, Search, Settings**.

**Dashboard:**  
- "Sync now" button, storage usage ring, recent downloads list, background sync status.

**Chat Picker:**  
- LazyColumn with search, multi‑selection, filter chips (Channel/Group/Private), limit input, media type selection.  
- "Start Download" triggers `StartDownloadUseCase`.

**Download Manager:**  
- Sections: Active (with progress and speed), Queued, Failed.  
- Swipe to cancel, tap to retry.

**Media Browser:**  
- Grid view with thumbnails (Coil), filters at top, pull‑to‑refresh.  
- Long press → share, delete, view details, re‑upload.

**Video Player:**  
- ExoPlayer with custom controls, download progress overlay, "Streaming" badge.

**AI Search:**  
- Search bar with voice input, filter sheet, results in list/grid with highlighted tags.

---

## 12. Development Roadmap (Phased)

### Phase 0 – Foundation (2 weeks)
- Fork Telegram‑FOSS, strip to minimal core.  
- Set up multi‑module Gradle project.  
- Integrate TDLib wrapper, create `ChatRepository` and `MediaRepository` skeletons.  
- Implement login flow reuse.

### Phase 1 – Core Download Engine (3 weeks)
- Build `download_queue` and `DownloadWorker` with foreground service.  
- Implement resume (store/read `last_synced_msg_id`).  
- Duplicate prevention.  
- Basic file storage (SAF) and folder structure.

### Phase 2 – UI & Management (3 weeks)
- Dashboard, Chat Picker, Download Manager, Media Browser.  
- FTS search with filters.  
- Statistics screen.

### Phase 3 – Background Sync & Cloud (2 weeks)
- Periodic WorkManager sync (scheduled).  
- Real‑time listener for selected chats.  
- Google Drive upload worker.  
- Auto‑forward to Telegram channel.

### Phase 4 – Advanced Media Processing (3 weeks)
- FFmpeg conversion on‑demand.  
- Thumbnail generation.  
- APK Intelligence engine.  
- Cache reuse integration.

### Phase 5 – AI & Streaming (3 weeks)
- On‑device image tagging, OCR, NSFW detection.  
- Perceptual hashing for duplicate images.  
- ExoPlayer streaming from partial downloads.

### Phase 6 – NAS & Polish (2 weeks)
- Local HTTP server (NanoHTTPD).  
- Chromecast support.  
- Private chat encryption.  
- Full test coverage, performance optimisation.

---

## 13. Complete Data Flow (Telegram Server → UI)

```
Telegram Cloud
    │
    ▼
TDLib (native) receives updateNewMessage / or history response
    │
    ▼
NotificationCenter (Telegram's event bus)
    │
    ▼
TdLibClient wrapper converts to Flow<Message>
    │
    ▼
StartDownloadUseCase / SyncEngine
    │
    ├─ Duplicate check (Room)
    │
    ├─ Enqueue DownloadTask (Room download_queue)
    │
    ▼
DownloadWorker (Foreground Service)
    │
    ├─ TDLib downloadFile → file.local.path (cache)
    │
    ├─ Copy to _Temp
    │
    ├─ [Optional] FFmpegProcessor
    │
    ├─ ThumbnailGenerator
    │
    ├─ Move to final folder (FileStorageManager)
    │
    ├─ AI analysis (WorkManager one‑time)
    │
    ├─ Insert media_item + ai_tags (Room)
    │
    ├─ Update archive_chats.last_synced_msg_id
    │
    └─ Mark queue task completed
    │
    ▼
UI update via Room Flow → ViewModel → Compose
```

---

## 14. Getting Started – Priority by Developer Profile

### If you're starting from scratch
Take **Phase 0 + Phase 1** as your foundation:
- Fork Telegram-FOSS and strip it to minimal core
- Set up the multi-module Gradle project
- Build the download queue and worker first

### If you want the fastest working prototype
Focus on just these three modules:
- `:telegram-base` (the fork — gives you login and TDLib for free)
- `:archiver-data` (TDLibWrapper + Room DB)
- `:archiver-sync` (the download worker)

Everything else — AI, NAS, streaming, FFmpeg — is additive. You can bolt those on later.

### If you're a solo developer
Honest priority order:

1. Login + chat list (from Telegram-FOSS fork — already done for you)
2. Download queue with resume logic (`last_synced_msg_id`)
3. File storage with SAF
4. Media browser UI
5. Search (FTS5 in Room)
6. Everything else

**Skip for now:** `:archiver-nas`, `:archiver-ai`, and streaming — these are impressive but not core to the value proposition. The core value is **reliable offline archiving**, which is Phase 0–2.

---

## 15. Final Word

This unified blueprint gives you **everything**:

- The **vision** of a full‑fledged Media OS  
- The **pragmatic implementation** starting from Telegram's own source  
- **Deep technical details**: data pipes, database schemas, flowcharts, and critical methods  
- **Modularity** so you can build progressively and never get lost

You now have a **real project blueprint** — one that you can hand to any Android engineer and start coding immediately, while still reaching the extraordinary feature set you dreamed of.

**Let's build Telegram Media OS.**
