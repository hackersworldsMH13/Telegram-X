# TELEGRAM MEDIA OS — UNIFIED ARCHITECTURE BLUEPRINT

*Merging the visionary "Media OS" with the production‑ready "Archiver Pro" foundation*

**Status:** Draft — Phase 0 not yet started  
**Date:** 2026-05-15  
**Repository:** hackersworldsMH13/Telegram-X

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

### 3.2 Domain Layer (`:archiver-core`)

Contains **pure business logic** – no Android dependencies.

**Key Use Cases:**
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

#### TDLib Wrapper (Kotlin coroutines)

```kotlin
class TdLibClient @Inject constructor(
    private val connectionsManager: ConnectionsManager
) {
    suspend fun getChatHistory(
        chatId: Long, 
        fromMessageId: Long, 
        limit: Int
    ): List<TdApi.Message> = suspendCoroutine { cont ->
        connectionsManager.sendRequest(
            TdApi.GetChatHistory(chatId, fromMessageId, 0, limit, false),
            { response -> cont.resume((response as TdApi.Messages).messages.asList()) },
            { error -> cont.resumeWithException(Exception(error.message)) }
        )
    }
}
```

#### Room Database Schema

**Table: `archive_chats`**
| Column               | Type    | Description                     |
|----------------------|---------|---------------------------------|
| chat_id              | Long PK | Telegram chat ID                |
| title                | String  | Chat display name               |
| username             | String? | Unique username if available    |
| chat_type            | Int     | Enum: 0=private,1=group,…       |
| last_synced_msg_id   | Long    | Resume point (default 0)        |
| is_sync_enabled      | Boolean | For scheduled sync              |
| private_encrypted    | Boolean | Whether files are encrypted     |

**Table: `media_items`**
| Column               | Type    | Notes                           |
|----------------------|---------|---------------------------------|
| id                   | Long PK | auto‑generated                  |
| message_id           | Long    | Telegram message ID             |
| chat_id              | Long FK | → archive_chats.chat_id         |
| media_type           | Int     | Enum: PHOTO, VIDEO, AUDIO, …    |
| original_file_name   | String? | Original filename from Telegram |
| local_path           | String  | Full path on device             |
| size_bytes           | Long    | File size in bytes              |
| md5_hash             | String? | For duplicate check             |
| phash                | String? | Perceptual hash (for images)    |
| ocr_text             | String? | Extracted text                  |
| download_timestamp   | Long    | Epoch millis                    |
| is_private           | Boolean | Private chat file flag          |

**Table: `download_queue`**
| Column        | Type    | Description |
|---------------|---------|-------------|
| task_id       | String PK | chat_id + message_id |
| chat_id       | Long    | Parent chat |
| message_id    | Long    | Message ID  |
| file_id       | Int     | TDLib file ID |
| media_type    | Int     | Media type enum |
| target_path   | String  | Final destination |
| priority      | Int     | Processing priority |
| retry_count   | Int     | Retry counter |
| status        | Int     | 0=PENDING,1=DOWNLOADING,2=DONE,3=FAILED |

---

## 4. Core Engine: Download & Sync Pipeline

### 4.1 End‑to‑End Message Processing

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
   ├─ Does message contain selected media type? → YES → enqueue
   └─ NO → update last_synced_msg_id (skip)
        │
        ▼
DownloadQueue (Room table) – FIFO processing
        │
        ▼
DownloadWorker (ForegroundService)
   ├─ Dequeue next task
   ├─ TDLib downloadFile(fileId)
   ├─ Copy from cache → _Temp/target
   ├─ Optional: FFmpeg conversion
   ├─ Generate thumbnail
   ├─ Move to final organized path
   ├─ Insert media_item + ai_tags (Room)
   ├─ Update last_synced_msg_id
   └─ Mark task completed
```

### 4.2 Resume Logic

- Store `last_synced_msg_id` per chat (initially 0)
- On restart: `getChatHistory(chatId, fromMessageId = last_synced_msg_id, ...)`
- **Important:** TDLib returns messages in descending order (newest first)
- Process in reverse to maintain forward progress
- Update `last_synced_msg_id` only after successful processing

### 4.3 Duplicate Prevention

Three‑tier check:
1. **Before download:** `SELECT EXISTS(media_items WHERE chat_id=? AND message_id=?)` – 100% accurate
2. **After download:** compute MD5, query by hash → if found, delete duplicate
3. **File system:** check if target path already exists

---

## 5. File Storage Layout

```
BaseDir/ (user selects via SAF)
 ├── Channels/<ChannelName>/
 │    ├── Photos/
 │    ├── Videos/
 │    ├── Audio/
 │    ├── Documents/
 │    ├── APKs/
 │    ├── Archives/
 │    └── Metadata/
 ├── PrivateChats/<ContactName>/
 │    └── … (same structure, optionally encrypted)
 ├── _Temp/ (in‑progress downloads)
 └── _CacheIndex/ (symlinks or DB records for Telegram cache reuse)
```

---

## 6. Security & Permissions

- **TDLib Session:** Already encrypted (Telegram's MTProto)
- **Room DB:** Optional SQLCipher encryption (key from Android Keystore)
- **Private media:** AES‑GCM with biometric binding
- **Permissions:** Only `READ_MEDIA_*`, `FOREGROUND_SERVICE`, `POST_NOTIFICATIONS`, `INTERNET`
- **No need:** `MANAGE_EXTERNAL_STORAGE` (using SAF instead)

---

## 7. Development Roadmap (16 weeks)

### Phase 0 – Foundation (2 weeks)
- Fork Telegram‑FOSS, strip to minimal core
- Set up multi‑module Gradle project
- Integrate TDLib wrapper
- Implement login flow reuse

### Phase 1 – Core Download Engine (3 weeks)
- Build `download_queue` and `DownloadWorker`
- Implement resume logic (`last_synced_msg_id`)
- Duplicate prevention
- Basic SAF storage + folder structure

### Phase 2 – UI & Management (3 weeks)
- Dashboard, Chat Picker, Download Manager, Media Browser
- FTS search with filters
- Statistics screen

### Phase 3 – Background Sync & Cloud (2 weeks)
- Periodic WorkManager sync
- Google Drive upload worker
- Auto‑forward to Telegram backup channel

### Phase 4 – Advanced Media Processing (3 weeks)
- FFmpeg conversion on‑demand
- Thumbnail generation
- APK Intelligence engine
- Cache reuse integration

### Phase 5 – AI & Streaming (3 weeks)
- On‑device image tagging (TFLite)
- OCR and NSFW detection
- ExoPlayer streaming from partial downloads

### Phase 6 – NAS & Polish (2 weeks)
- Local HTTP server (NanoHTTPD)
- Chromecast support
- Private chat encryption
- Full test coverage, performance optimization

---

## 8. Solo Developer Priority

1. Login + chat list (Telegram-FOSS fork)
2. Download queue with resume (`last_synced_msg_id`)
3. File storage (SAF) and folder organization
4. Media browser UI
5. FTS search with filters
6. **Later:** AI, NAS, streaming

**Core value:** Reliable offline archiving (Phase 0–2)

---

## 9. Final Word

This blueprint provides:

- ✅ Complete vision of a full‑fledged Media OS
- ✅ Pragmatic implementation starting from Telegram's source
- ✅ Deep technical details: data pipes, schemas, flowcharts
- ✅ Modularity for progressive development
- ✅ Clear implementation path from solo to team

**Ready to build. Let's go.**
