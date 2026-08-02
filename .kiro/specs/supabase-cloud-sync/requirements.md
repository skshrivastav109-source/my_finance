# Requirements Document: Supabase Cloud Sync

## Introduction

The Personal Finance Dashboard currently uses browser-isolated localStorage for data persistence. This limitation prevents users from accessing their financial data across multiple browsers or devices. This feature introduces Supabase-powered cloud synchronization to enable cross-browser account access, real-time data synchronization, and persistent cloud storage while maintaining offline-first functionality and the existing 5-minute session timeout behavior.

The feature preserves existing localStorage data during migration, handles network disconnections gracefully, and maintains session isolation across browser tabs through Supabase's real-time database capabilities.

---

## Glossary

- **Supabase**: Cloud backend service providing PostgreSQL database, real-time APIs, and authentication
- **Cloud_Storage**: Server-side persistent data store in Supabase PostgreSQL
- **Local_Storage**: Browser-side IndexedDB or localStorage for offline-first caching
- **Sync_Engine**: Client-side process that synchronizes data between Local_Storage and Cloud_Storage
- **Real-Time_Sync**: Live data updates pushed from Cloud_Storage to connected clients
- **User_Account**: Unique identity tied to email, username, and Supabase auth UID
- **Session**: Authenticated state for a single browser tab, expires after 5 minutes of inactivity
- **Data_Consistency**: State where Local_Storage matches Cloud_Storage (within latency bounds)
- **Conflict_Resolution**: Process for handling divergent edits to the same data record
- **OTP**: One-time password sent to email for authentication verification
- **Migration**: Process of uploading existing localStorage data to Cloud_Storage during first login
- **Offline_Mode**: Client-side operation when network is unavailable; queued for sync when online
- **Cross-Browser_Access**: Ability to retrieve the same User_Account's data in different browsers

---

## Requirements

### Requirement 1: User Authentication with Supabase

**User Story:** As a user, I want to authenticate using my email and OTP, so that I can securely create a cloud-backed account accessible across browsers.

#### Acceptance Criteria

1. WHEN a user submits the signup form with name, email, username, and password, THE Auth_Service SHALL validate the email format and check username uniqueness against Supabase
2. WHEN the email is valid and username is unique, THE Auth_Service SHALL generate a 6-digit OTP, send it to the user's email, and display the OTP verification form
3. WHEN the user enters a valid OTP within 10 minutes AND password hashing succeeds, THE Auth_Service SHALL create a Supabase user account, store user metadata (name, username, email, createdAt) in the users table, and initiate data migration from localStorage to Cloud_Storage; IF password hashing fails or OTP is expired, THEN THE Auth_Service SHALL reject account creation and display an error
4. IF the OTP is invalid or expired, THEN THE Auth_Service SHALL display an error message and allow the user to resend the OTP
5. WHEN a user submits the login form with username and password, THE Auth_Service SHALL authenticate via Supabase, verify the password hash, and establish a session with a 5-minute inactivity timeout
6. IF login credentials are incorrect, THEN THE Auth_Service SHALL display "Invalid username or password" and remain on the login screen
7. WHERE an email-based social login is planned, THE Auth_Service SHALL support Supabase OAuth providers in the future without breaking existing OTP flow

---

### Requirement 2: Data Migration from localStorage to Cloud_Storage

**User Story:** As an existing user, I want my historical data to automatically migrate to the cloud, so that I don't lose my transaction history when switching to cloud sync.

#### Acceptance Criteria

1. WHEN a user completes OTP verification for the first time, THE Migration_Service SHALL read all data from the user's localStorage (income, expenses, loans, budgets, transfers, bank accounts)
2. WHEN localStorage contains valid user data, THE Migration_Service SHALL upload all records to Cloud_Storage in the users/{userId}/data/ namespace, preserving original timestamps and record IDs
3. WHEN migration completes successfully, THE Migration_Service SHALL clear the old localStorage keys and display a success toast "Data migrated to cloud"
4. IF a network error occurs during migration, THE Migration_Service SHALL retry up to 3 times with exponential backoff (1s, 2s, 4s); if all 3 attempts fail, display "Migration failed; data saved locally, will retry later"
5. WHEN migration is in progress, THE App SHALL prevent user navigation and display a loading indicator with "Syncing your data..."
6. IF the user closes the app during migration, THE App SHALL resume migration on next login, checking for incomplete uploads and completing them

