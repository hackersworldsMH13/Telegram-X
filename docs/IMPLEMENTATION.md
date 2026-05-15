# IMPLEMENTATION GUIDE – Telegram Media OS

**Start Date:** 2026-05-15  
**Phase:** 0 (Foundation)  
**Duration:** 16 weeks total

---

## Getting Started: Phase 0 – Foundation (Weeks 1-2)

This phase focuses on setting up the project infrastructure and integrating the Telegram-FOSS base.

### Step 1: Fork & Strip Telegram-FOSS

**Action:**
1. Fork [Telegram-FOSS](https://github.com/Telegram-FOSS-Team/Telegram-FOSS) to your account
2. Clone locally: `git clone https://github.com/YOUR_USERNAME/Telegram-FOSS.git`
3. Create a new branch: `git checkout -b telegram-media-os-base`

**What to keep:**
- `TdLib` native binaries and JNI wrapper
- `ConnectionsManager` (TDLib communication)
- `NotificationCenter` (event bus)
- `FileLoader`, `ImageLoader` utilities
- `LoginActivity`, basic session persistence

**What to remove:**
- All UI screens except login (strip `ChatActivity`, `PhotoViewer`, etc.)
- Telegram's built-in download manager
- Telegram's backup/cloud sync features
- Unrelated features (Stories, Reactions, etc.)

**Result:** A minimal ~50MB APK with just TDLib + login capability.

---

### Step 2: Create Multi-Module Gradle Project

**Directory Structure:**

```
Telegram-X/
├── settings.gradle.kts
├── build.gradle.kts (root)
├── gradle.properties
│
├── app/
│   ├── build.gradle.kts
│   └── src/main/...
│
├── :telegram-base/
│   ├── build.gradle.kts
│   └── src/main/...
│
├── :archiver-core/
│   ├── build.gradle.kts
│   └── src/main/...
│
├── :archiver-data/
│   ├── build.gradle.kts
│   └── src/main/...
│
├── :archiver-ui/
│   ├── build.gradle.kts
│   └── src/main/...
│
├── :archiver-sync/
│   ├── build.gradle.kts
│   └── src/main/...
│
└── docs/
    └── ARCHITECTURE.md
```

**Root build.gradle.kts:**

```kotlin
plugins {
    id("com.android.application") version "8.1.0" apply false
    id("com.android.library") version "8.1.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.0" apply false
    id("com.google.dagger.hilt.android") version "2.47" apply false
}

ext {
    set("compileSdkVersion", 34)
    set("minSdkVersion", 24)
    set("targetSdkVersion", 34)
    set("kotlinVersion", "1.9.0")
}
```

**settings.gradle.kts:**

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "Telegram-X"
include(
    ":app",
    ":telegram-base",
    ":archiver-core",
    ":archiver-data",
    ":archiver-ui",
    ":archiver-sync"
)
```

---

### Step 3: Define Domain Models (`:archiver-core`)

**File: `archiver-core/src/main/kotlin/com/telegram/archiver/core/domain/model/ArchiveChat.kt`**

```kotlin
package com.telegram.archiver.core.domain.model

data class ArchiveChat(
    val chatId: Long,
    val title: String,
    val username: String? = null,
    val chatType: ChatType,
    val lastSyncedMessageId: Long = 0L,
    val isSyncEnabled: Boolean = true,
    val isPrivateEncrypted: Boolean = false
)

enum class ChatType {
    PRIVATE_CHAT,
    GROUP_CHAT,
    SUPERGROUP,
    CHANNEL,
    SECRET_CHAT
}
```

**File: `archiver-core/src/main/kotlin/com/telegram/archiver/core/domain/model/MediaItem.kt`**

```kotlin
package com.telegram.archiver.core.domain.model

import java.io.File

data class MediaItem(
    val id: Long,
    val messageId: Long,
    val chatId: Long,
    val mediaType: MediaType,
    val originalFileName: String? = null,
    val localPath: String,
    val sizeBytes: Long,
    val md5Hash: String? = null,
    val phash: String? = null,
    val ocrText: String? = null,
    val downloadTimestamp: Long,
    val isPrivate: Boolean = false
)

enum class MediaType {
    PHOTO,
    VIDEO,
    AUDIO,
    DOCUMENT,
    ANIMATION,
    VOICE,
    VIDEO_NOTE,
    STICKER,
    ARCHIVE,
    APK
}
```

**File: `archiver-core/src/main/kotlin/com/telegram/archiver/core/domain/model/DownloadTask.kt`**

```kotlin
package com.telegram.archiver.core.domain.model

data class DownloadTask(
    val taskId: String,
    val chatId: Long,
    val messageId: Long,
    val fileId: Int,
    val mediaType: MediaType,
    val targetPath: String,
    val priority: Int = 0,
    val retryCount: Int = 0,
    val status: DownloadStatus = DownloadStatus.PENDING
)

enum class DownloadStatus {
    PENDING,
    DOWNLOADING,
    COMPLETED,
    FAILED,
    CANCELLED
}
```

---

### Step 4: Define Repository Interfaces (`:archiver-core`)

**File: `archiver-core/src/main/kotlin/com/telegram/archiver/core/domain/repository/ChatRepository.kt`**

```kotlin
package com.telegram.archiver.core.domain.repository

import com.telegram.archiver.core.domain.model.ArchiveChat
import kotlinx.coroutines.flow.Flow

interface ChatRepository {
    suspend fun getDialogs(limit: Int = 100): List<ArchiveChat>
    suspend fun updateLastSyncMessageId(chatId: Long, msgId: Long)
    suspend fun getChatById(chatId: Long): ArchiveChat?
    fun observeChats(): Flow<List<ArchiveChat>>
}
```

**File: `archiver-core/src/main/kotlin/com/telegram/archiver/core/domain/repository/MediaRepository.kt`**

```kotlin
package com.telegram.archiver.core.domain.repository

import com.telegram.archiver.core.domain.model.MediaItem
import kotlinx.coroutines.flow.Flow

interface MediaRepository {
    suspend fun recordDownload(media: MediaItem): Long
    suspend fun isDuplicate(chatId: Long, msgId: Long): Boolean
    suspend fun isDuplicateByHash(hash: String): Boolean
    fun search(query: String): Flow<List<MediaItem>>
    fun observeMediaItems(): Flow<List<MediaItem>>
}
```

**File: `archiver-core/src/main/kotlin/com/telegram/archiver/core/domain/repository/DownloadQueueRepository.kt`**

```kotlin
package com.telegram.archiver.core.domain.repository

import com.telegram.archiver.core.domain.model.DownloadTask
import com.telegram.archiver.core.domain.model.DownloadStatus
import kotlinx.coroutines.flow.Flow

interface DownloadQueueRepository {
    suspend fun enqueue(task: DownloadTask)
    suspend fun dequeue(limit: Int): List<DownloadTask>
    suspend fun markCompleted(taskId: String)
    suspend fun markFailed(taskId: String, error: String)
    fun observeQueue(): Flow<List<DownloadTask>>
}
```

---

### Step 5: Create Use Cases (`:archiver-core`)

**File: `archiver-core/src/main/kotlin/com/telegram/archiver/core/domain/usecase/FetchChatsUseCase.kt`**

```kotlin
package com.telegram.archiver.core.domain.usecase

import com.telegram.archiver.core.domain.model.ArchiveChat
import com.telegram.archiver.core.domain.repository.ChatRepository
import javax.inject.Inject

class FetchChatsUseCase @Inject constructor(
    private val chatRepository: ChatRepository
) {
    suspend operator fun invoke(limit: Int = 100): List<ArchiveChat> {
        return chatRepository.getDialogs(limit)
    }
}
```

**File: `archiver-core/src/main/kotlin/com/telegram/archiver/core/domain/usecase/StartDownloadUseCase.kt`**

```kotlin
package com.telegram.archiver.core.domain.usecase

import com.telegram.archiver.core.domain.model.DownloadTask
import com.telegram.archiver.core.domain.model.MediaType
import com.telegram.archiver.core.domain.repository.ChatRepository
import com.telegram.archiver.core.domain.repository.DownloadQueueRepository
import com.telegram.archiver.core.domain.repository.MediaRepository
import javax.inject.Inject

class StartDownloadUseCase @Inject constructor(
    private val chatRepository: ChatRepository,
    private val downloadQueueRepository: DownloadQueueRepository,
    private val mediaRepository: MediaRepository
) {
    suspend operator fun invoke(
        chatIds: List<Long>,
        mediaTypes: List<MediaType>,
        limit: Int = 1000
    ) {
        chatIds.forEach { chatId ->
            val chat = chatRepository.getChatById(chatId) ?: return@forEach
            
            // Load last synced message ID (resume point)
            val fromMessageId = chat.lastSyncedMessageId
            
            // Fetch messages from TDLib (mock for now)
            val messages = fetchMessagesFromTDLib(chatId, fromMessageId, limit)
            
            // Process each message
            messages.forEach { message ->
                // Check for duplicate
                if (mediaRepository.isDuplicate(chatId, message.messageId)) {
                    return@forEach
                }
                
                // Check media type
                if (!mediaTypes.contains(message.mediaType)) {
                    return@forEach
                }
                
                // Create download task
                val task = DownloadTask(
                    taskId = "$chatId-${message.messageId}",
                    chatId = chatId,
                    messageId = message.messageId,
                    fileId = message.fileId,
                    mediaType = message.mediaType,
                    targetPath = buildTargetPath(chat, message)
                )
                
                downloadQueueRepository.enqueue(task)
            }
        }
    }
    
    private suspend fun fetchMessagesFromTDLib(
        chatId: Long,
        fromMessageId: Long,
        limit: Int
    ): List<Message> {
        // This will be implemented in data layer
        return emptyList()
    }
    
    private fun buildTargetPath(chat: Chat, message: Message): String {
        // This will be implemented in data layer
        return ""
    }
}

// Mock models for now
data class Message(
    val messageId: Long,
    val fileId: Int,
    val mediaType: MediaType
)

data class Chat(
    val chatId: Long,
    val title: String
)
```

---

### Step 6: Set Up Room Database (`:archiver-data`)

**File: `archiver-data/src/main/kotlin/com/telegram/archiver/data/db/ArchiveDatabase.kt`**

```kotlin
package com.telegram.archiver.data.db

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import com.telegram.archiver.data.db.dao.ArchiveChatDao
import com.telegram.archiver.data.db.dao.MediaItemDao
import com.telegram.archiver.data.db.dao.DownloadQueueDao
import com.telegram.archiver.data.db.entity.ArchiveChatEntity
import com.telegram.archiver.data.db.entity.MediaItemEntity
import com.telegram.archiver.data.db.entity.DownloadQueueEntity

@Database(
    entities = [
        ArchiveChatEntity::class,
        MediaItemEntity::class,
        DownloadQueueEntity::class
    ],
    version = 1,
    exportSchema = true
)
abstract class ArchiveDatabase : RoomDatabase() {
    abstract fun archiveChatDao(): ArchiveChatDao
    abstract fun mediaItemDao(): MediaItemDao
    abstract fun downloadQueueDao(): DownloadQueueDao

    companion object {
        @Volatile
        private var INSTANCE: ArchiveDatabase? = null

        fun getDatabase(context: Context): ArchiveDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    ArchiveDatabase::class.java,
                    "archive.db"
                ).build().also { INSTANCE = it }
            }
        }
    }
}
```

**File: `archiver-data/src/main/kotlin/com/telegram/archiver/data/db/entity/ArchiveChatEntity.kt`**

```kotlin
package com.telegram.archiver.data.db.entity

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "archive_chats")
data class ArchiveChatEntity(
    @PrimaryKey
    val chatId: Long,
    val title: String,
    val username: String? = null,
    val chatType: Int,
    val lastSyncedMsgId: Long = 0L,
    val isSyncEnabled: Boolean = true,
    val privateEncrypted: Boolean = false
)
```

**File: `archiver-data/src/main/kotlin/com/telegram/archiver/data/db/entity/MediaItemEntity.kt`**

```kotlin
package com.telegram.archiver.data.db.entity

