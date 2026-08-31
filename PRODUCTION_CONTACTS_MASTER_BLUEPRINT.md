# Production Android Contacts Manager — Master Blueprint & Replication Prompt

> **Purpose**: This document is a complete, self-contained master blueprint and prompt specification. Giving this document to any AI coding agent allows it to reconstruct and build this exact production-grade, 120Hz-smooth Android Contacts Manager application from scratch.

---

## 1. System Overview & Core Mandates

- **Target OS**: Android 7.0 (API 24) to Android 15/17+ (API 37+)
- **Primary Language**: Kotlin 2.0+ (Strict Type Safety & Coroutines)
- **UI Framework**: Jetpack Compose with Material 3 (100% Declarative UI)
- **Architecture**: Clean Architecture + MVVM + Unidirectional Data Flow (UDF)
- **Data Layer**: Direct integration with Android's system **Contacts Provider** (`ContentResolver` & `ContactsContract`).
- **CRITICAL RULE**: Android Contacts Provider is the **Single Source of Truth**. **DO NOT** duplicate or mirror user contacts into a local Room database.
- **Role Constraint**: **DO NOT** request or assume fake `RoleManager` dialer/contacts roles (`ROLE_CONTACTS` or `ROLE_DIALER`).
- **Privacy & Security**: Zero Contact PII logging (names, numbers, emails, addresses must never be logged or transmitted). Offline-first.

---

## 2. Critical Performance Secrets (The 120Hz Compose Blueprint)

If an AI agent creates a naive `LazyColumn` in Jetpack Compose, the list will stutter and drop frames during rapid flings. The following rules **must** be enforced:

### A. Zero Allocations in Data Models
- **Never compute regex or string splits in getters on the UI thread**.
- `ContactSummary.initials`, `sortLetter`, and `stableKey` must be pre-calculated in constructor properties using primitive character indexing (`indexOf(' ')`), never `split("\\s+".toRegex())`.

```kotlin
private fun computeSortLetter(displayName: String): Char {
    val firstChar = displayName.trimStart().firstOrNull()?.uppercaseChar() ?: '#'
    return if (firstChar in 'A'..'Z') firstChar else '#'
}

private fun computeInitials(displayName: String): String {
    val trimmed = displayName.trim()
    if (trimmed.isEmpty()) return "?"
    val firstSpace = trimmed.indexOf(' ')
    return if (firstSpace > 0 && firstSpace + 1 < trimmed.length) {
        val nextChar = trimmed.substring(firstSpace + 1).trimStart().firstOrNull()
        if (nextChar != null && nextChar.isLetterOrDigit()) {
            "${trimmed[0].uppercaseChar()}${nextChar.uppercaseChar()}"
        } else {
            trimmed.take(2).uppercase()
        }
    } else {
        trimmed.take(2).uppercase()
    }
}

data class ContactSummary(
    val id: Long,
    val lookupKey: String,
    val displayName: String,
    val photoThumbnailUri: String? = null,
    val isFavorite: Boolean = false,
    val hasPhoneNumber: Boolean = false,
    val stableKey: String = if (lookupKey.isNotEmpty()) lookupKey else id.toString(),
    val sortLetter: Char = computeSortLetter(displayName),
    val initials: String = computeInitials(displayName)
)
```

### B. Flattened List Architecture with Strict `contentType`
- **Do not nest section headers conditionally inside contact item composables**.
- Pre-compute a flattened list on `Dispatchers.Default` using a sealed interface:

```kotlin
sealed interface ContactListItem {
    data class Header(val letter: Char) : ContactListItem
    data class Contact(val contact: ContactSummary) : ContactListItem
}
```
- In `LazyColumn`, supply dedicated keys and `contentType`:
```kotlin
items(
    items = listItems,
    key = { item ->
        when (item) {
            is ContactListItem.Header -> "header_${item.letter}"
            is ContactListItem.Contact -> item.contact.stableKey
        }
    },
    contentType = { item ->
        when (item) {
            is ContactListItem.Header -> "header"
            is ContactListItem.Contact -> "contact"
        }
    }
)
```