---

### Requirement 3: Real-Time Cloud Synchronization

**User Story:** As a user, I want my data to sync automatically across all my browser tabs and devices, so that I always see the latest financial information.

#### Acceptance Criteria

1. WHEN a user creates, updates, or deletes a transaction (income/expense/loan/budget/transfer), THE Sync_Engine SHALL immediately write to Local_Storage and queue the change for Cloud_Storage upload
2. WHEN the network is available, THE Sync_Engine SHALL upload the queued change to Cloud_Storage within 2 seconds and receive a server timestamp
3. WHEN Cloud_Storage confirms the write, THE Sync_Engine SHALL update the local record with the server timestamp and mark it as synced
4. WHEN data changes in Cloud_Storage by another user session or device, THE Real_Time_Listener SHALL receive the update via Supabase real-time subscriptions and merge it into Local_Storage without user interruption
5. WHEN multiple tabs/devices edit the same record simultaneously, THE Conflict_Resolver SHALL apply last-write-wins using server timestamps, with the winning version pushed to all clients
6. IF a local edit conflicts with a server version, THE Sync_Engine SHALL display a toast "Conflict resolved: latest version loaded" and show the server version
7. WHEN the network is offline, THE Sync_Engine SHALL queue all writes locally and display a "Offline mode" indicator in the UI

---

### Requirement 4: Offline-First Local Caching with Eventual Consistency

**User Story:** As a user, I want to continue using the app when offline and have my changes sync when I reconnect, so that network unavailability doesn't block my work.

#### Acceptance Criteria

1. WHEN the network is offline, THE Local_Cache SHALL remain functional for read and write operations using only Local_Storage/IndexedDB
2. WHEN a user creates or modifies data while offline, THE Local_Cache SHALL queue the change with a pending status and a local timestamp
3. WHEN the network reconnects, THE Sync_Engine SHALL detect the change by monitoring navigator.onLine and trigger a sync attempt
4. WHEN syncing pending changes, THE Sync_Engine SHALL upload all queued writes in order, applying server timestamps and conflict resolution
5. IF a queued write fails after 5 retry attempts, THE Sync_Engine SHALL display "Sync failed for [transaction]; will retry later" and keep the item queued
6. WHERE a user's session expires while offline, THE App SHALL keep the pending queue intact so it syncs after re-login
7. WHEN all pending changes have synced, THE Sync_Engine SHALL display a toast "All changes synced" and clear the pending indicator

---

### Requirement 5: Session Management with 5-Minute Inactivity Timeout

**User Story:** As a user concerned about security, I want my session to auto-logout after 5 minutes of inactivity, so that my account is protected if I leave my browser unattended.

#### Acceptance Criteria

1. WHEN a user logs in, THE Session_Manager SHALL initialize a 5-minute inactivity timer and reset it on user activity (click, keypress, scroll, touch)
2. WHEN 5 minutes elapse without user activity, THE Session_Manager SHALL clear the Supabase auth session, flush pending sync queue to localStorage, and redirect to the login screen with the message "Session expired due to inactivity. Please login again.", regardless of offline or online status
3. WHEN a user switches to a different tab and then returns, THE Session_Manager SHALL verify the session is still valid and resume sync operations
4. WHEN a user is in offline mode and the session expires, THE Session_Manager SHALL preserve the pending queue in localStorage, clear the user state on screen, and enable queue recovery on next login
5. WHERE a session has multiple browser tabs open, THE Session_Manager SHALL timeout independently per tab, not affecting other tabs
6. WHEN a user manually logs out, THE Session_Manager SHALL clear the session, sign out of Supabase, and display the login screen

---

### Requirement 6: Cross-Browser Account Access and Data Retrieval

**User Story:** As a user, I want to log in with the same credentials in any browser and see all my data, so that I'm not locked to a single browser.

#### Acceptance Criteria