import androidx.room.Entity
import androidx.room.ForeignKey
import androidx.room.Index
import androidx.room.PrimaryKey

@Entity(
    tableName = "media_items",
    foreignKeys = [
        ForeignKey(
            entity = ArchiveChatEntity::class,
            parentColumns = ["chatId"],
            childColumns = ["chatId"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [
        Index(value = ["chatId", "messageId"], unique = true)
    ]
)
data class MediaItemEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val messageId: Long,
    val chatId: Long,
    val mediaType: Int,
    val originalFileName: String? = null,
    val localPath: String,
    val sizeBytes: Long,
    val md5Hash: String? = null,
    val phash: String? = null,
    val ocrText: String? = null,
    val downloadTimestamp: Long,
    val isPrivate: Boolean = false
)
```

**File: `archiver-data/src/main/kotlin/com/telegram/archiver/data/db/entity/DownloadQueueEntity.kt`**

```kotlin
package com.telegram.archiver.data.db.entity

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "download_queue")
data class DownloadQueueEntity(
    @PrimaryKey
    val taskId: String,
    val chatId: Long,
    val messageId: Long,
    val fileId: Int,
    val mediaType: Int,
    val targetPath: String,
    val priority: Int = 0,
    val retryCount: Int = 0,
    val status: Int = 0 // 0=PENDING, 1=DOWNLOADING, 2=DONE, 3=FAILED
)
```

---

### Step 7: Create DAOs

**File: `archiver-data/src/main/kotlin/com/telegram/archiver/data/db/dao/ArchiveChatDao.kt`**

```kotlin
package com.telegram.archiver.data.db.dao

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import androidx.room.Update
import com.telegram.archiver.data.db.entity.ArchiveChatEntity
import kotlinx.coroutines.flow.Flow

@Dao
interface ArchiveChatDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertChat(chat: ArchiveChatEntity)

    @Update
    suspend fun updateChat(chat: ArchiveChatEntity)

    @Query("SELECT * FROM archive_chats WHERE chatId = :chatId")
    suspend fun getChatById(chatId: Long): ArchiveChatEntity?

    @Query("SELECT * FROM archive_chats ORDER BY title ASC")
    suspend fun getAllChats(): List<ArchiveChatEntity>

    @Query("SELECT * FROM archive_chats ORDER BY title ASC")
    fun observeChats(): Flow<List<ArchiveChatEntity>>

    @Query("UPDATE archive_chats SET lastSyncedMsgId = :msgId WHERE chatId = :chatId")
    suspend fun updateLastSyncedMessageId(chatId: Long, msgId: Long)
}
```

**File: `archiver-data/src/main/kotlin/com/telegram/archiver/data/db/dao/MediaItemDao.kt`**

```kotlin
package com.telegram.archiver.data.db.dao

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import com.telegram.archiver.data.db.entity.MediaItemEntity
import kotlinx.coroutines.flow.Flow