### C. Fixed Height Constraints (Zero Dynamic Remeasurement)
- Give `ContactRow` a strict fixed height (e.g. `Modifier.fillMaxWidth().height(58.dp)`).
- Give `AlphabetHeader` a strict fixed height (e.g. `Modifier.fillMaxWidth().height(32.dp)`).
- This allows Compose's `LazyLayout` to compute prefetch coordinates in $0\mu s$ without measuring layout bounds on the UI thread during high-speed flings.

### D. Zero-Layer GPU Clipping & `BasicText`
- Avoid `Modifier.clip(CircleShape)` on parent containers—it allocates off-screen GPU `saveLayer` buffers. Use `Modifier.background(color, CircleShape)`.
- Use **`BasicText`** with pre-cached `TextStyle` inside fast-scrolling list items to bypass `MaterialTheme` dynamic `CompositionLocal` lookups on every frame.

### E. Coil Hardware Bitmaps & In-Memory Cache
- In `Application.onCreate()`, configure `ImageLoaderFactory` with 25% RAM `MemoryCache` and 50MB `DiskCache`.
- In avatar items, pass `ImageRequest.Builder(context).allowHardware(true).crossfade(false).memoryCacheKey(photoUri).build()`.

### F. FastScroller Mutex Management
- In touch scrubbing (`FastScroller`), maintain a single `var scrollJob: Job?`. Cancel previous jobs before calling `lazyListState.scrollToItem(targetIndex)` to prevent Compose scroll mutex lock contention.
- Keep the touch strip strictly `28.dp` wide so it never interferes with list flings.

---

## 3. Contacts Provider & ContentResolver Blueprint

### A. Main List Lightweight Projection
Never query `ContactsContract.Data` for the main list. Query only lightweight columns:
```kotlin
val SUMMARY_PROJECTION = arrayOf(
    Contacts._ID,
    Contacts.LOOKUP_KEY,
    Contacts.DISPLAY_NAME_PRIMARY,
    Contacts.PHOTO_THUMBNAIL_URI,
    Contacts.STARRED,
    Contacts.HAS_PHONE_NUMBER
)
```

### B. Third-Party Shadow Sync Filtering (`IN_VISIBLE_GROUP = 1`)
WhatsApp, Telegram, and sync adapters create internal shadow contact records. To match Google Contacts and avoid duplicate list items:
- Filter the main query with `${Contacts.IN_VISIBLE_GROUP} = 1`.
- Include an automatic fallback to unfiltered queries if a device has no contact groups configured (e.g. SIM-only contacts).

### C. Contact Detail Deduplication
When opening contact details, raw contacts from Google, WhatsApp, SIM, and Truecaller are aggregated. Normalize and deduplicate:
- **Phone Numbers**: Strip non-digits (`replace("[\\s\\-\\(\\)\\.]".toRegex(), "")`) to deduplicate `86258 16285` vs `8625816285`.
- **Emails**: Trim & lowercase deduplication.
- **Websites & Addresses**: Normalized string deduplication.

### D. Crash Prevention on OEM Custom ROMs
- **NEVER** query non-standard internal columns like `RawContacts.RAW_CONTACT_IS_READ_ONLY`—it throws `IllegalArgumentException: Invalid column` on standard Android SQLite providers.
- Always wrap detail queries in `runCatching { ... }.getOrNull()`.

### E. Atomic Batch Writes
All creations, updates, and deletions must execute as atomic multi-row batches using `ContentResolver.applyBatch(ContactsContract.AUTHORITY, ops)` with back-references:
```kotlin
val ops = ArrayList<ContentProviderOperation>()
ops.add(ContentProviderOperation.newInsert(RawContacts.CONTENT_URI)
    .withValue(RawContacts.ACCOUNT_NAME, accountName)
    .withValue(RawContacts.ACCOUNT_TYPE, accountType)
    .build())

ops.add(ContentProviderOperation.newInsert(Data.CONTENT_URI)
    .withValueBackReference(Data.RAW_CONTACT_ID, 0)
    .withValue(Data.MIMETYPE, StructuredName.CONTENT_ITEM_TYPE)
    .withValue(StructuredName.GIVEN_NAME, givenName)
    .build())

contentResolver.applyBatch(ContactsContract.AUTHORITY, ops)
```