1. WHEN a user logs in to a new browser, THE Auth_Service SHALL authenticate via Supabase and retrieve the user's full dataset (income, expenses, loans, budgets, transfers, bank accounts) from Cloud_Storage
2. WHEN retrieving data, THE Data_Loader SHALL populate Local_Storage with the cloud data, enabling offline-first functionality in the new browser
3. WHEN the cloud dataset is large (>1000 records), THE Data_Loader SHALL load data in pages (500 records per page) and display a loading indicator "Loading your financial data..."
4. IF a user modifies data in Browser A, THE Real_Time_Listener in Browser B SHALL receive the update via Supabase real-time subscriptions and refresh the UI without requiring a manual refresh
5. WHEN a user logs out in Browser A, THE data remains in Cloud_Storage; logging in with Browser B retrieves the same data, confirming data persistence across browsers
6. WHERE a user's data grows over time, THE Data_Loader SHALL efficiently retrieve only new/modified records using last-sync timestamps to minimize bandwidth

---

### Requirement 7: Reliable Sync Queue with Retry Logic

**User Story:** As a user, I want my transactions to reliably upload to the cloud even with network hiccups, so that I don't lose data due to temporary connectivity issues.

#### Acceptance Criteria

1. WHEN a write operation fails, THE Sync_Queue SHALL retry automatically with exponential backoff (500ms, 1s, 2s, 4s, 8s) up to 5 times
2. WHEN all retries are exhausted, THE Sync_Queue SHALL persist the failed item in a failures table in Local_Storage, display a notification with retry option, and keep the item visible in the UI with a "sync failed" indicator
3. WHEN the user clicks "Retry now", THE Sync_Queue SHALL immediately attempt to upload the failed item
4. WHEN a queued item is successfully uploaded AND removed from queue, THE Sync_Queue SHALL update the record with the server timestamp, mark it as synced, and clear all failure indicators
5. WHEN the app restarts, THE Sync_Queue SHALL load any pending or failed items from Local_Storage and resume syncing where it left off
6. IF the same record is edited while it's pending upload, THE Sync_Queue SHALL replace the pending version with the new edit and maintain the queue position

---

### Requirement 8: Real-Time Subscriptions for Multi-Client Updates

**User Story:** As a user with multiple devices, I want to see updates made on one device appear instantly on another, so that my financial dashboard is always current across all my devices.

#### Acceptance Criteria

1. WHEN a user logs in, THE Real_Time_Listener SHALL subscribe to all user data channels in Cloud_Storage (users/{userId}/income, users/{userId}/expenses, etc.) using Supabase real-time subscriptions
2. WHEN another client creates, updates, or deletes a record in Cloud_Storage, THE Real_Time_Listener SHALL receive the change event and merge it into Local_Storage
3. WHEN a received update is a modification to an existing record AND there are local pending changes, THE Real_Time_Listener SHALL check for conflicts and apply Conflict_Resolver logic; WHEN there are no pending changes, THE Real_Time_Listener SHALL apply the received update without conflict checking
4. WHEN Real_Time_Sync delivers an update, THE UI SHALL refresh the affected view (e.g., Income table, Dashboard totals) without requiring manual refresh
5. IF the real-time connection is lost, THE Real_Time_Listener SHALL attempt to reconnect using exponential backoff and display "Reconnecting..." if disconnected for >5 seconds
6. WHEN the real-time connection is restored, THE Real_Time_Listener SHALL fetch any missed updates from Cloud_Storage using a since-timestamp query to catch up
7. WHERE a user has multiple tabs open, THE Real_Time_Listener SHALL deduplicate events across tabs using a shared worker or localStorage polling to avoid redundant renders

---

### Requirement 9: Data Schema in Supabase Cloud_Storage

**User Story:** As a developer, I want a well-defined data schema in Supabase, so that data is organized, queryable, and supports future features like analytics and data export.

#### Acceptance Criteria

1. THE Cloud_Storage SHALL maintain the following tables: users, user_banks, user_incomes, user_expenses, user_loans, user_budgets, user_transfers, with each table scoped to user_id for row-level security
2. WHEN a user creates a record, THE Backend_Service SHALL automatically populate created_at (server timestamp), updated_at, and mark synced=true
3. WHEN a user updates a record, THE Backend_Service SHALL update updated_at with server timestamp and increment a version field for conflict tracking
4. WHERE Supabase policies are enforced, THE Backend_Service SHALL enforce row-level security so users can only read/write their own data; this allows asymmetric permissions where users might write data they cannot read back
5. THE Cloud_Storage schema SHALL support future features: data export (CSV/JSON), analytics queries, and cross-user transfers (with permissions)