@Dao
interface MediaItemDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertMediaItem(item: MediaItemEntity): Long

    @Query(
        "SELECT EXISTS(SELECT 1 FROM media_items " +
        "WHERE chatId = :chatId AND messageId = :messageId)"
    )
    suspend fun isDuplicate(chatId: Long, messageId: Long): Boolean

    @Query(
        "SELECT EXISTS(SELECT 1 FROM media_items WHERE md5Hash = :hash)"
    )
    suspend fun isDuplicateByHash(hash: String): Boolean

    @Query("SELECT * FROM media_items ORDER BY downloadTimestamp DESC")
    fun observeMediaItems(): Flow<List<MediaItemEntity>>

    @Query(
        "SELECT * FROM media_items WHERE originalFileName LIKE :query " +
        "OR ocrText LIKE :query ORDER BY downloadTimestamp DESC"
    )
    fun search(query: String): Flow<List<MediaItemEntity>>
}
```

**File: `archiver-data/src/main/kotlin/com/telegram/archiver/data/db/dao/DownloadQueueDao.kt`**

```kotlin
package com.telegram.archiver.data.db.dao

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import com.telegram.archiver.data.db.entity.DownloadQueueEntity
import kotlinx.coroutines.flow.Flow

@Dao
interface DownloadQueueDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertTask(task: DownloadQueueEntity)

    @Query(
        "SELECT * FROM download_queue " +
        "WHERE status = 0 " +
        "ORDER BY priority DESC, taskId ASC " +
        "LIMIT :limit"
    )
    suspend fun dequeueNextTasks(limit: Int): List<DownloadQueueEntity>

    @Query("UPDATE download_queue SET status = 2 WHERE taskId = :taskId")
    suspend fun markCompleted(taskId: String)

    @Query("UPDATE download_queue SET status = 3, retryCount = retryCount + 1 WHERE taskId = :taskId")
    suspend fun markFailed(taskId: String)

    @Query("SELECT * FROM download_queue ORDER BY priority DESC")
    fun observeQueue(): Flow<List<DownloadQueueEntity>>
}
```

---

### Step 8: Create Repository Implementations (`:archiver-data`)

**File: `archiver-data/src/main/kotlin/com/telegram/archiver/data/repository/ChatRepositoryImpl.kt`**

```kotlin
package com.telegram.archiver.data.repository