---

## 4. Real-Time Synchronization Engine

To eliminate the 5–10 second delay when switching between Google Contacts and this app:

1. **Root Authority Registration**: Register `ContentObserver` on:
   - `content://com.android.contacts` (`ContactsContract.AUTHORITY`)
   - `ContactsContract.Contacts.CONTENT_URI`
   - `ContactsContract.Data.CONTENT_URI`
   - `ContactsContract.RawContacts.CONTENT_URI`
2. **Instant Foreground Resume Trigger**:
   In `MainActivity.onResume()`, call `appContainer.contactsRepository.refresh()`, which emits to the shared flow immediately.
3. **Debounce Buffer**: Set to `100ms` for smooth conflation without perception latency.

---

## 5. Navigation & Screen Structure

- **Architecture**: Jetpack Navigation Compose with Kotlinx Serialization type-safe routes (`@Serializable`).
- **Main Tabs**:
  1. `Contacts`: Main alphabetical list, search bar, fast alphabet scroller, floating add button.
  2. `Favorites`: Live starred contacts synchronized with `Contacts.STARRED`.
  3. `Fix & Manage`: Contact health metrics (unnamed count, contacts without phone numbers, duplicate merge tools, VCF import/export).
  4. `Settings`: Theme mode (System/Light/Dark), Dynamic Color toggle (Android 12+), Sort order (First Name vs Last Name), Name format (First Name First vs Last Name First).
- **Secondary Screens**:
  - `ContactDetailScreen`: Photo header, Call, Text, Email, Share action chips (`FilledTonalIconButton`), expandable info cards, star toggle, delete dialog.
  - `ContactEditorScreen`: Name fields (Prefix, First, Middle, Last, Suffix), dynamic multi-phone, multi-email, address, company, birthday, notes, account destination picker, zero-permission Jetpack Photo Picker (`ActivityResultContracts.PickVisualMedia`).
  - `ContactsAccessSetupScreen`: Permission onboarding with explanation, privacy declaration, and system settings navigation.

---

## 6. Directory Structure & Key Files

```
app/src/main/java/com/vbappsstudio/contacts/
├── ContactsApplication.kt                # App entry point, DI Container init, Coil ImageLoaderFactory
├── MainActivity.kt                       # SplashScreen API, onResume sync trigger, NavHost root
├── core/
│   ├── di/AppContainer.kt                # Pure Kotlin Dependency Injection container
│   ├── dispatchers/CoroutineDispatchers.kt# IO, Default, Main testable dispatchers
│   ├── intents/ContactIntents.kt         # Safe external intent dispatcher (Call, SMS, Email, Share)
│   └── permissions/
│       ├── ContactsAccessState.kt        # NoAccess, ReadOnly, ReadWrite capability states
│       └── PermissionManager.kt          # Permission checker and app settings launcher
├── data/
│   ├── datasource/ContactsDataSource.kt  # Low-level ContentResolver wrapper
│   ├── model/
│   │   ├── ContactSummary.kt             # Lightweight, pre-calculated list item model
│   │   ├── ContactDetails.kt             # Full profile detail model
│   │   └── LabeledItem.kt                # Typed phone, email, address records
│   ├── observer/ContactsContentObserver.kt# Multi-URI real-time sync observer
│   ├── preferences/PreferencesRepository.kt# Jetpack DataStore preferences
│   ├── provider/
│   │   ├── ContactsQuery.kt              # Optimized SQLite projections, filters, and deduplicators
│   │   └── ContactsWriter.kt             # Atomic ContentProviderOperation batch writers
│   └── repository/ContactsRepository.kt  # Unified repository interface and implementation
├── domain/
│   ├── model/AlphabetSection.kt          # Flattened ContactListItem and grouping models
│   └── usecase/
│       ├── BuildAlphabetSectionsUseCase.kt# <50ms 10,000-contact sectioning & indexer
│       └── ContactsUseCases.kt           # Domain use cases (Get, Search, Details, Save, Delete, Favorite)
├── feature/
│   ├── contacts/                         # Main contact list & FastScroller
│   ├── detail/                           # Profile viewer & quick actions
│   ├── editor/                           # Multi-field editor & Photo Picker
│   ├── favorites/                        # Starred contacts
│   ├── fixandmanage/                     # Health metrics & cleanup tools
│   ├── monetization/                     # Decoupled Ad/IAP boundary
│   ├── settings/                         # DataStore theme & sorting preferences
│   ├── setup/                            # Permissions onboarding UI
│   └── startup/                          # Splash coordinator & router
└── ui/
    ├── navigation/                       # Type-safe AppNavHost & routes
    └── theme/                            # Material 3 dynamic color theme
```