---

### Requirement 10: Handling Duplicate Records During Sync

**User Story:** As a user, I want my data to remain consistent, so that duplicate transactions don't appear when syncing across browsers.

#### Acceptance Criteria

1. WHEN a record is uploaded to Cloud_Storage, THE Upload_Service SHALL include a unique client-generated ID (uuid) and check for duplicates using this ID
2. IF a record with the same client ID already exists in Cloud_Storage, THE Upload_Service SHALL skip the upload and mark the local copy as synced; if duplicate detection fails, allow the upload to proceed to prevent data loss
3. WHEN data is synced from Cloud_Storage to Local_Storage, THE Sync_Engine SHALL use record IDs as merge keys and skip records already present with the same ID
4. WHEN the user logs in to a new browser, THE Data_Loader SHALL match records from localStorage (with client IDs) to Cloud_Storage records; if matches are found, skip re-uploading them

---

### Requirement 11: Logout and Session Cleanup

**User Story:** As a user, I want to securely log out, so that my session is properly closed and my data is protected.

#### Acceptance Criteria

1. WHEN a user clicks logout, THE Auth_Service SHALL initiate logout; if any cleanup operation (Supabase sign-out, token clearing) fails, THE Auth_Service SHALL block logout completion and keep the user in their current session while displaying an error message
2. WHEN logout is complete, THE App SHALL display the login screen with no cached user data visible
3. WHERE a user has pending sync items when logging out, THE Sync_Engine SHALL flush them to localStorage for recovery on next login
4. WHEN a user logs back in, THE App SHALL reconstruct the session and retrieve the latest Cloud_Storage data; pending items from the previous session shall be synced

---

### Requirement 12: Error Handling and User Feedback

**User Story:** As a user, I want clear feedback when something goes wrong (network errors, sync failures, conflicts), so that I can take action and understand the app's state.

#### Acceptance Criteria

1. WHEN a network error occurs, THE Error_Handler SHALL display a toast with the error message (e.g., "Network error: failed to sync. Will retry automatically.")
2. WHEN a critical error prevents the app from functioning, THE Error_Handler SHALL display a modal with the error, a description of the issue, and an "OK" or "Retry" button
3. WHEN a critical error and an auth error occur simultaneously, THE Error_Handler SHALL display the critical error modal (taking priority) rather than redirecting to login
4. WHEN a data conflict is detected, THE Error_Handler SHALL notify the user: "Conflict resolved: your local version was replaced by the latest cloud version" and show what changed (optional: provide undo option)
5. WHEN sync queue reaches >10 failed items, THE Error_Handler SHALL display a warning banner "Sync issues detected; some data may not be saved to cloud" and provide a "Retry All" button
6. WHERE a user is offline, THE Topbar SHALL display a persistent indicator (e.g., "📡 Offline mode") only when network is unavailable (not when pending items exist but network is online)

---

### Requirement 13: Performance and Optimization

**User Story:** As a user, I want the app to remain responsive even with large datasets, so that I don't experience lag when managing thousands of transactions.

#### Acceptance Criteria

1. WHEN a user has >5000 records, THE Data_Loader SHALL use pagination AND lazy loading together (500 records per page); if either pagination or lazy loading fails, disable both mechanisms to maintain data integrity
2. WHEN sync operations occur, THE Sync_Engine SHALL batch writes to Cloud_Storage (max 50 records per request) to reduce network overhead and server load; this applies both online and offline, with batches queued for transmission when connectivity is available
3. WHEN real-time updates are received, THE UI SHALL debounce updates for the same view (e.g., Income table) to prevent excessive re-renders; coalesce updates over 100ms windows
4. WHERE IndexedDB is used for Local_Storage, THE Cache_Store SHALL index records by user_id, date, and category to enable fast queries (e.g., "all expenses in January 2024")
5. WHEN a user is offline, THE App SHALL not attempt network calls; checks for network availability SHALL use navigator.onLine and be cached locally with a 5-second TTL

---

### Requirement 14: Data Export and Future Extensibility