import com.telegram.archiver.core.domain.model.ArchiveChat
import com.telegram.archiver.core.domain.model.ChatType
import com.telegram.archiver.core.domain.repository.ChatRepository
import com.telegram.archiver.data.db.dao.ArchiveChatDao
import com.telegram.archiver.data.db.entity.ArchiveChatEntity
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import javax.inject.Inject

class ChatRepositoryImpl @Inject constructor(
    private val chatDao: ArchiveChatDao
) : ChatRepository {

    override suspend fun getDialogs(limit: Int): List<ArchiveChat> {
        return chatDao.getAllChats().map { it.toDomain() }
    }

    override suspend fun updateLastSyncMessageId(chatId: Long, msgId: Long) {
        chatDao.updateLastSyncedMessageId(chatId, msgId)
    }

    override suspend fun getChatById(chatId: Long): ArchiveChat? {
        return chatDao.getChatById(chatId)?.toDomain()
    }

    override fun observeChats(): Flow<List<ArchiveChat>> {
        return chatDao.observeChats().map { list -> list.map { it.toDomain() } }
    }

    private fun ArchiveChatEntity.toDomain(): ArchiveChat {
        return ArchiveChat(
            chatId = chatId,
            title = title,
            username = username,
            chatType = ChatType.values()[chatType],
            lastSyncedMessageId = lastSyncedMsgId,
            isSyncEnabled = isSyncEnabled,
            isPrivateEncrypted = privateEncrypted
        )
    }
}
```

**File: `archiver-data/src/main/kotlin/com/telegram/archiver/data/repository/MediaRepositoryImpl.kt`**

```kotlin
package com.telegram.archiver.data.repository

