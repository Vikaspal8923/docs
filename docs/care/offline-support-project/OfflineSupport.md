# Care Enhancement Proposal (CEP): Offline Support Implementation

## Overview

This proposal details the implementation of offline capabilities in CARE using **TanStack Query with IndexedDB persistence**. The solution enables healthcare workers to perform critical operations during internet outages with automatic synchronization when connectivity is restored.

We evaluated multiple methods to achieve this functionality. Information about these alternative approaches is documented in:  
[Additional Approaches Audit](AdditionalInfo.md)

The current proposal focuses specifically on our selected production-ready solution. Comprehensive implementation details for the chosen approach will be maintained in this document.

The implementation will focus on supporting specific critical workflows that require offline functionality. All workflows designated for offline support are documented in: [Additional Info Audit](AdditionalInfo.md)

## Implementation Phases

1. **[Phase 1: Caching ](#phase-1-caching)**
2. **[Phase 2: Offline Writes](#phase-2-offline-writes)**
3. **[Phase 3: Synchronization](#phase-3-synchronization)**
4. **[Phase 4: Notifications](#phase-4-notifications)**

### Phase 1: Caching

#### What Are We Caching?

- **API Responses:** We cache the results of API calls for workflows that require offline support (such as patient data, forms, and other critical resources). Only endpoints explicitly marked for offline use (`meta.persist: true`) are cached.
- **Selective Data:** Not all API responses are cached—only those relevant to offline workflows, as determined by the application's business logic and user needs.
- **Granularity:** Data is cached per `queryKey`, ensuring that only the specific data a user has accessed is available offline. For example, if a user views Patient A's record, only that record is cached for offline use.

#### Why Do We Need Caching?

- **Offline Access:** Caching enables users to access previously viewed data even when there is no internet connection, ensuring continuity of care and workflow.
- **Performance:** By serving data from the local cache, the application reduces redundant network requests and improves data retrieval speed.
- **User Experience:** Users can continue to view and interact with critical information during connectivity interruptions, minimizing workflow disruptions.
- **Reliability:** Caching prevents data loss and enables seamless transitions between online and offline states, which is crucial for healthcare environments where connectivity may be unreliable.

**Note**: Configuration details needed for caching using tanstack query are 
available in [Additional Approaches Audit](AdditionalInfo.md). This section focuses on their 
technical implementation.

#### How Are We Caching?

- **Technology Stack:**
  - **TanStack Query (React Query):** Used for data fetching, caching, and state management.
  - **IndexedDB (via Dexie.js):** Provides persistent local storage for cached data, allowing it to survive page reloads and browser restarts.
- **Custom Persister:**
  - A custom persister is implemented using Dexie.js to store and retrieve TanStack Query's cache from IndexedDB.
  - The persister implements three core methods: `persistClient` (save cache), `restoreClient` (load cache), and `removeClient` (clear cache on logout).
- **Configuration:**
  - Only queries with `meta.persist: true` are persisted to IndexedDB, ensuring that only essential data is stored.
  - Cached data is stored with a configurable expiration (`maxAge`), and in-memory cache is managed with `gcTime`.
  - The application is wrapped in `PersistQueryClientProvider` to enable persistence and manage cache lifecycle.
- **Cache Invalidation:**
  - Cached data is cleared on logout or session expiration to prevent data leakage between users and ensure privacy.

#### How Does Caching Help in Offline Support?

- **Data Availability:** Users can access cached data for supported workflows even when offline, allowing them to continue working without interruption.
- **Seamless Experience:** When the app detects offline status, it serves data from the cache instead of making network requests, providing a smooth user experience.
- **Automatic Updates:** When connectivity is restored, fresh data is fetched from the server and the cache is updated automatically.
- **Security:** Cache is user-isolated and cleared on logout, ensuring that data from one user is not accessible to another.

#### Practical Example

Suppose a healthcare worker accesses a patient's record while online. The data is fetched from the server and cached locally. If the worker later loses connectivity, they can still view the cached patient record. When connectivity returns, the app fetches the latest data and updates the cache.

#### Additional Details and Limitations

- **Only Accessed Data is Cached:** Data must be accessed at least once while online to be available offline. New or unvisited records will not be available until accessed online.
- **Selective Caching:** Mark only essential queries with `meta.persist: true` to avoid excessive storage usage and ensure optimal performance.
- **Network Detection Caveats:** TanStack Query’s network detection may sometimes give false positives (e.g., connected to Wi-Fi without internet). This can affect when cached data is served versus when a network request is attempted.
- **Session Isolation:** Cache is cleared on logout to prevent data mixing between users, which is especially important on shared devices.

#### Technical Implementation Summary

- **Dexie.js Schema:**
  - A table named `queryCache` is used to store cached query data, keyed by a unique cache key and timestamped for expiration management.
- **PersistQueryClientProvider:**
  - The app is wrapped in this provider, which manages the persistence and hydration of the query cache.
- **Query Configuration:**
  - Queries that should be cached for offline use are configured with `meta: { persist: true }` and `networkMode: "online"`.
  - Example:

```typescript
const { data: user, isLoading } = useQuery({
  queryKey: ["currentUser", accessToken],
  queryFn: query(routes.currentUser, { silent: true }),
  networkMode: "online",
  retry: false,
  enabled: !!localStorage.getItem(LocalStorageKeys.accessToken),
  meta: { persist: true },
});
```

- **Cache Lifecycle:**
  - Data is cached when fetched online, served from cache when offline, and updated when connectivity is restored.
  - Cache is cleared on logout or session expiration.

This caching strategy forms the foundation for robust offline support, ensuring that users have access to critical data regardless of network conditions, while maintaining security, performance, and data integrity.

### Phase 2: Offline Writes

This phase implements saving form data to IndexedDB (via Dexie.js) when offline, for later synchronization when connectivity returns.

**Implementation Flow**:

1. On form submission, check network status
2. If offline:
   - Save form data to IndexedDB
   - **Normalize the form payload to match the API response structure**
   - **Update the cache using setQueryData to reflect the new/updated record**
   - **Update all relevant cached API responses that relate to this form (e.g., detail and list endpoints)**
   - Queue for later synchronization
3. If online:
   - Process normally via `useMutation`

#### Normalizing and Caching Offline Writes

After saving a record offline, it is important to ensure the local cache reflects the new or updated data, so the user experience remains consistent and the UI shows the latest state—even while offline. This is achieved by:

- **Normalizing the Payload:** The form data is transformed to match the structure of the API response (as if it had come from the backend). This ensures consistency between online and offline data representations.
- **Updating the Cache:**
  - Use TanStack Query's `setQueryData` to update the cache for the relevant query key(s) with the normalized data.
  - Update not only the detail endpoint (e.g., the specific record) but also any related list endpoints (e.g., the list of all encounters) to ensure the new or updated record appears in all relevant places in the UI.

**Example: Creating an Encounter Offline**

Suppose a user creates a new encounter while offline:

1. The form payload is saved to IndexedDB as an offline write.
2. The payload is normalized to match the encounter API response structure (e.g., adding default fields, IDs, or computed values as needed).
3. The cache is updated:
   - `setQueryData(["encounter", newEncounterId], normalizedEncounter)` updates the cache for the encounter detail endpoint.
   - `setQueryData(["encounter-list", patientId], prevList => [...prevList, normalizedEncounter])` updates the cache for the encounter list endpoint, ensuring the new encounter appears in the list view.

This approach ensures that after creating an encounter offline, the user immediately sees the new encounter in both the detail and list views, providing a seamless and consistent experience.

#### Additional Details on Offline Write Handling

- **Form Validation:** All forms are validated client-side before saving to IndexedDB. Users are notified immediately if any required fields are missing or invalid, ensuring only valid data is stored for later sync.
- **Temporary IDs:** When creating new records offline (such as a new encounter or patient), temporary UUIDs are generated. These IDs are used to maintain relationships between records (e.g., linking a new encounter to a new patient) until real IDs are assigned after successful sync with the backend.
- **Payload Normalization:** The form data is normalized to match the API response structure. This ensures that the UI can display the new or updated record as if it had come from the backend, maintaining consistency between online and offline states.
- **Immediate Cache Update:** After saving the offline write, the cache is updated using TanStack Query's `setQueryData` for both detail and list endpoints. This allows the UI to reflect the new or updated record instantly, even before it is synced to the backend.
- **Atomicity and Consistency:** The offline write and cache update are performed together to prevent UI inconsistencies. If the cache update fails, the system can roll back the IndexedDB write or notify the user of the issue.
- **Metadata Storage:** Each offline write includes metadata such as timestamp, operation type (create/update), and the target route. This metadata is useful for later synchronization, debugging, and providing UI indicators (such as "pending" badges).
- **Developer Extensibility:** Developers can hook into the offline write process to trigger custom logic, such as analytics, local notifications, or additional cache updates, making the system flexible and extensible.
- **Example UI Feedback:**
  - "Your data has been saved locally and will be synced when you’re back online."
  - "This record is pending sync."

#### 1. Dexie.js Table Schema

so first see the table schema and then we will discusse about each entrie of the table schema. we will use same indexdDB database instant we use for caching. W just create another Table in it for offline writes.

```ts
offlineWrites!: Dexie.Table<
    {
      id: string;
      userId: string;
      syncrouteKey: string;
      type?:string;
      resourceType?: string;
      pathParams?: Record<string, any>;
      payload: unknown;
      response?: unknown;
      parentMutationIds?: string[]; 
      clientTimestamp: number;
      serverTimestamp?: string;
      lastAttemptAt?: number;
      syncStatus: "pending" | "success" | "failed" | "conflict";
      lastError?: string;
      retries?: number;
      conflictData?: unknown;
      queryrouteKey?: string;
      queryParams?: Record<string, any>;
    },
    string
  >;
 constructor() {
   this.version(2).stores({
      queryCache: "cacheKey, timestamp",
      offlineWrites: "id, userId,  timestamp",
    });
 }
```

Below is a concise description of each field in the offlineWrites Dexie table :-

1. **`id: string`** :- Unique identifier for this offline‑write record (usually a UUID).
2. **`userId: string`** :- The ID of the current user (e.g. `user.external_id`) who initiated this write. Help to prevent syncing data of one user by another user if there is same device.
3. **`syncrouteKey: string`** :- Key/name of the mutation route to replay when syncing (e.g. `"updatePatient"`), allowing reuse of existing mutation functions.
4. **`resourceType?: string`** :- Human‑readable tag for the type of resource being written (e.g. `"patient"`, `"form"`), useful for grouping or logging.
5. **`pathParams?: Record<string, any>`** :- Route parameters (e.g. `{ id: patientId }`) required by the `syncrouteKey` mutation, so the same API function signature can be called offline.
6. **`payload: unknown`** :- The actual data object (e.g. updated form fields) that will be sent to the server when back online.
7. **`clientTimestamp: number`** :- Timestamp (e.g. `Date.now()`) when the user saved this record offline—used to order writes or detect staleness.
8. **`serverTimestamp?: string`** :- Optional server‑provided timestamp of the last known server version (e.g. `patientQuery.data.modified_date`) for conflict detection at frontend level.
9. **`lastAttemptAt?: number`** :- Timestamp of the last time you tried syncing this record—used to implement retry/backoff logic.
10. **`syncStatus: "pending" | "success" | "failed" | "conflict"`** :- Current sync state:
    - `"pending"` = not yet retried
    - `"success"` = synced successfully
    - `"failed"` = last sync attempt errored
    - `"conflict"` = server data has diverged from local payload
11. **`lastError?: string`** :- Error message (e.g. HTTP status or validation error) from the most recent sync attempt, for debugging or user notification.
12. **`retries?: number`** :- Number of times this record has been retried so far—used to limit retry attempts or escalate conflicts.
13. **`conflictData?: unknown`** :- If a conflict is detected (e.g. server has newer data), this holds the server’s latest version so you can show a merge UI.
14. **`queryrouteKey?: string`** :- Key/name of the “fetch” route (e.g. `"getPatient"`) that can be called before syncing, in order to retrieve the current server state for conflict checks.
15. **`queryParams?: Record<string, any>`** :- Route parameters (e.g. `{ id: patientId }`) required by `queryrouteKey` to fetch the current form/patient data before submitting the offline write.
16. **`parentMutationIds`** :- array of parent mutations id It help us during syncing of parent-child  dependent mutations.
17. **`response`** :- store response after successful sync.
18. **`type`** :-  store the the type of mutation like 'createpatient,'updatepatient'


#### 2. **SaveofflineWrites** function

We have a generalized `saveOfflineWrites` function that should be called when submitting form data while offline. We will pass all necessary information to it so that it can save the data locally in IndexedDB.

```ts
export const saveOfflineWrite = async ({
  userId,
  syncrouteKey,
  payload,
  pathParams,
  type,
  resourceType,
  serverTimestamp,
  queryParams,
  queryrouteKey,
  parentMutationIds 
  dependentFields,
}: SaveOfflineWriteParams) => {
  const writeEntry = {
    id,
    userId,
    syncrouteKey,
    payload,
    pathParams,
    type,
    resourceType,
    clientTimestamp: Date.now(),
    serverTimestamp,
    syncStatus: "pending" as const,
    retries: 0,
    queryParams,
    queryrouteKey,
    parentMutationIds,
    dependentFields,
  };

  try {
    await db.offlineWrites.add(writeEntry);
    console.log("Offline write saved successfully:", writeEntry);
  } catch (error) {
    console.error(" Failed to save offline write:", error);
  }
};
```

> Note : Here again a critical point that is using navigator.online or tanstack network provider for checking network status, both can sometime giving wrong network status value in some cases.

### Phase 3: Synchronization

Now this is one of the important phase of offline support functionality. Here we will synchronize the data that we save during offline. so for that we need a sync manage . A production‑grade sync manager should include:

#### 1. fetching pending writes (`getPendingWrites`)

- Use the following method to retrieve all records:

  ```ts
  getPendingAndRetryableWrites(userId)
```

```ts
export async function getPendingAndRetryableWrites(
  userId: string
): Promise<OfflineWriteRecord[]> {
  return db.offlineWrites
    .where("userId")
    .equals(userId)
    .and((w) => {
      const isPending = w.syncStatus === "pending";
      const isFailedButRetryable =
        w.syncStatus === "failed" && (w.retries || 0) < MAX_RETRIES;
      return isPending || isFailedButRetryable;
    })
    .toArray();
}
```

#### 2. Processing each write (`syncOneWrite`) with proper error/409 handling and Retry/backoff logic (`shouldRetry`, `computeBackoffDelay`, scheduleRetry).

- For every item in that list, call `syncOneWrite(write)`, which:
  Runs conflict detection first (`detectAndMarkConflict`). If the server’s `modified_date` differs from the cached `serverTimestamp`, it immediately marks that record as "conflict" and skips any further mutation.
- If no `conflict`, attempts the API mutation `(mutate(route, payload))`.
- On success, updates syncStatus = "success" and timestamps the write.
    • On error: If HTTP 409, marks `syncStatus = "conflict"` and stores the server’s data in conflictData. Otherwise, marks syncStatus = "failed", increments retries, saves the error message, and—if `retries < MAX_RETRIES—schedules` a retry via `computeBackoffDelay(retries) + scheduleRetry`.

```ts
export async function syncOneWrite(write: OfflineWriteRecord): Promise<void> {
  const now = Date.now();

  // 1) Detect (and mark) conflict up front
  const didConflict = await detectAndMarkConflict(write);
  if (didConflict) {
    return;
  }

  try {
    // 2) Execute the mutation route
    const route = offlineRoutes[write.syncrouteKey as OfflineRouteKey];
    const mutationFn = mutate(route, { pathParams: write.pathParams as any });
    await mutationFn(write.payload);

    // 3) On success, mark as synced
    await db.offlineWrites.update(write.id, {
      syncStatus: "success",
      lastAttemptAt: now,
    });

    toast.success(`Sync succeeded for write ${write.id}`);
  } catch (error: any) {
    const nextRetries = (write.retries || 0) + 1;
    const isConflict = error?.response?.status === 409;

    const updateFields: Partial<OfflineWriteRecord> = {
      lastAttemptAt: now,
      retries: nextRetries,
      lastError: error?.message || "Unknown error",
    };

    if (isConflict) {
      // 4.a) On HTTP 409, mark as conflict
      updateFields.syncStatus = "conflict";
      updateFields.conflictData = error.response?.data;
    } else {
      // 4.b) On other errors, mark as failed
      updateFields.syncStatus = "failed";
    }

    await db.offlineWrites.update(write.id, updateFields);
    toast.error(`Sync failed for write ${write.id}: ${updateFields.lastError}`);

    // 5) Retry/backoff logic (only for non-conflict)
    if (!isConflict && shouldRetry(nextRetries)) {
      const delay = computeBackoffDelay(nextRetries);
      scheduleRetry(write.userId, delay);
    }
  }
}

export function shouldRetry(currentRetries: number): boolean {
  return currentRetries < MAX_RETRIES;
}

export function computeBackoffDelay(attempt: number): number {
  return BASE_BACKOFF_MS * 2 ** (attempt - 1);
}

export function scheduleRetry(userId: string, delayMs: number): void {
  setTimeout(() => {
    processSyncQueue(userId);
  }, delayMs);
}
```

#### 3. Conflict detection & resolution (`detectAndMarkConflict`) :- If server’s modified_date has changed since we cached it, that’s a conflict.

- `detectAndMarkConflict`(write) fetches the current server record (using queryrouteKey/queryParams) and compares `serverData.modified_date` with `write.serverTimestamp`.
- If they differ, it updates that write’s `syncStatus = "conflict"` and saves conflictData so the UI can prompt the user.

```ts
async function detectAndMarkConflict(
  write: OfflineWriteRecord
): Promise<boolean> {
  if (!write.queryrouteKey || !write.queryParams) {
    return false;
  }

  try {
    const fetchFn = query(
      offlineRoutes[write.queryrouteKey as OfflineRouteKey]
    );
    const serverData = await fetchFn({ pathParams: write.queryParams as any });

    if (serverData.modified_date !== write.serverTimestamp) {
      await db.offlineWrites.update(write.id, {
        syncStatus: "conflict",
        conflictData: serverData,
        lastAttemptAt: Date.now(),
      });
      return true;
    }
  } catch (err) {
    console.warn(`detectAndMarkConflict: failed for write ${write.id}`, err);
  }

  return false;
}
```

#### 4. Queue orchestration (`processSyncQueue`) triggered by network or timed events.

- `processSyncQueue(userId)` is triggered whenever the browser fires "online" (via onNetworkStatusChange) .
- Each pass fetches the current pending/retryable writes and calls syncOneWrite on each in series.

```ts
export async function processSyncQueue(userId: string): Promise<void> {
  if (!navigator.onLine || !userId) return;

  const writesToProcess = await getPendingAndRetryableWrites(userId);

  for (const write of writesToProcess) {
    await syncOneWrite(write);
  }
}
```

#### 5. cleanup (`cleanupSuccessfulWrites`).

- `cleanupSuccessfulWrites(userId, olderThanMs)` deletes any records where `


### Phase 4: Notifications (Sync Status Page)

The Notifications phase is implemented as a **Sync Status Page**—a dedicated interface where users can monitor and manage the status of all their offline writes and sync operations.

#### Purpose
- Provide a clear, centralized view of all data that is pending sync, has failed to sync, has conflicts, or has been successfully synced.
- Empower users to take action on failed or conflicted records directly from the page.

#### What the Sync Status Page Shows
- **Conflicts Queue:** List of records where a conflict was detected during sync. Each entry shows details of the conflict and provides an action to resolve it (e.g., open a merge dialog or accept/reject changes).
- **Failed Queue:** List of records that failed to sync due to network/server errors. Each entry includes error details and a "Retry" button to attempt sync again.
- **Success Queue:** (Optional) List or count of records that have been successfully synced, for audit or reassurance.
- **Pending Queue:** (Optional) Records that are saved locally and waiting for the next sync attempt.

#### Example UI Elements
- **Tables or Lists:** Each queue is displayed as a table or list with columns for record type, timestamp, status, and actions.
- **Badges:** Status badges (e.g., "conflict", "failed", "pending", "success") for quick visual identification.
- **Action Buttons:**
  - **Retry:** Manually retry syncing a failed or conflicted record.
  - **Resolve:** Open a conflict resolution dialog for conflicted records.
  - **View Details:** Inspect the payload, error, or conflict data for any record.

#### User Actions
- Users can manually retry failed/conflicted syncs.
- Users can resolve conflicts through a dedicated UI.
- Users can view details of any record in the sync queues.

This Sync Status Page ensures transparency and gives users control over their offline data, making it easy to track, troubleshoot, and resolve sync issues as they arise.


## Limitations and Known Issues

Before discussing limitation and known Issue , let say go through how CARE work offline :

- User logged-in  and run website, Whenever user go throught he workflows for which offline functionality is added there API-data will caached and save locally
- now suppose user goes offline( during Active session), Now it will can access those pages and data that is present in cache or user previosly visit.
- now if user fill forms , it will be saved offline if navigator.online==false(means we are offline).
- Now when network status switch to online , syncing start and all our data will be sync to backend.
- user can see their conflict and faild data in notification part.

Now lets take a look om limitation and known issue now :

 - **Reliance on navigator.onLine** : Browsers often report “online” when a device is connected to a local network but actually has no Internet access (e.g. captive portals or firewalls). This can cause the sync manager to attempt writes even though the server is unreachable
 - **No Background Sync Outside Active Tab**: All synchronization occurs only while the app is open and the user is logged-in. If the user closes the tab or the device sleeps, pending writes remain unsynced until the app is re‐opened.
- **Conflict Resolution Assumes Timestamp Accuracy**: We rely on comparing serverData.modified_date to write.serverTimestamp. If the server’s clock drifts or the timestamp field is not updated reliably, conflicts may be missed or falsely detected.
- **No offline support after Log-out**: when a user logout , its cached data will be cleared and it cannot access website offline. But clearing cache during logout increase security and also prevent mixing of data of two or more user if their is same device.

> Note : For some simple cases we can handle dependency enforcement  of  data during online. Simply means for those child write that not depend heavily on parent record.

## Implemenation Timeline and Milestone

As project implementation is divided into four phases , we  will  complete phase 1-2  from week 1-6 (till mid evaluation) and  phase 3-4 from week 7-12. 

| **Phase**               | **Timeline**      | **Key Deliverables**                                                                 |
|-------------------------|-------------------|-------------------------------------------------------------------------------------|
| **Phase 1: Caching**    | Week 1 – Week 3   | • Dexie schema & IndexedDB setup<br>• `PersistQueryClientProvider` integrated<br>• Selective caching (`meta.persist`) verified<br>• Implementation of all tech config mention in phase 1<br>• Offline reads tested |
| **Phase 2: Offline Writes**(mid-evaluation) | Week 4 – Week 6 | • `offlineWrites` table created<br>• `saveOfflineWrite` hooked into forms|
| **Phase 3: Synchronization** | Week 7 – Week 9 | • `syncOneWrite`, retry/backoff, and `computeBackoffDelay` complete<br>• `detectAndMarkConflict` implemented<br>• Implementation of all tech config mention in phase 3<br>• End-to-end sync (offline → online) tested |