**User Story:** As a user, I want to export my financial data, so that I can backup or use my data in other tools.

#### Acceptance Criteria

1. WHERE a data export feature is planned, THE Cloud_Storage schema SHALL support efficient queries by date, category, and amount for export operations
2. WHEN this feature is implemented, THE Export_Service SHALL retrieve data from Cloud_Storage and format it as JSON or CSV without blocking the UI
3. THE Cloud_Storage schema SHALL maintain backward compatibility when initially created and immediately, preserving support for future features (budgeting recommendations, analytics) to query historical data; if export feature implementation requires schema changes that break compatibility, the export feature SHALL be implemented even with breaking changes

---

### Requirement 15: Testing Data Consistency and Sync Properties

**User Story:** As a developer, I want to ensure data consistency across all scenarios, so that the sync system is reliable and trustworthy.

#### Acceptance Criteria

1. FOR ALL data that is uploaded to Cloud_Storage and then downloaded, the record SHALL have identical content (round-trip property: upload(data) → download → data should equal original, with server timestamps added)
2. WHEN data is synced across multiple browsers and edited sequentially, THE final state in Cloud_Storage SHALL reflect the last edit's timestamp, ensuring no data loss or ordering issues
3. WHEN a user has queued edits and then logs in to a different browser, THE queued edits from the first browser SHALL sync eventually to Cloud_Storage, and the second browser SHALL retrieve the combined state
4. WHEN the database is queried for a user's records, ALL records created by that user SHALL be present in the result, confirming no records are lost during sync (idempotency property: multiple syncs of the same batch result in the same state)

---

## Non-Functional Requirements

### Security
- All communication with Supabase SHALL use HTTPS/TLS encryption
- Passwords SHALL be hashed client-side before transmission (existing hashPassword function)
- Supabase row-level security policies SHALL prevent users from accessing other users' data
- OTP tokens SHALL expire after 10 minutes and be single-use
- Session tokens SHALL be stored securely in Supabase and invalidated on logout

### Reliability
- Sync operations SHALL be resilient to network interruptions and support automatic retry with exponential backoff
- Pending sync items SHALL be persisted locally and survive app restarts
- Real-time subscriptions SHALL automatically reconnect on network restoration
- Conflict resolution SHALL guarantee eventual consistency across all clients

### Usability
- The user SHALL not need to manually trigger sync; all operations SHALL be automatic
- Sync status (online/offline/syncing/failed) SHALL be clearly visible in the UI
- Error messages SHALL be in plain language and actionable

### Performance
- Initial data load for up to 5000 records SHALL complete within 3 seconds
- Individual write operations (create/update transaction) SHALL complete within 500ms on local storage, with cloud sync within 2 seconds on good network
- Real-time updates from other devices SHALL appear within 1 second of being written to Cloud_Storage
- The app SHALL remain responsive even during large batch syncs (pagination and batching)

### Compatibility
- The feature SHALL be backward compatible with existing localStorage data
- The app SHALL continue to support the 5-minute session timeout without modification
- The feature SHALL not require changes to the existing UI or user workflows

---

## Acceptance Criteria Summary

| # | Area | Key Test | 
|---|------|----------|
| 1 | Auth | OTP signup creates Supabase user and migrates data |
| 2 | Migration | localStorage data is uploaded to cloud on first login |
| 3 | Sync | Changes sync to cloud within 2s and appear on other browsers in <1s |
| 4 | Offline | App works offline; pending changes sync when reconnected |
| 5 | Session | 5-min inactivity timeout clears session; pending queue preserved |
| 6 | Cross-browser | Login in new browser retrieves same data from cloud |
| 7 | Retry | Failed syncs auto-retry with exponential backoff |
| 8 | Real-time | Multi-device edits merge without data loss |
| 9 | Schema | Data organized by user_id in Supabase; RLS enforced |
| 10 | Dedup | Duplicate records are detected and skipped |
| 11 | Logout | Session cleared; pending items flushed to localStorage |
| 12 | Errors | Clear error messages for network, auth, conflict scenarios |
| 13 | Performance | Large datasets load progressively; syncs batched |
| 14 | Export | Schema supports future export features |
| 15 | Consistency | Data round-trips intact; idempotent syncs; eventual consistency |

