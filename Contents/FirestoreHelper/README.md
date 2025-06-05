Below is a complete breakdown of **FirebaseHelper**—a statically accessible utility class that layers Firestore functionality on top of Unity, **JsonHelper**, and a hypothetical **NetworkRequestManager**. It provides:

* Initialization of Firebase & Firestore
* CRUD operations (create/update, read, update fields, delete) for individual documents
* Parametric querying of collections
* Real-time “listener” subscriptions to document changes
* Batch writes (set/update/delete multiple documents atomically)
* A small DSL (`QueryCondition`, `QueryOperator`) for building queries
* Type-safe (de)serialization by leveraging **JsonHelper** and Unity’s `JsonUtility`
* Methods to upload files via **NetworkRequestManager** (e.g. HTTP-download) and store metadata in Firestore
* A helper to fetch remote JSON (via **NetworkRequestManager**) and sync it into Firestore

Use **FirebaseHelper** whenever you need to interact with Firestore from Unity—whether performing one-off reads and writes, subscribing to real-time updates, or batching a set of operations.

---

## Table of Contents

1. [Overview & Purpose](#overview--purpose)
2. [Initialization](#initialization)
3. [Document Operations](#document-operations)

   * 3.1. `SetDocumentAsync<T>(string collection, string documentId, T data)`
   * 3.2. `GetDocumentAsync<T>(string collection, string documentId)`
   * 3.3. `UpdateDocumentAsync(string collection, string documentId, Dictionary<string, object> updates)`
   * 3.4. `DeleteDocumentAsync(string collection, string documentId)`
4. [Query Operations](#query-operations)

   * 4.1. `QueryDocumentsAsync<T>(string collection, List<QueryCondition> conditions = null, int limit = 0)`
   * 4.2. `QueryCondition` & `QueryOperator` helper classes
5. [Real-Time Listeners](#real-time-listeners)

   * 5.1. `ListenToDocument<T>(string collection, string documentId, Action<T> onUpdate, Action<Exception> onError = null)`
   * 5.2. `RemoveListener(string listenerKey)`
   * 5.3. `RemoveAllListeners()`
6. [Batch Operations](#batch-operations)

   * 6.1. `ExecuteBatchAsync(List<BatchOperation> operations)`
   * 6.2. `BatchOperation` & `BatchOperationType` helper types
7. [Helper Types & Methods](#helper-types--methods)

   * 7.1. `QueryCondition` class
   * 7.2. `BatchOperation` & `BatchOperationType`
   * 7.3. `EnsureInitialized()` internal guard
8. [Integration with NetworkRequestManager](#integration-with-networkrequestmanager)

   * 8.1. `UploadFileWithMetadataAsync(string fileUrl, string destinationPath, string collection, string documentId, Dictionary<string, object> additionalMetadata = null)`
   * 8.2. `SyncRemoteJsonWithFirestoreAsync(string jsonUrl, string collection, string documentId)`
9. [Lifecycle & Cleanup](#lifecycle--cleanup)
10. [Real-World Usage Examples](#real-world-usage-examples)

    * 10.1. Initializing Firebase at startup
    * 10.2. Writing complex data (POCO) into a Firestore document
    * 10.3. Reading a document and converting it back to a C# object
    * 10.4. Updating only specific fields in an existing document
    * 10.5. Deleting a document
    * 10.6. Querying a collection with filters (e.g. “status == ‘active’ AND score ≥ 100”)
    * 10.7. Listening for real-time updates to a player’s profile document
    * 10.8. Executing multiple writes in a single Firestore batch (e.g. setting several documents at once)
    * 10.9. Uploading a file via HTTP, storing it in Firebase Storage, and saving its metadata in Firestore
    * 10.10. Fetching remote JSON and synchronizing it into Firestore
11. [Best Practices & Tips](#best-practices--tips)
12. [Summary of Public API](#summary-of-public-api)

---

## 1. Overview & Purpose

```csharp
public static class FirebaseHelper
{
    // … (methods and fields detailed below) …
}
```

**FirebaseHelper** is designed to wrap Firestore operations behind a few easy-to-use static methods. Internally, it:

1. **Initializes Firebase & Firestore** (once) when any operation is requested.
2. **Serializes** arbitrary C# objects (`T data`) to JSON via **JsonHelper**, then writes them to Firestore as `Dictionary<string, object>`.
3. **Deserializes** Firestore document snapshots back into typed objects using `JsonHelper` + `JsonUtility.FromJson<T>`.
4. Provides a simple **query DSL** for building `where` clauses.
5. Allows **real-time “snapshot listeners”** on a per-document basis.
6. Facilitates **batch writes** (set/update/delete) in a single commit.
7. Tightly integrates with an assumed **NetworkRequestManager** for HTTP requests, e.g. to download a file before uploading to Firebase Storage (not fully implemented).
8. Exposes two specialized methods under “NetworkRequestManager integration” to upload a file and store its metadata, and to fetch remote JSON and sync it to Firestore.

Use **FirebaseHelper** if you want to avoid writing repetitive Firestore boilerplate—especially converting between C# classes and the Firestore `Dictionary<string, object>` format, managing listener lifecycles, or doing batch writes.

---

## 2. Initialization

```csharp
private static FirebaseFirestore _firestoreInstance;
private static bool _isInitialized;
private static readonly Dictionary<string, ListenerRegistration> _activeListeners 
    = new Dictionary<string, ListenerRegistration>();

public static async Task<bool> InitializeAsync()
{
    if (_isInitialized) return true;

    try
    {
        // Ensure Firebase dependencies are resolved
        var app = await FirebaseApp.CheckAndFixDependenciesAsync();
        if (app != DependencyStatus.Available)
        {
            Debug.LogError("Could not resolve Firebase dependencies");
            return false;
        }

        // Acquire the Firestore instance
        _firestoreInstance = FirebaseFirestore.DefaultInstance;
        _isInitialized = true;
        return true;
    }
    catch (Exception ex)
    {
        Debug.LogError($"Firebase initialization failed: {ex.Message}");
        return false;
    }
}
```

* **What It Does**:

  * Calls `FirebaseApp.CheckAndFixDependenciesAsync()`, which verifies that all required Firebase SDK components are available and, if possible, fixes any missing dependencies.
  * If dependencies are satisfied, sets `_firestoreInstance = FirebaseFirestore.DefaultInstance` and marks `_isInitialized = true`.
  * Returns `true` if initialization succeeds; otherwise, logs an error and returns `false`.

* **When to Call**:

  * Internally, almost every public method first calls `EnsureInitialized()` (see below) to lazily initialize Firestore if needed.
  * You can also call `await FirebaseHelper.InitializeAsync()` at game startup (e.g., in a manager’s `Awake()` or `Start()`), to ensure readiness ahead of time.

```csharp
private static async Task<bool> EnsureInitialized()
{
    if (!_isInitialized)
    {
        return await InitializeAsync();
    }
    return true;
}
```

* **`EnsureInitialized()`** simply returns early if `_isInitialized` is already `true`; otherwise, it calls `InitializeAsync()`.

---

## 3. Document Operations

### 3.1 `public static async Task<bool> SetDocumentAsync<T>(string collection, string documentId, T data)`

```csharp
public static async Task<bool> SetDocumentAsync<T>(string collection, string documentId, T data)
{
    if (!await EnsureInitialized()) return false;

    try
    {
        // 1. Serialize 'data' to JSON:
        string json = JsonHelper.Serialize(data);
        // 2. Convert JSON to Dictionary<string, object>
        Dictionary<string, object> dict = JsonHelper.DeserializeToDictionary(json);

        // 3. Write/overwrite the Firestore document:
        await _firestoreInstance
            .Collection(collection)
            .Document(documentId)
            .SetAsync(dict);

        return true;
    }
    catch (Exception ex)
    {
        Debug.LogError($"Error setting document: {ex.Message}");
        return false;
    }
}
```

**Purpose**:
Create or overwrite a Firestore document at `“/{collection}/{documentId}”` with the contents of `data`. Internally, it:

1. Ensures Firestore is initialized.
2. Uses **JsonHelper** to serialize the generic object `data` to a JSON string, then deserialize that JSON into a `Dictionary<string, object>` (Firestore’s expected format).
3. Calls `SetAsync(dict)` on the document reference, which either creates the document (if none exists) or overwrites it (including subfields not present in `dict`).

**Usage Example**:

Suppose you have a C# class:

```csharp
[Serializable]
public class PlayerProfile
{
    public string Username;
    public int Level;
    public float Health;
}

// Elsewhere...
var profile = new PlayerProfile { Username = "Alice", Level = 5, Health = 100f };
bool success = await FirebaseHelper.SetDocumentAsync("profiles", "alice123", profile);
if (success)
    Debug.Log("Player profile successfully written to Firestore.");
```

* The resulting Firestore document `profiles/alice123` will contain fields `"Username": "Alice"`, `"Level": 5`, `"Health": 100.0`.

---

### 3.2 `public static async Task<T> GetDocumentAsync<T>(string collection, string documentId)`

```csharp
public static async Task<T> GetDocumentAsync<T>(string collection, string documentId)
{
    if (!await EnsureInitialized()) return default;

    try
    {
        DocumentSnapshot snapshot = await _firestoreInstance
            .Collection(collection)
            .Document(documentId)
            .GetSnapshotAsync();

        if (!snapshot.Exists)
            return default;  // Document doesn’t exist → return default(T)

        // 1. Convert Firestore document data (a Dictionary<string, object>) back to JSON:
        string json = JsonHelper.Serialize(snapshot.ToDictionary());
        // 2. Deserialize JSON into type T using Unity's JsonUtility:
        return JsonUtility.FromJson<T>(json);
    }
    catch (Exception ex)
    {
        Debug.LogError($"Error getting document: {ex.Message}");
        return default;
    }
}
```

**Purpose**:
Fetch a Firestore document from `“/{collection}/{documentId}”`, then convert it back into a C# object of type `T`. Internally:

1. Ensures Firestore is initialized.
2. Awaits `GetSnapshotAsync()`. If the document doesn’t exist, returns `default(T)` (typically `null` for reference types).
3. If it exists, calls `snapshot.ToDictionary()`, which gives `Dictionary<string, object>` for that document’s fields.
4. Serializes that dictionary back to a JSON string via **JsonHelper**.
5. Uses Unity’s `JsonUtility.FromJson<T>(json)` to produce a typed object of type `T`.

**Important**:

* Unity’s `JsonUtility` only supports basic types and nested classes with `[Serializable]` and public fields; it doesn’t handle dictionaries directly. That’s why the two-step: dictionary → JSON → `T`.
* If your type `T` has fields that Unity’s `JsonUtility` cannot handle (e.g., `List<object>` or types not marked `[Serializable]`), you may need to adjust or use `JsonHelper.Deserialize<T>(json)` instead.

**Usage Example**:

```csharp
[Serializable]
public class PlayerProfile
{
    public string Username;
    public int Level;
    public float Health;
}

// Elsewhere:
PlayerProfile profile = await FirebaseHelper.GetDocumentAsync<PlayerProfile>("profiles", "alice123");
if (profile != null)
{
    Debug.Log($"Loaded profile: {profile.Username}, level {profile.Level}, HP {profile.Health}");
}
else
{
    Debug.Log("Profile not found or error occurred.");
}
```

---

### 3.3 `public static async Task<bool> UpdateDocumentAsync(string collection, string documentId, Dictionary<string, object> updates)`

```csharp
public static async Task<bool> UpdateDocumentAsync(
    string collection,
    string documentId,
    Dictionary<string, object> updates)
{
    if (!await EnsureInitialized()) return false;

    try
    {
        await _firestoreInstance
            .Collection(collection)
            .Document(documentId)
            .UpdateAsync(updates);

        return true;
    }
    catch (Exception ex)
    {
        Debug.LogError($"Error updating document: {ex.Message}");
        return false;
    }
}
```

**Purpose**:
Partially update an existing document (only the specified fields). Equivalent to a PATCH: it does not overwrite the entire document—only the keys in `updates` are modified. Keys not present in `updates` remain unchanged.

* If you provide a field that doesn’t exist in the document, Firestore will create it.
* If you set a field’s value to `FieldValue.Delete`, Firestore will remove that field.

**Usage Example**:

```csharp
// Suppose in Firestore we already have document "profiles/alice123" with fields Username, Level, Health
var updates = new Dictionary<string, object> 
{
    { "Level", 6 },
    { "LastLogin", DateTime.UtcNow } // adds a new timestamp field
};

bool success = await FirebaseHelper.UpdateDocumentAsync("profiles", "alice123", updates);
if (success)
    Debug.Log("Profile partially updated: Level and LastLogin modified.");
```

---

### 3.4 `public static async Task<bool> DeleteDocumentAsync(string collection, string documentId)`

```csharp
public static async Task<bool> DeleteDocumentAsync(string collection, string documentId)
{
    if (!await EnsureInitialized()) return false;

    try
    {
        await _firestoreInstance
            .Collection(collection)
            .Document(documentId)
            .DeleteAsync();

        return true;
    }
    catch (Exception ex)
    {
        Debug.LogError($"Error deleting document: {ex.Message}");
        return false;
    }
}
```

**Purpose**:
Remove the document at `“/{collection}/{documentId}”` entirely from Firestore. After this operation, `GetDocumentAsync` for that document will return `default(T)` and `snapshot.Exists` will be false.

**Usage Example**:

```csharp
bool success = await FirebaseHelper.DeleteDocumentAsync("profiles", "alice123");
if (success)
    Debug.Log("Profile deleted successfully.");
```

---

## 4. Query Operations

### 4.1 `public static async Task<List<T>> QueryDocumentsAsync<T>(string collection, List<QueryCondition> conditions = null, int limit = 0)`

```csharp
public static async Task<List<T>> QueryDocumentsAsync<T>(
    string collection,
    List<QueryCondition> conditions = null,
    int limit = 0)
{
    if (!await EnsureInitialized()) return new List<T>();

    try
    {
        Query query = _firestoreInstance.Collection(collection);

        // 1. Apply each condition (if any)
        if (conditions != null)
        {
            foreach (var condition in conditions)
            {
                query = condition.ApplyToQuery(query);
            }
        }

        // 2. Apply a limit, if specified
        if (limit > 0)
            query = query.Limit(limit);

        // 3. Execute the query
        QuerySnapshot querySnapshot = await query.GetSnapshotAsync();

        // 4. Convert each document back to T
        List<T> results = new List<T>();
        foreach (DocumentSnapshot doc in querySnapshot.Documents)
        {
            string json = JsonHelper.Serialize(doc.ToDictionary());
            T item = JsonUtility.FromJson<T>(json);
            results.Add(item);
        }
        return results;
    }
    catch (Exception ex)
    {
        Debug.LogError($"Error querying documents: {ex.Message}");
        return new List<T>();
    }
}
```

**Purpose**:
Fetch multiple documents from a Firestore collection that satisfy a set of conditions (e.g., `"score" >= 100 AND "status" == "active"`). Returns a `List<T>` where `T` is some `[Serializable]` class.

1. Ensures Firestore is initialized.
2. Starts with `Query query = Collection(collection)`.
3. For each `QueryCondition` in `conditions`, calls `ApplyToQuery(query)`, which returns an updated `Query` with the `Where` clause applied.
4. If `limit > 0`, appends `.Limit(limit)` to the query.
5. Awaits `query.GetSnapshotAsync()` to retrieve matching documents.
6. For each `DocumentSnapshot`, uses `doc.ToDictionary()` → JSON → `JsonUtility.FromJson<T>(json)` to convert back to a typed C# object.
7. Returns the list of typed objects.

**`QueryCondition` and `QueryOperator`** are small helper types (see below) that let you specify something like:

```csharp
var conditions = new List<FirebaseHelper.QueryCondition>
{
    new FirebaseHelper.QueryCondition {
        Field = "score",
        Operator = FirebaseHelper.QueryOperator.GreaterThanOrEqualTo,
        Value = 100
    },
    new FirebaseHelper.QueryCondition {
        Field = "status",
        Operator = FirebaseHelper.QueryOperator.EqualTo,
        Value = "active"
    }
};

List<PlayerProfile> topPlayers = await FirebaseHelper.QueryDocumentsAsync<PlayerProfile>(
    "leaderboard",
    conditions,
    limit: 10
);
```

---

### 4.2 `QueryCondition` & `QueryOperator` Helper Types

```csharp
public class QueryCondition
{
    public string Field { get; set; }
    public QueryOperator Operator { get; set; }
    public object Value { get; set; }

    public Query ApplyToQuery(Query query)
    {
        return Operator switch
        {
            QueryOperator.EqualTo                 => query.WhereEqualTo(Field, Value),
            QueryOperator.LessThan                => query.WhereLessThan(Field, Value),
            QueryOperator.LessThanOrEqualTo       => query.WhereLessThanOrEqualTo(Field, Value),
            QueryOperator.GreaterThan             => query.WhereGreaterThan(Field, Value),
            QueryOperator.GreaterThanOrEqualTo    => query.WhereGreaterThanOrEqualTo(Field, Value),
            QueryOperator.ArrayContains           => query.WhereArrayContains(Field, Value),
            _                                     => query
        };
    }
}

public enum QueryOperator
{
    EqualTo,
    LessThan,
    LessThanOrEqualTo,
    GreaterThan,
    GreaterThanOrEqualTo,
    ArrayContains
}
```

* **`QueryCondition`**:

  * Holds a field name (e.g. `"score"`)
  * An operator (e.g. `QueryOperator.GreaterThanOrEqualTo`)
  * A value (e.g. `100`)
  * Its `ApplyToQuery(Query q)` method simply calls the correct Firestore `Where…` method on the incoming `Query` instance and returns it.

* **`QueryOperator`**:

  * A small `enum` that corresponds exactly to Firestore’s `.WhereEqualTo()`, `.WhereLessThan()`, etc.
  * `ArrayContains` allows you to check if an array field contains a given element.

---

## 5. Real-Time Listeners

### 5.1 `public static void ListenToDocument<T>(string collection, string documentId, Action<T> onUpdate, Action<Exception> onError = null)`

```csharp
public static void ListenToDocument<T>(
    string collection,
    string documentId,
    Action<T> onUpdate,
    Action<Exception> onError = null)
{
    if (!_isInitialized)
    {
        Debug.LogError("Firebase not initialized");
        return;
    }

    string listenerKey = $"{collection}/{documentId}";

    // 1. If there’s already a listener for this key, remove it first:
    RemoveListener(listenerKey);

    try
    {
        // 2. Start listening
        var listener = _firestoreInstance
            .Collection(collection)
            .Document(documentId)
            .Listen(snapshot =>
            {
                if (snapshot.Exists)
                {
                    // Convert updated data to type T and invoke callback
                    string json = JsonHelper.Serialize(snapshot.ToDictionary());
                    T data = JsonUtility.FromJson<T>(json);
                    onUpdate?.Invoke(data);
                }
            });

        // 3. Track this listener so we can remove it later
        _activeListeners[listenerKey] = listener;
    }
    catch (Exception ex)
    {
        Debug.LogError($"Error setting up listener: {ex.Message}");
        onError?.Invoke(ex);
    }
}
```

**Purpose**:
Subscribe to real-time updates on a specific Firestore document. Whenever any data in that document changes (including first initial load), Firestore will invoke the provided callback with a fresh snapshot.

1. **Guard**: If `_isInitialized` is `false`, logs an error and returns immediately.
2. Builds a unique `listenerKey = $"{collection}/{documentId}"` to track the listener.
3. If a listener already exists under that key, calls `RemoveListener(listenerKey)` to prevent duplicate subscriptions.
4. Calls `.Listen(...)` on the document reference. Inside the callback:

   * If `snapshot.Exists` (document still exists), convert its dictionary to JSON, then to type `T`, and invoke `onUpdate(data)`.
   * (You could also handle the case where `!snapshot.Exists` if you want to notify clients that the document was deleted.)
5. Stores the returned `ListenerRegistration` object in `_activeListeners[listenerKey]` so that you can stop it later.

* **`ListenerRegistration`** is a Firestore handle: calling `.Stop()` on it unregisters the listener.

**Usage Example**:

```csharp
// 1. Define a data class:
[Serializable]
public class PlayerProfile
{
    public string Username;
    public int Level;
    public float Health;
}

// 2. Somewhere in code (e.g., after calling InitializeAsync()):
FirebaseHelper.ListenToDocument<PlayerProfile>(
    "profiles",
    "alice123",
    onUpdate: profile =>
    {
        // Called on every change to profiles/alice123
        Debug.Log($"Realtime profile update: {profile.Username} now at level {profile.Level}");
    },
    onError: ex =>
    {
        Debug.LogError($"Listener error: {ex.Message}");
    }
);
```

Once set up, your `onUpdate` callback fires immediately with the current snapshot (if it exists), and then again whenever the document changes.

---

### 5.2 `public static void RemoveListener(string listenerKey)`

```csharp
public static void RemoveListener(string listenerKey)
{
    if (_activeListeners.TryGetValue(listenerKey, out var listener))
    {
        listener.Stop();                   // Unregister from Firestore updates
        _activeListeners.Remove(listenerKey);
    }
}
```

**Purpose**:
Stop an existing real-time listener identified by `listenerKey` (the same string you passed to `ListenToDocument`). Internally, it:

1. Checks `_activeListeners` if a `ListenerRegistration` exists for that key.
2. Calls `listener.Stop()`, which unsubscribes from Firestore updates.
3. Removes that entry from `_activeListeners`.

**Usage Example**:

```csharp
// Suppose you previously called:
// FirebaseHelper.ListenToDocument<PlayerProfile>("profiles", "alice123", /* ... */);

// Later, maybe on scene unload:
FirebaseHelper.RemoveListener("profiles/alice123");
```

---

### 5.3 `public static void RemoveAllListeners()`

```csharp
public static void RemoveAllListeners()
{
    foreach (var listener in _activeListeners.Values)
    {
        listener.Stop();
    }
    _activeListeners.Clear();
}
```

**Purpose**:
Bulk-removes all real-time Firestore listeners created by **FirebaseHelper**. Useful if you’re unloading a scene or shutting down the game.

**Usage Example**:

```csharp
// When your game or application is closing, or when you know you don't need updates:
// for instance, in OnDestroy() of a GameManager:
FirebaseHelper.RemoveAllListeners();
```

---

## 6. Batch Operations

### 6.1 `public static async Task<bool> ExecuteBatchAsync(List<BatchOperation> operations)`

```csharp
public static async Task<bool> ExecuteBatchAsync(List<BatchOperation> operations)
{
    if (!await EnsureInitialized()) return false;

    try
    {
        // 1. Start a new batch
        WriteBatch batch = _firestoreInstance.StartBatch();

        // 2. For each operation, add the appropriate write to the batch
        foreach (var operation in operations)
        {
            DocumentReference docRef = _firestoreInstance
                .Collection(operation.Collection)
                .Document(operation.DocumentId);

            switch (operation.Type)
            {
                case BatchOperationType.Set:
                    batch.Set(docRef, operation.Data);
                    break;
                case BatchOperationType.Update:
                    batch.Update(docRef, operation.Data);
                    break;
                case BatchOperationType.Delete:
                    batch.Delete(docRef);
                    break;
            }
        }

        // 3. Commit all writes atomically
        await batch.CommitAsync();
        return true;
    }
    catch (Exception ex)
    {
        Debug.LogError($"Error executing batch: {ex.Message}");
        return false;
    }
}
```

**Purpose**:
Perform multiple Firestore write operations in a single **atomic** batch. The batch can include:

* **Set**: Write or overwrite a document with provided data.
* **Update**: Modify specific fields of an existing document.
* **Delete**: Delete a document entirely.

If any operation in the batch fails (e.g., permission error, network error), none of the changes are committed.

**Usage Example**:

```csharp
// Suppose we want to set two user profiles and delete a third in one atomic batch:
var ops = new List<FirebaseHelper.BatchOperation>
{
    new FirebaseHelper.BatchOperation {
        Collection = "profiles",
        DocumentId = "userA",
        Type = FirebaseHelper.BatchOperationType.Set,
        Data = new Dictionary<string, object> {
            { "Username", "Alice" },
            { "Level", 7 }
        }
    },
    new FirebaseHelper.BatchOperation {
        Collection = "profiles",
        DocumentId = "userB",
        Type = FirebaseHelper.BatchOperationType.Set,
        Data = new Dictionary<string, object> {
            { "Username", "Bob" },
            { "Level", 3 }
        }
    },
    new FirebaseHelper.BatchOperation {
        Collection = "profiles",
        DocumentId = "userC",
        Type = FirebaseHelper.BatchOperationType.Delete,
        Data = null // Data is ignored for delete
    }
};

bool batchSuccess = await FirebaseHelper.ExecuteBatchAsync(ops);
if (batchSuccess)
    Debug.Log("Batch write succeeded: userA, userB created/updated; userC deleted.");
```

---

### 6.2 `BatchOperation` & `BatchOperationType` Helper Types

```csharp
public class BatchOperation
{
    public string Collection { get; set; }
    public string DocumentId { get; set; }
    public Dictionary<string, object> Data { get; set; }
    public BatchOperationType Type { get; set; }
}

public enum BatchOperationType
{
    Set,
    Update,
    Delete
}
```

* **`BatchOperation`**:

  * `Collection`: Name of the Firestore collection.
  * `DocumentId`: ID of the document within the collection.
  * `Data`: A dictionary representing the document’s fields (`null` for deletes).
  * `Type`: One of `Set`, `Update`, or `Delete`.

---

## 7. Helper Types & Methods

### 7.1 `QueryCondition` (and `QueryOperator`)

*(Described in section 4 above.)*

### 7.2 `BatchOperation` & `BatchOperationType`

*(Described in section 6 above.)*

### 7.3 Internal Method: `EnsureInitialized()`

```csharp
private static async Task<bool> EnsureInitialized()
{
    if (!_isInitialized)
    {
        return await InitializeAsync();
    }
    return true;
}
```

* Ensures that `InitializeAsync()` has been called once before performing any Firestore operation.
* Returns `true` if Firestore is ready, `false` otherwise.

---

## 8. Integration with NetworkRequestManager

> *Note: This section assumes you have a separate `NetworkRequestManager` with methods `GetBytesAsync(string url)` and `GetStringAsync(string url)` that perform HTTP downloads.*

### 8.1 `public static async Task<bool> UploadFileWithMetadataAsync(string fileUrl, string destinationPath, string collection, string documentId, Dictionary<string, object> additionalMetadata = null)`

```csharp
public static async Task<bool> UploadFileWithMetadataAsync(
    string fileUrl,
    string destinationPath,
    string collection,
    string documentId,
    Dictionary<string, object> additionalMetadata = null)
{
    try
    {
        // 1. Download file bytes via NetworkRequestManager
        byte[] fileData = await NetworkRequestManager.GetBytesAsync(fileUrl);

        // 2. [Placeholder] Upload to Firebase Storage
        //    (Implementation depends on Firebase Storage SDK; not provided here.)
        //    e.g.: await FirebaseStorage.DefaultInstance
        //              .GetReference(destinationPath)
        //              .PutBytesAsync(fileData);

        // 3. Prepare metadata dictionary
        Dictionary<string, object> metadata = new Dictionary<string, object>
        {
            { "path", destinationPath },
            { "uploadedAt", DateTime.UtcNow },
            { "size", fileData.Length }
        };
        if (additionalMetadata != null)
        {
            foreach (var kvp in additionalMetadata)
                metadata[kvp.Key] = kvp.Value;
        }

        // 4. Write metadata to Firestore
        return await SetDocumentAsync(collection, documentId, metadata);
    }
    catch (Exception ex)
    {
        Debug.LogError($"Error uploading file with metadata: {ex.Message}");
        return false;
    }
}
```

**Purpose**:

1. **Downloads** a file from `fileUrl` as a byte array.
2. (Placeholder) **Uploads** the byte array to Firebase Storage at `destinationPath` (this code assumes you’ll plug in your own Storage upload logic).
3. Builds a metadata dictionary containing:

   * `"path"`: where it’s stored in Firebase Storage
   * `"uploadedAt"`: a timestamp
   * `"size"`: file size in bytes
   * plus any `additionalMetadata` you pass in.
4. Writes that metadata dictionary into Firestore under `“/{collection}/{documentId}”` using `SetDocumentAsync`.

**Usage Example**:

```csharp
async void ExampleUpload()
{
    string imageUrl = "https://example.com/images/avatar.png";
    string storagePath = "avatars/aliceAvatar.png";
    bool success = await FirebaseHelper.UploadFileWithMetadataAsync(
        imageUrl,
        storagePath,
        collection: "userFiles",
        documentId: "aliceAvatarMetadata",
        additionalMetadata: new Dictionary<string, object> {
            { "userId", "alice123" },
            { "description", "User’s profile avatar" }
        }
    );

    if (success)
        Debug.Log("File uploaded and metadata saved.");
}
```

---

### 8.2 `public static async Task<bool> SyncRemoteJsonWithFirestoreAsync(string jsonUrl, string collection, string documentId)`

```csharp
public static async Task<bool> SyncRemoteJsonWithFirestoreAsync(
    string jsonUrl,
    string collection,
    string documentId)
{
    try
    {
        // 1. Fetch JSON string from remote URL
        string jsonData = await NetworkRequestManager.GetStringAsync(jsonUrl);

        // 2. Parse JSON into a Dictionary<string, object>
        Dictionary<string, object> data = JsonHelper.DeserializeToDictionary(jsonData);

        // 3. Write or overwrite Firestore document with that data
        return await SetDocumentAsync(collection, documentId, data);
    }
    catch (Exception ex)
    {
        Debug.LogError($"Error syncing remote JSON with Firestore: {ex.Message}");
        return false;
    }
}
```

**Purpose**:
Download a JSON payload from a remote HTTP endpoint (`jsonUrl`), parse it into a dictionary, and push it into Firestore at `“/{collection}/{documentId}”`. Useful for periodically syncing some remote configuration or data set into your Firestore database.

**Usage Example**:

```csharp
async void ExampleSync()
{
    string remoteConfigUrl = "https://example.com/configs/gameSettings.json";
    bool success = await FirebaseHelper.SyncRemoteJsonWithFirestoreAsync(
        remoteConfigUrl,
        "configs",
        "gameSettings"
    );

    if (success)
        Debug.Log("Remote JSON successfully synced to Firestore.");
}
```

---

## 9. Lifecycle & Cleanup

* **Initialization**: Call `await FirebaseHelper.InitializeAsync()` before any other method, or rely on lazy initialization via `EnsureInitialized()`.
* **Real-Time Listeners**:

  * Calling `ListenToDocument<T>` registers a listener.
  * You must explicitly call `RemoveListener(key)` or `RemoveAllListeners()` to stop receiving updates and to prevent memory leaks.
  * Typically, call `RemoveAllListeners()` on application quit or scene unload.

There is no explicit “shutdown Firestore” call here—Firebase’s SDK handles that automatically when the application closes.

---

## 10. Real-World Usage Examples

Below are end-to-end examples demonstrating each major feature in a Unity context.

---

### 10.1 Initializing Firebase at Startup

```csharp
public class GameManager : MonoBehaviour
{
    private async void Awake()
    {
        bool initOK = await FirebaseHelper.InitializeAsync();
        if (!initOK)
        {
            Debug.LogError("Firebase failed to initialize. Disabling online features.");
            // You might disable any UI buttons that rely on Firestore, etc.
        }
        else
        {
            Debug.Log("Firebase and Firestore ready to go.");
        }
    }
}
```

* Call `InitializeAsync()` in `Awake` so you can know whether Firestore is available. You can then safely call other **FirebaseHelper** methods afterward.

---

### 10.2 Writing Complex Data (POCO) into a Firestore Document

```csharp
[Serializable]
public class LevelData
{
    public int LevelNumber;
    public string LevelName;
    public Vector3 SpawnPoint;         // Vector3 requires a custom JsonConverter if you want to store it
    public List<int> EnemyIDs;
}

public class LevelManager : MonoBehaviour
{
    public async void SaveLevelData()
    {
        var data = new LevelData
        {
            LevelNumber = 3,
            LevelName = "Haunted Forest",
            SpawnPoint = new Vector3(0, 1, 0), // If using JsonHelper.Serialize, register a Vector3Converter
            EnemyIDs = new List<int> { 101, 102, 103 }
        };

        bool ok = await FirebaseHelper.SetDocumentAsync("levels", "level_3", data);
        if (ok)
            Debug.Log("Level data saved.");
    }
}
```

* Make sure that if you have a field like `Vector3 SpawnPoint`, you register a `Vector3Converter` with **JsonHelper** (as shown in the previous examples). Otherwise `JsonUtility.FromJson` won’t correctly reconstruct `Vector3`.

---

### 10.3 Reading a Document and Converting It Back to a C# Object

```csharp
public class LevelLoader : MonoBehaviour
{
    public async void LoadLevelData(int levelNumber)
    {
        string docId = $"level_{levelNumber}";
        LevelData data = await FirebaseHelper.GetDocumentAsync<LevelData>("levels", docId);

        if (data != null)
        {
            Debug.Log($"Loaded Level: {data.LevelName}, spawn at {data.SpawnPoint}");
            // Use data.SpawnPoint, data.EnemyIDs, etc.
        }
        else
        {
            Debug.LogWarning($"Level {levelNumber} data not found in Firestore.");
        }
    }
}
```

* `GetDocumentAsync<LevelData>` reads back the dictionary from Firestore, serializes it to JSON, then runs `JsonUtility.FromJson<LevelData>`.
* Again, ensure any special types (e.g. `Vector3`) have converters if needed.

---

### 10.4 Updating Only Specific Fields in an Existing Document

```csharp
public class PlayerProgressUpdater : MonoBehaviour
{
    public async void UpdatePlayerScore(string playerId, int newScore)
    {
        var updates = new Dictionary<string, object>
        {
            { "Score", newScore },
            { "LastPlayed", DateTime.UtcNow }
        };

        bool ok = await FirebaseHelper.UpdateDocumentAsync("players", playerId, updates);
        if (ok)
            Debug.Log($"Player {playerId}’s score updated to {newScore}.");
    }
}
```

* Only the specified keys (`"Score"` and `"LastPlayed"`) are updated; any other fields (e.g., `"Username"`, `"Level"`) remain intact.

---

### 10.5 Deleting a Document

```csharp
public class AdminPanel : MonoBehaviour
{
    public async void RemoveObsoleteProfile(string userId)
    {
        bool ok = await FirebaseHelper.DeleteDocumentAsync("profiles", userId);
        if (ok)
            Debug.Log($"Profile {userId} deleted from Firestore.");
        else
            Debug.LogWarning($"Failed to delete profile {userId}.");
    }
}
```

* `DeleteDocumentAsync` simply calls `DeleteAsync()` on the Firestore document. After that, `GetDocumentAsync` for that ID will return `default(T)` and `snapshot.Exists` will be false.

---

### 10.6 Querying a Collection with Filters

```csharp
public class LeaderboardManager : MonoBehaviour
{
    /// <summary>
    /// Fetch top N players whose score ≥ minScore.
    /// </summary>
    public async void FetchTopPlayers(int minScore, int topN)
    {
        var conditions = new List<FirebaseHelper.QueryCondition>
        {
            new FirebaseHelper.QueryCondition {
                Field = "Score",
                Operator = FirebaseHelper.QueryOperator.GreaterThanOrEqualTo,
                Value = minScore
            }
        };

        // Note: You might also sort by a field, but Firestore sorting isn’t covered by QueryCondition—just add .OrderBy("Score") manually if needed.
        List<PlayerProfile> topPlayers = await FirebaseHelper.QueryDocumentsAsync<PlayerProfile>(
            "players",
            conditions,
            limit: topN
        );

        foreach (var player in topPlayers)
        {
            Debug.Log($"Player: {player.Username}, Score: {player.Score}");
        }
    }
}
```

* Only supports simple “where” ops. If you need ordering or more complex queries (e.g., range on multiple fields, `orderBy`, `startAfter`, `startAt`), you’d extend the DSL or manually manipulate the `Query` object.

---

### 10.7 Listening for Real-Time Updates to a Player’s Profile Document

```csharp
public class PlayerProfileUI : MonoBehaviour
{
    private string _playerId = "alice123";

    private void OnEnable()
    {
        FirebaseHelper.ListenToDocument<PlayerProfile>(
            "profiles",
            _playerId,
            onUpdate: profile =>
            {
                // Called immediately with current snapshot, then on each change
                UpdateUIWithProfile(profile);
            },
            onError: ex =>
            {
                Debug.LogError($"Realtime listener error: {ex.Message}");
            }
        );
    }

    private void OnDisable()
    {
        // Stop listening when this UI is hidden/destroyed
        FirebaseHelper.RemoveListener($"profiles/{_playerId}");
    }

    private void UpdateUIWithProfile(PlayerProfile profile)
    {
        // e.g., update health bar, level text, etc.
        Debug.Log($"Realtime: {profile.Username} is now at HP {profile.Health}");
    }
}
```

* As soon as you call `ListenToDocument`, Firestore fires the listener once with the current data (if it exists).
* Any subsequent change in Firestore for that document invokes the callback again.
* Don’t forget to `RemoveListener("profiles/alice123")` in `OnDisable` or whenever you no longer need updates—otherwise, the listener will persist (and even outlive this component).

---

### 10.8 Executing Multiple Writes in a Single Firestore Batch

```csharp
public class GameEventManager : MonoBehaviour
{
    public async void AwardDailyBonusToPlayers(List<string> playerIds)
    {
        var operations = new List<FirebaseHelper.BatchOperation>();

        foreach (var pid in playerIds)
        {
            var op = new FirebaseHelper.BatchOperation
            {
                Collection = "players",
                DocumentId = pid,
                Type = FirebaseHelper.BatchOperationType.Update,
                Data = new Dictionary<string, object>
                {
                    { "Coins", FieldValue.Increment(100) }, // Firestore-specific increment
                    { "LastDailyBonus", Timestamp.GetCurrentTimestamp() }
                }
            };
            operations.Add(op);
        }

        bool ok = await FirebaseHelper.ExecuteBatchAsync(operations);
        if (ok)
            Debug.Log("Daily bonus awarded to all selected players.");
        else
            Debug.LogError("Failed to award daily bonuses in batch.");
    }
}
```

* Notice the use of Firestore’s `FieldValue.Increment(100)` to increment an existing numeric field rather than overwrite it. If you need more Firestore‐specific features like server timestamps or array union, simply pass those as values in `Data`.
* If any single update in that batch fails (e.g., document doesn’t exist, permissions error), none of them are committed.

---

### 10.9 Uploading a File via HTTP and Saving Its Metadata in Firestore

```csharp
public class UserAvatarUploader : MonoBehaviour
{
    public async void UploadAvatar(string imageUrl, string userId)
    {
        string storagePath = $"avatars/{userId}.png";
        bool success = await FirebaseHelper.UploadFileWithMetadataAsync(
            imageUrl,
            storagePath,
            collection: "avatars",
            documentId: userId + "_avatar",
            additionalMetadata: new Dictionary<string, object>
            {
                { "userId", userId },
                { "description", "User avatar image" }
            }
        );

        if (success)
            Debug.Log("Avatar uploaded and metadata stored in Firestore.");
        else
            Debug.LogError("Avatar upload failed.");
    }
}
```

* Internally, this method downloads the file via `NetworkRequestManager.GetBytesAsync(imageUrl)`, then (if you implement the Storage upload logic) uploads it to Firebase Storage under `avatars/{userId}.png`.
* Finally, it writes a Firestore document in collection `"avatars"` with ID `"{userId}_avatar"` that records:

  * `"path"`: the storage path
  * `"uploadedAt"`: timestamp
  * `"size"`: file size in bytes
  * plus any `additionalMetadata`.

---

### 10.10 Fetching Remote JSON and Synchronizing It into Firestore

```csharp
public class RemoteConfigSync : MonoBehaviour
{
    public async void SyncConfig()
    {
        string configUrl = "https://example.com/appConfig.json";
        bool success = await FirebaseHelper.SyncRemoteJsonWithFirestoreAsync(
            configUrl,
            collection: "configs",
            documentId: "appConfig"
        );

        if (success)
            Debug.Log("Configuration JSON synced to Firestore.");
        else
            Debug.LogError("Failed to sync configuration.");
    }
}
```

* The method uses `NetworkRequestManager.GetStringAsync(configUrl)` to retrieve the JSON text.
* Then `JsonHelper.DeserializeToDictionary(jsonData)` → `Dictionary<string, object>`.
* Finally writes that dictionary into Firestore under `configs/appConfig`.

---

## 11. Best Practices & Tips

1. **Call `InitializeAsync()` Early**

   * Although `EnsureInitialized()` is called lazily, it’s good practice to explicitly call `await FirebaseHelper.InitializeAsync()` in your main entry point (e.g., `GameManager.Awake`) so you know immediately whether Firestore is available and can disable any UI that relies on it.

2. **Watch for Unity’s `JsonUtility` Limitations**

   * `JsonUtility.FromJson<T>` only works with fields marked `[Serializable]` and does not support dictionaries or complex types (e.g., `Dictionary<string, object>`). That’s why **FirebaseHelper** first converts Firestore’s dictionary to JSON (via **JsonHelper**) before calling `JsonUtility.FromJson<T>`.
   * If your classes contain `List<object>` or polymorphic types, consider using `JsonHelper.Deserialize<T>(json)` directly (it uses **Newtonsoft.Json**) instead of `JsonUtility`.

3. **FieldValue Helpers**

   * When doing partial updates (`UpdateDocumentAsync` or in a batch), you can pass Firestore’s special sentinel values like `FieldValue.ServerTimestamp`, `FieldValue.Increment(…)`, or `FieldValue.ArrayUnion(…)` directly in your `updates` dictionary.

4. **Handling Non-Existence in Listeners**

   * `ListenToDocument<T>` only invokes `onUpdate` when `snapshot.Exists` is true. If you want to handle deletions (i.e., if a document is removed), you could modify the listener callback to call `onUpdate(default)` or a separate callback when `snapshot.Exists` is false.

5. **Batch Writes Are Atomic**

   * Remember that **either all writes in the batch succeed or none do**. This is critical for consistency in multiplayer games or inventory updates.

6. **Cleanup Listeners**

   * Always call `RemoveListener(listenerKey)` or `RemoveAllListeners()` when a component is disabled or destroyed. Otherwise, you risk callbacks firing on destroyed objects and potential memory leaks.

7. **Payload Size**

   * Firestore has size limits per document (roughly 1 MiB). Before `SetDocumentAsync`, ensure your serialized dictionary/dataset won’t exceed that. You can inspect the JSON size via `json.Length`.

8. **Use `SyncRemoteJsonWithFirestoreAsync` Sparingly**

   * Downloading large JSON files repeatedly can be slow or consume bandwidth. Consider caching or checking if your local version is up to date before syncing.

9. **Error Handling**

   * All methods catch exceptions, log errors with `Debug.LogError`, and return a “safe” default (`false` or `null`). In production, consider surfacing these errors to players (e.g., “Unable to save progress—check your connection”).

10. **Thread Context**

    * All public methods return `Task` or `Task<T>` so you can `await` them from an `async` Unity context. However, Unity’s main thread is single-threaded, so be careful not to block it. Use `await` rather than `.Result` to avoid deadlocks.

---

## 12. Summary of Public API

Below is a concise list of all **public** methods and types in **FirebaseHelper**, grouped by category. For each method, see the sections above for detailed descriptions and examples.

### Initialization

| Signature                             | Description                                       |
| ------------------------------------- | ------------------------------------------------- |
| `static Task<bool> InitializeAsync()` | Resolve Firebase dependencies & obtain Firestore. |

### Document Operations

| Signature                                                                                                         | Description                                                |
| ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `static Task<bool> SetDocumentAsync<T>(string collection, string documentId, T data)`                             | Serialize `data` via JsonHelper and write to Firestore.    |
| `static Task<T> GetDocumentAsync<T>(string collection, string documentId)`                                        | Read document, convert to JSON, then to `T` (JsonUtility). |
| `static Task<bool> UpdateDocumentAsync(string collection, string documentId, Dictionary<string, object> updates)` | Partially update specified fields in a document.           |
| `static Task<bool> DeleteDocumentAsync(string collection, string documentId)`                                     | Delete a document entirely from Firestore.                 |

### Query Operations

| Signature                                                                                                               | Description                                                                |
| ----------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `static Task<List<T>> QueryDocumentsAsync<T>(string collection, List<QueryCondition> conditions = null, int limit = 0)` | Fetch documents matching conditions; convert each to type `T`.             |
| **Helper Types:** <br>`class QueryCondition`, <br>`enum QueryOperator { EqualTo, LessThan, …, ArrayContains }`          | Represent field, operator, value for building Firestore `.Where…` clauses. |

### Real-Time Listeners

| Signature                                                                                                                     | Description                                                                                      |
| ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `static void ListenToDocument<T>(string collection, string documentId, Action<T> onUpdate, Action<Exception> onError = null)` | Subscribe to real-time updates for a specific document.                                          |
| `static void RemoveListener(string listenerKey)`                                                                              | Stop and remove a previously registered listener (whose key was `$"{collection}/{documentId}"`). |
| `static void RemoveAllListeners()`                                                                                            | Stop and clear all active real-time listeners.                                                   |

### Batch Operations

| Signature                                                                                                                                                                                              | Description                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `static Task<bool> ExecuteBatchAsync(List<BatchOperation> operations)`                                                                                                                                 | Perform multiple set/update/delete operations atomically in one Firestore batch.    |
| **Helper Types:** <br>`class BatchOperation { string Collection; string DocumentId; Dictionary<string, object> Data; BatchOperationType Type; }` <br>`enum BatchOperationType { Set, Update, Delete }` | Describe each write operation (set, update, or delete) to be included in the batch. |

### Integration with NetworkRequestManager

> *(Requires a separate NetworkRequestManager with `GetBytesAsync(string)` and `GetStringAsync(string)`.)*

| Signature                                                                                                                                                                             | Description                                                                                                          |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `static Task<bool> UploadFileWithMetadataAsync( string fileUrl, string destinationPath, string collection, string documentId, Dictionary<string, object> additionalMetadata = null )` | Download a file from `fileUrl`, upload to Firebase Storage at `destinationPath`, then save metadata in Firestore.    |
| `static Task<bool> SyncRemoteJsonWithFirestoreAsync( string jsonUrl, string collection, string documentId )`                                                                          | Download JSON from `jsonUrl`, parse via JsonHelper, and write as a Firestore document under `collection/documentId`. |

---

With this documentation in hand, you should be able to use **FirebaseHelper** to perform virtually any Firestore‐related task from within your Unity project—ranging from simple writes and reads to real-time updates, batching, and even syncing external JSON or file uploads.