import com.telegram.archiver.core.domain.model.MediaItem
import com.telegram.archiver.core.domain.model.MediaType
import com.telegram.archiver.core.domain.repository.MediaRepository
import com.telegram.archiver.data.db.dao.MediaItemDao
import com.telegram.archiver.data.db.entity.MediaItemEntity
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import javax.inject.Inject

class MediaRepositoryImpl @Inject constructor(
    private val mediaItemDao: MediaItemDao
) : MediaRepository {

    override suspend fun recordDownload(media: MediaItem): Long {
        return mediaItemDao.insertMediaItem(media.toEntity())
    }

    override suspend fun isDuplicate(chatId: Long, msgId: Long): Boolean {
        return mediaItemDao.isDuplicate(chatId, msgId)
    }

    override suspend fun isDuplicateByHash(hash: String): Boolean {
        return mediaItemDao.isDuplicateByHash(hash)
    }

    override fun search(query: String): Flow<List<MediaItem>> {
        return mediaItemDao.search("%$query%").map { list -> 
            list.map { it.toDomain() }
        }
    }

    override fun observeMediaItems(): Flow<List<MediaItem>> {
        return mediaItemDao.observeMediaItems().map { list -> 
            list.map { it.toDomain() }
        }
    }

    private fun MediaItem.toEntity(): MediaItemEntity {
        return MediaItemEntity(
            id = id,
            messageId = messageId,
            chatId = chatId,
            mediaType = mediaType.ordinal,
            originalFileName = originalFileName,
            localPath = localPath,
            sizeBytes = sizeBytes,
            md5Hash = md5Hash,
            phash = phash,
            ocrText = ocrText,
            downloadTimestamp = downloadTimestamp,
            isPrivate = isPrivate
        )
    }

    private fun MediaItemEntity.toDomain(): MediaItem {
        return MediaItem(
            id = id,
            messageId = messageId,
            chatId = chatId,
            mediaType = MediaType.values()[mediaType],
            originalFileName = originalFileName,
            localPath = localPath,
            sizeBytes = sizeBytes,
            md5Hash = md5Hash,
            phash = phash,
            ocrText = ocrText,
            downloadTimestamp = downloadTimestamp,
            isPrivate = isPrivate
        )
    }
}
```

---

### Step 9: Setup Dependency Injection (`:app`)

**File: `app/src/main/kotlin/com/telegram/archiver/di/DatabaseModule.kt`**

```kotlin
package com.telegram.archiver.di