---

## 7. Build Configuration & Dependencies

### `gradle/libs.versions.toml`
```toml
[versions]
agp = "8.8.2"
kotlin = "2.0.21"
composeBom = "2024.12.01"
coreKtx = "1.15.0"
coreSplashscreen = "1.0.1"
navigationCompose = "2.8.5"
datastorePreferences = "1.1.1"
coilCompose = "2.7.0"
kotlinxCoroutines = "1.9.0"
kotlinxSerialization = "1.7.3"

[libraries]
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
androidx-core-splashscreen = { group = "androidx.core", name = "core-splashscreen", version.ref = "coreSplashscreen" }
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
androidx-compose-material3 = { group = "androidx.compose.material3", name = "material3" }
androidx-compose-material-icons-extended = { group = "androidx.compose.material", name = "material-icons-extended" }
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }
androidx-datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastorePreferences" }
coil-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coilCompose" }
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "kotlinxCoroutines" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinxSerialization" }
```

### `app/build.gradle.kts`
```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.kotlin.serialization)
}

android {
    namespace = "com.vbappsstudio.contacts"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.vbappsstudio.contacts"
        minSdk = 24
        targetSdk = 35
        versionCode = 1
        versionName = "1.0.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    buildFeatures {
        compose = true
    }
}
```

---

## 8. Master Agent Replication Prompt

*Copy and paste the prompt below into any AI agent to replicate this exact application:*

```text
You are a Principal Android Architect. Build a production-grade Android Contacts Manager application for Google Play with support for 10,000+ contacts and 120Hz smooth scrolling.

Follow the specifications in the master blueprint:
1. Android Contacts Provider is the single source of truth using ContentResolver and ContactsContract. Do NOT duplicate contacts into a local Room database.
2. Real-time synchronization using a ContentObserver on root contacts authority with instant foreground onResume() refresh and 100ms debounce.
3. Strict lightweight projection query for main list (_ID, LOOKUP_KEY, DISPLAY_NAME_PRIMARY, PHOTO_THUMBNAIL_URI, STARRED, HAS_PHONE_NUMBER).
4. Filter out shadow third-party sync contacts with Contacts.IN_VISIBLE_GROUP = 1 matching Google Contacts.
5. In ContactSummary, precalculate initials and sort keys with zero regex and zero array allocations.
6. In Compose LazyColumn, implement flattened list items (ContactListItem.Header and ContactListItem.Contact) with strict contentType and fixed heights (58dp for rows, 32dp for headers) and BasicText for 120Hz scrolling.
7. FastScroller alphabet scrubber with single Job cancellation to prevent scroll mutex contention.
8. Normalize and deduplicate multi-raw contact phone numbers (e.g. "86258 16285" vs "8625816285").
9. Multi-field Contact Editor with zero-permission Jetpack Photo Picker (PickVisualMedia) and atomic ContentProviderOperation.applyBatch writes.
10. Android SplashScreen API, Permissions onboarding screen, DataStore settings, and decoupled Monetization boundary.
11. Pass all unit tests and build cleanly with R8 release minification enabled.
```
