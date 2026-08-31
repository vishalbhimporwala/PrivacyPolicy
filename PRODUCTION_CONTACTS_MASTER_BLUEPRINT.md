# Production Android Contacts Manager — Master Blueprint & Complete AI Replication Prompt

> **Purpose**: This document contains the exhaustive, screen-by-screen, feature-by-feature master prompt. Giving this file to any AI coding agent provides all specifications, code structures, data contracts, and performance secrets required to build this exact production-grade, 120Hz-smooth Android Contacts Manager application.

---

# 📋 THE COMPLETE AI AGENT MASTER PROMPT

*Copy everything between the start and end prompt markers below and give it to any AI agent:*

```text
================================================================================
>>> START MASTER PROMPT: PRODUCTION ANDROID CONTACTS MANAGER <<<
================================================================================

You are a Principal Android Architect & Staff Android Engineer. Build a complete, production-grade Android Contacts Manager application intended for Google Play distribution and 10,000+ contact scale.

Do NOT create fake mock repositories, in-memory lists, or demo architectures. The application must work directly with Android's system Contacts Provider using ContentResolver and ContactsContract so that changes made by this app or external apps (like Google Contacts, WhatsApp, OEM apps) are instantly reflected bidirectionally.

--------------------------------------------------------------------------------
1. CORE ARCHITECTURAL RULES & MANDATES
--------------------------------------------------------------------------------
1. Android Contacts Provider is the SINGLE SOURCE OF TRUTH.
   - Do NOT duplicate or cache contacts into a local Room database.
   - All queries, mutations, and batch writes must run strictly on Dispatchers.IO via ContentResolver.
2. NO Fake RoleManager / Dialer Flow:
   - Do NOT request or assume fake ROLE_CONTACTS or ROLE_DIALER roles.
   - Never declare CALL_PHONE or SEND_SMS permissions. External actions must use standard safe Android intents (ACTION_DIAL, ACTION_SENDTO, ACTION_VIEW, ACTION_SEND) with ActivityNotFoundException fallbacks.
3. Privacy & Security:
   - Zero Contact PII logging: Never log contact names, phone numbers, emails, addresses, or notes to logcat.
   - Fully offline-first: No internet permission required for contacts.
4. Splash Screen API:
   - Use androidx.core.splashscreen.SplashScreen.
   - StartupCoordinator with StartupState (Loading, ContactsAccessRequired, Ready) to determine routing without artificial delays or blocking the main UI thread.

--------------------------------------------------------------------------------
2. 120Hz FAST-SCROLLING PERFORMANCE SECRETS (COMPOSE LAZYCOLUMN)
--------------------------------------------------------------------------------
To achieve locked 120 FPS scrolling without micro-stutters or frame drops:
1. Zero Regex / Precomputed Properties in ContactSummary:
   - NEVER call split("\\s+".toRegex()) inside getters or composables.
   - Precalculate initials, sortLetter, and stableKey in the constructor using primitive character indexing (indexOf(' ')).
2. Flattened List Hierarchy with Strict contentType:
   - Do NOT nest section headers conditionally inside contact item composables.
   - Pre-compute a flattened List<ContactListItem> on Dispatchers.Default with:
     * ContactListItem.Header(val letter: Char) -> key: "header_$letter", contentType: "header"
     * ContactListItem.Contact(val contact: ContactSummary) -> key: contact.stableKey, contentType: "contact"
3. Exact Fixed Height Constraints:
   - ContactRow must have a strict fixed height: Modifier.fillMaxWidth().height(58.dp).padding(horizontal = 20.dp).
   - AlphabetHeader must have a strict fixed height: Modifier.fillMaxWidth().height(32.dp).padding(horizontal = 20.dp).
   - This allows Compose's LazyLayout to compute prefetch coordinates in 0 microseconds during fast flings.
4. Zero-Layer GPU Clipping & BasicText:
   - Avoid Modifier.clip(CircleShape) on parent Box containers (which creates off-screen Skia saveLayer buffers). Use Modifier.background(backgroundColor, CircleShape).
   - Use BasicText with pre-cached static TextStyle in ContactRow and ContactAvatar to bypass MaterialTheme dynamic CompositionLocal lookups on every frame.
5. Coil In-Memory Hardware Bitmap Caching:
   - In Application class, configure ImageLoaderFactory with 25% RAM MemoryCache and 50MB DiskCache.
   - In avatar ImageRequest, pass allowHardware(true), crossfade(false), and memoryCacheKey(photoUri).
6. FastScroller Mutex Management:
   - In FastScroller, maintain a single scrollJob: Job? and cancel previous jobs before calling lazyListState.scrollToItem(targetIndex) to prevent Compose scroll mutex fighting.
   - Lock FastScroller width to exactly 28.dp so it never intercepts normal list flings.

--------------------------------------------------------------------------------
3. CONTACTS PROVIDER & QUERY DEDUPLICATION ENGINE
--------------------------------------------------------------------------------
1. Main List Lightweight Projection:
   Query only lightweight columns:
   - Contacts._ID, Contacts.LOOKUP_KEY, Contacts.DISPLAY_NAME_PRIMARY, Contacts.PHOTO_THUMBNAIL_URI, Contacts.STARRED, Contacts.HAS_PHONE_NUMBER.
2. Third-Party Sync Shadow Filtering (IN_VISIBLE_GROUP = 1):
   - WhatsApp, Telegram, and sync adapters create internal shadow contact records.
   - Query with Contacts.IN_VISIBLE_GROUP = 1 to hide shadow duplicates, matching Google Contacts.
   - Include automatic fallback to unfiltered queries if a device has no contact groups configured.
3. Multi-Account Phone & Email Deduplication:
   - When viewing contact details, aggregate raw contacts from Google, WhatsApp, SIM, etc.
   - Normalize phone numbers by stripping whitespace, hyphens, and parentheses to deduplicate "86258 16285" vs "8625816285".
   - Deduplicate emails case-insensitively and trim whitespace.
4. OEM Custom ROM Crash Prevention:
   - NEVER query internal non-standard columns like RawContacts.RAW_CONTACT_IS_READ_ONLY.
   - Always wrap detail queries in runCatching { ... }.getOrNull().
5. Atomic Batch Mutations:
   - All contact creations, updates, and deletes must execute atomically using ContentResolver.applyBatch(ContactsContract.AUTHORITY, ops) with back-references (withValueBackReference(Data.RAW_CONTACT_ID, 0)).

--------------------------------------------------------------------------------
4. REAL-TIME SYNCHRONIZATION ENGINE
--------------------------------------------------------------------------------
1. Multi-URI Registration:
   Register ContentObserver on:
   - content://com.android.contacts (ContactsContract.AUTHORITY)
   - ContactsContract.Contacts.CONTENT_URI
   - ContactsContract.Data.CONTENT_URI
   - ContactsContract.RawContacts.CONTENT_URI
2. Instant Foreground onResume() Trigger:
   In MainActivity.onResume(), immediately call appContainer.contactsRepository.refresh() to emit a sync signal without waiting for OS background alarms.
3. Conflated 100ms Debounce:
   Debounce raw ContentObserver events to 100ms for rapid, buttery-smooth updates.

--------------------------------------------------------------------------------
5. SCREEN-BY-SCREEN DETAILED SPECIFICATIONS
--------------------------------------------------------------------------------

### SCREEN 1: ContactsAccessSetupScreen (Permissions Onboarding)
- Visual illustration/icon for Contacts access.
- Headline & concise description explaining why Read/Write permissions are required.
- Clear Privacy Commitment card ("Your contacts stay completely on your device. We never upload or share your data").
- Primary Button: "Grant Access" (launches RequestMultiplePermissions for READ_CONTACTS and WRITE_CONTACTS).
- Denied / Permanently Denied State: Displays "Contacts permission is required to manage your contacts" with a button to "Open App Settings" via Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).

### SCREEN 2: MainScreen (Bottom Navigation Container)
- Bottom Navigation Bar with 4 tabs:
  1. Contacts (Icon: Contacts)
  2. Favorites (Icon: Favorite / Star)
  3. Fix & Manage (Icon: Build / Tools)
  4. Settings (Icon: Settings)

### SCREEN 3: ContactsScreen (Main Contact List)
- Top Bar: SearchBar with placeholder ("Search X contacts..."), search icon, and clear query button.
- Real-time debounced (200ms) search across contact names and phone number digits with coroutine cancellation (flatMapLatest).
- Main List: LazyColumn rendering flattened items (AlphabetHeader and ContactRow) with strict keys and contentTypes.
- Alphabet FastScroller on the right side:
  * 28dp touch strip showing 'A'..'Z' and '#'.
  * Touch & drag scrubbing with tactile haptic feedback (TextHandleMove).
  * Floating circular letter bubble indicator tracking the thumb during drag.
- Floating Action Button (FAB): "+" icon to open ContactEditorScreen in create mode (visible when write access is available).
- Empty States:
  * Search empty: "No matching contacts found for '<query>'".
  * List empty: "No contacts yet. Contacts added to your phone will appear here automatically."

### SCREEN 4: FavoritesScreen (Starred Contacts)
- Top Bar: "Favorites" title.
- Displays contacts where Contacts.STARRED == 1.
- Tapping a contact navigates directly to ContactDetailScreen.
- Empty State: Star outline icon with message "No favorite contacts. Star your favorite contacts from their details page to access them quickly here."

### SCREEN 5: ContactDetailScreen (Profile & Quick Actions)
- Top App Bar:
  * Back navigation arrow.
  * Star/Favorite toggle icon (filled gold star when starred, outline when not).
  * Edit icon button (navigates to ContactEditorScreen in edit mode).
  * Delete icon button with Confirmation Dialog ("Delete contact? This contact will be permanently deleted from your device").
- Header Section:
  * Large circular profile photo (or colorful avatar with initials fallback).
  * Contact display name (bold headline).
  * Account source badge: "Saved to <email/Google Account>" or "Saved to Phone/SIM".
- Quick Action Buttons (Row of circular FilledTonalIconButton with labels):
  1. Call (Icons.Default.Call) -> ACTION_DIAL with tel:<number>
  2. Text (Icons.AutoMirrored.Filled.Message) -> ACTION_SENDTO with smsto:<number>
  3. Email (Icons.Default.Email) -> ACTION_SENDTO with mailto:<email>
  4. Share (Icons.Default.Share) -> ACTION_SEND with contact vCard/text payload
  * Buttons are automatically disabled with dimmed styling if the contact lacks phone/email.
- Expandable Detail Cards:
  * Phone Numbers Card: List of numbers with type label (Mobile, Work, Home), call tap, SMS button, and Copy to clipboard button.
  * Email Addresses Card: List of emails with type label, email send button, and Copy button.
  * Postal Addresses Card: List of addresses with Map pin icon and ACTION_VIEW geo intent.
  * Organization Card: Company name, job title, and department.
  * Websites Card: List of URLs with browser launch intent.
  * Birthday / Events Card: Date display with calendar icon.
  * Notes Card: Full contact notes.

### SCREEN 6: ContactEditorScreen (Create & Edit)
- Top App Bar:
  * Close/Cancel 'X' button with Discard Changes confirmation dialog if fields are modified.
  * Title: "Create Contact" or "Edit Contact".
  * "Save" text button (validates that at least a name or phone number is provided).
- Account Destination Selector (for new contacts):
  * "Save to" dropdown card listing available device accounts (e.g. Google accounts vs Device/Phone storage).
- Photo Picker Section:
  * Large avatar with camera overlay button.
  * Uses zero-permission ActivityResultContracts.PickVisualMedia() for photo selection.
  * "Remove photo" option if a photo exists.
- Name Fields (OutlinedTextFields):
  * First name, Middle name, Last name.
  * Expandable "More name fields" button revealing Name prefix (Dr., Mr.) and Name suffix (Jr., III).
- Dynamic Multi-Phone Section:
  * List of phone number rows with number input and Type dropdown (Mobile, Home, Work, Main, Other, Custom).
  * Delete row button for each entry.
  * "+ Add phone" text button to add additional numbers.
- Dynamic Multi-Email Section:
  * List of email rows with email input and Type dropdown (Home, Work, Other, Custom).
  * Delete row button for each entry.
  * "+ Add email" text button.
- Organization Section: Company name, Job title, Department.
- Address Section: Street address with Type dropdown.
- Websites Section: "+ Add website" with URL inputs.
- Significant Date / Birthday Section: Date input.
- Notes Section: Multi-line note input.

### SCREEN 7: FixAndManageScreen (Health Metrics & Tools)
- Top App Bar: "Fix & Manage".
- Cleanup & Maintenance Card:
  * "Merge duplicates": Combined scanner extension point.
  * "Contacts without phone numbers": Live metric badge showing exact count.
  * "Contacts without names": Live metric badge showing exact count.
- Import & Export Card:
  * "Export to .vcf file": Save all contacts to a standard vCard file.
  * "Import from .vcf file": Restore contacts from a vCard backup.
  * "Backup & Restore": Manage device backups.
- Organization Tools Card:
  * "Contact Groups & Labels": Extension point for ContactsContract.Groups.

### SCREEN 8: SettingsScreen (DataStore Preferences)
- Top App Bar: "Settings".
- Display & Theme Section:
  * Theme Mode Selector: Radio dialog / dropdown for "System default", "Light", "Dark".
  * Dynamic Color Toggle (Material You): Switch enabling/disabling dynamic wallpaper colors on Android 12+ (API 31+).
- Contact List Preferences Section:
  * Sort by: "First name" vs "Last name".
  * Name format: "First name first" (John Doe) vs "Last name first" (Doe, John).
  * Default Account for New Contacts: Selector for Google account vs Device storage.
- Privacy & Information Card:
  * App version, Single Source of Truth architecture description, and zero-data-collection privacy guarantee.

--------------------------------------------------------------------------------
6. DEPENDENCY INJECTION & MONETIZATION BOUNDARY
--------------------------------------------------------------------------------
1. AppContainer:
   - Provide a clean, pure Kotlin DI container (AppContainer / DefaultAppContainer) initialized in ContactsApplication.
   - Holds singletons: CoroutineDispatchers, PermissionManager, ContactsObserver, PreferencesRepository, ContactsDataSource, ContactsRepository, UseCases.
2. Monetization Boundary:
   - Keep future monetization (AdMob, In-App Purchases, Subscriptions) strictly isolated behind a MonetizationBoundary / MonetizationManager presentation wrapper.
   - Contact domain and data layers must never reference or depend on ads or billing code.

--------------------------------------------------------------------------------
7. VERIFICATION & TESTING REQUIREMENTS
--------------------------------------------------------------------------------
1. Unit Tests:
   - AlphabetSectionTest: Benchmarks grouping 10,000 items in < 50ms and 20,000 items in < 100ms.
   - ContactSummaryModelTest: Verifies initials extraction and sort letter logic without regex.
   - ContactsAccessStateTest: Verifies capability flags (canRead, canWrite, isRestricted).
   - StartupCoordinatorTest: Verifies splash state transitions.
   - ContactsRepositoryTest: Verifies repository operations and observer triggers.
   - DeduplicationTest: Verifies phone number formatting normalization and deduplication.
2. Release Build:
   - Run ./gradlew testDebugUnitTest assembleDebug assembleRelease.
   - Must pass all tests and compile cleanly with R8 optimization enabled (0 errors).

================================================================================
>>> END MASTER PROMPT: PRODUCTION ANDROID CONTACTS MANAGER <<<
================================================================================
```