import android.content.Context
import androidx.room.Room
import com.telegram.archiver.data.db.ArchiveDatabase
import com.telegram.archiver.data.db.dao.ArchiveChatDao
import com.telegram.archiver.data.db.dao.MediaItemDao
import com.telegram.archiver.data.db.dao.DownloadQueueDao
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Singleton
    @Provides
    fun provideArchiveDatabase(
        @ApplicationContext context: Context
    ): ArchiveDatabase {
        return Room.databaseBuilder(
            context,
            ArchiveDatabase::class.java,
            "archive.db"
        ).build()
    }

    @Singleton
    @Provides
    fun provideArchiveChatDao(database: ArchiveDatabase): ArchiveChatDao {
        return database.archiveChatDao()
    }

    @Singleton
    @Provides
    fun provideMediaItemDao(database: ArchiveDatabase): MediaItemDao {
        return database.mediaItemDao()
    }

    @Singleton
    @Provides
    fun provideDownloadQueueDao(database: ArchiveDatabase): DownloadQueueDao {
        return database.downloadQueueDao()
    }
}
```

**File: `app/src/main/kotlin/com/telegram/archiver/di/RepositoryModule.kt`**

```kotlin
package com.telegram.archiver.di

import com.telegram.archiver.core.domain.repository.ChatRepository
import com.telegram.archiver.core.domain.repository.MediaRepository
import com.telegram.archiver.core.domain.repository.DownloadQueueRepository
import com.telegram.archiver.data.repository.ChatRepositoryImpl
import com.telegram.archiver.data.repository.MediaRepositoryImpl
import com.telegram.archiver.data.repository.DownloadQueueRepositoryImpl
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    abstract fun bindChatRepository(impl: ChatRepositoryImpl): ChatRepository

    @Binds
    abstract fun bindMediaRepository(impl: MediaRepositoryImpl): MediaRepository

    @Binds
    abstract fun bindDownloadQueueRepository(impl: DownloadQueueRepositoryImpl): DownloadQueueRepository
}
```

---

### Step 10: Create Application Class

**File: `app/src/main/kotlin/com/telegram/archiver/TelegramMediaOsApp.kt`**

```kotlin
package com.telegram.archiver

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class TelegramMediaOsApp : Application()
```

---

## Next Steps

1. **Complete Phase 0:** Build the basic UI layer (Dashboard, Chat Picker)
2. **Start Phase 1:** Implement TDLib wrapper and download worker
3. **Add testing:** Unit tests for use cases and repositories

Would you like me to continue with:
- Phase 1 implementation (Download engine)?
- UI layer setup (Compose screens)?
- Gradle build configuration for all modules?
- TDLib wrapper implementation?

Let me know what you'd like to focus on next!
