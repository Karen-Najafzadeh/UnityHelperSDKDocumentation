Below is a comprehensive reference for **PrefsHelper**—a static utility that wraps Unity’s `PlayerPrefs`, JSON serialization, and Firebase Cloud Firestore to give you:

1. **Type-safe preference access** via an `enum` annotated with `[Type]` attributes
2. **Automatic JSON serialization** of Unity types (`Vector3`, `Color`, etc.)
3. **Optional cloud synchronization** (push/pull) to Firestore documents
4. **In-memory caching** to minimize repeated reads
5. **Bulk operations** (set/delete multiple keys)
6. **Complex-object support** (store/retrieve arbitrary objects as JSON)
7. **Encrypted storage** (left as a potential extension point—currently uses raw `PlayerPrefs`)
8. **Migration hooks** (via enum changes)

Below you’ll find:

1. [Enum & Attribute Definition](#enum--attribute-definition)
2. [Overview & Use Cases](#overview--use-cases)
3. [Public API](#public-api)

   * [Core Operations](#core-operations)
   * [Bulk Operations](#bulk-operations)
   * [Cloud Sync](#cloud-sync)
   * [JSON Integration](#json-integration)
4. [Helper Methods (Internal)](#helper-methods-internal)
5. [Real-World Usage Examples](#real-world-usage-examples)

   * [1. Storing & Retrieving Primitive Types](#1-storing--retrieving-primitive-types)
   * [2. Storing & Retrieving Unity Types (`Vector3`, `Color`)](#2-storing--retrieving-unity-types-vector3-color)
   * [3. Complex-Object via JSON](#3-complex-object-via-json)
   * [4. Bulk Set/Delete](#4-bulk-setdelete)
   * [5. Manual Cloud Sync / Auto Sync](#5-manual-cloud-sync--auto-sync)
   * [6. Migration Scenario](#6-migration-scenario)
6. [Design Notes & Tips](#design-notes--tips)
7. [Summary of Public Members](#summary-of-public-members)

---

## Enum & Attribute Definition

```csharp
/// <summary>
/// Attribute to specify the type of a preference key
/// </summary>
public class TypeAttribute : Attribute
{
    public string Type { get; }
    public TypeAttribute(string type) => Type = type;
}

/// <summary>
/// An enum listing all preference keys, each annotated with a [Type(...)].
/// </summary>
public enum GamePrefs
{
    [Type("int")]        Score,
    [Type("string")]     PlayerName,
    [Type("bool")]       IsTutorialComplete,
    [Type("vector3")]    LastPosition,
    [Type("color")]      UIColor,
    [Type("string")]     ComplexData  // for JSON-serialized objects
}
```

* **`TypeAttribute`**:

  * A simple attribute class that holds a single string, e.g. `"int"`, `"string"`, `"bool"`, `"vector3"`, `"color"`, etc.
  * Used to tell `PrefsHelper` how to read/write that key in `PlayerPrefs`.

* **`GamePrefs`**:

  * An example enumeration of all keys you want to store.
  * Each member is annotated with `[Type("…")]`, indicating how to handle the underlying value.
  * When you call `PrefsHelper.Set(GamePrefs.Score, 123)`, it will look up `TypeAttribute` → `"int"` → call `PlayerPrefs.SetInt("Score", 123)`.

You can create your own enum just as easily:

```csharp
public enum MyPrefs
{
    [Type("int")]    HighScore,
    [Type("float")]  MusicVolume,
    [Type("bool")]   HasCompletedTutorial,
    [Type("vector3")] PlayerSpawnPosition,
    [Type("string")] LastGameSummary
}
```

---

## Overview & Use Cases

**PrefsHelper** centralizes local (`PlayerPrefs`) and remote (Firestore) preference management. It offers:

1. **Type Safety**

   * You never pass a raw string key. Instead, you use `PrefsHelper.Set(GamePrefs.Score, 100)` or `PrefsHelper.Get<int, GamePrefs>(GamePrefs.Score)`.

2. **Automatic Conversion**

   * Supports primitives (`int`, `float`, `string`, `bool`) out of the box.
   * Also serializes Unity types (`Vector2`, `Vector3`, `Vector4`, `Color`, `Quaternion`) to JSON string and back.

3. **Caching**

   * Once you call `Get`, the value is stored in an in-memory `_cache`, so repeated `Get(Score)` does not hit `PlayerPrefs` again.

4. **Cloud Sync**

   * By default, whenever you call `Set(...)` with `cloudSync = true` (the default), **PrefsHelper** will:

     1. Write to `PlayerPrefs` locally
     2. Update its in-memory `_cache`
     3. Push to Firestore under collection `"preferences"`, document = key name, with JSON payload `{ "value": …, "timestamp": … }`

   * You can also pull from cloud via `SyncFromCloud<GamePrefs>()`, which will fetch every Firestore document under `"preferences"` and overwrite local values.

5. **Bulk Operations**

   * `SetBulk` to set many keys in one call (optionally sync at end).
   * `DeleteAll` to clear every key.

6. **Complex Objects**

   * `SetJson` + `GetJson` let you store any serializable object as JSON under a single key (e.g. `ComplexData` in the example).

7. **Automatic `PlayerPrefs.Save()`**

   * Every time you call `SetLocalValue`, the code calls `PlayerPrefs.Save()` immediately. That ensures local disk persistence.

8. **Migration Hooks**

   * Because you use an `enum` for keys, you can rename or remove keys centrally. If you change the enum, old PlayerPrefs keys remain untouched until you migrate them.

---

## Public API

### Core Operations

#### `public static async Task Set<TEnum>(TEnum key, object value, bool cloudSync = true) where TEnum : Enum`

* **Purpose**:
  Store a value (of any supported type) under the enum key; update local `PlayerPrefs`, update cache, then (optionally) push to Firestore.

* **Parameters**:

  1. `TEnum key`: An enum member, e.g. `GamePrefs.Score`
  2. `object value`: The actual value to store. Must be compatible with the `[Type("…")]` annotation for that key.
  3. `bool cloudSync`: If `true` (default), after writing locally, it also calls `SyncToCloud(...)`—uploading `{ "value": <converted>, "timestamp": <now> }` to Firestore.

* **How It Works Internally**:

  1. `keyName = key.ToString()` (e.g. `"Score"`)
  2. `type = GetKeyType(key)` → looks up `TypeAttribute` on the enum member (e.g. `"int"`)
  3. `SetLocalValue(keyName, value, type)` → calls the appropriate `PlayerPrefs.SetInt/SetFloat/SetString` or `JsonUtility.ToJson` for Unity types.
  4. `_cache[keyName] = value`
  5. If `cloudSync && _autoCloudSync`, call `await SyncToCloud(keyName, value)`

     * That builds `{ "value": <value>, "timestamp": DateTime.UtcNow }` and calls `FirebaseHelper.SetDocumentAsync("preferences", keyName, data)`.

* **Return**:
  An awaitable `Task`.

  * Awaiting it ensures that, if cloud sync is enabled, you know when Firestore write has completed (or failed).

* **Usage Example**:

  ```csharp
  // 1. Storing an integer:
  await PrefsHelper.Set(GamePrefs.Score, 1500);
  //   → PlayerPrefs.SetInt("Score", 1500); PlayerPrefs.Save();
  //   → _cache["Score"] = 1500
  //   → Firestore: preferences/Score = { "value": 1500, "timestamp": <now> }

  // 2. Storing a bool:
  await PrefsHelper.Set(GamePrefs.IsTutorialComplete, true);

  // 3. Storing a Vector3:
  Vector3 pos = new Vector3(1, 2, 3);
  await PrefsHelper.Set(GamePrefs.LastPosition, pos);

  // 4. If you don’t want cloud sync:
  await PrefsHelper.Set(GamePrefs.PlayerName, "Alice", cloudSync: false);
  //   → only writes to PlayerPrefs & cache, no Firestore call
  ```

---

#### `public static T Get<T, TEnum>(TEnum key, T defaultValue = default) where TEnum : Enum`

* **Purpose**:
  Retrieve a stored value of type `T` for the given enum key. Uses cache if available; otherwise reads from `PlayerPrefs` and populates cache.

* **Parameters**:

  1. `TEnum key`: The preference key, e.g. `GamePrefs.Score`
  2. `T defaultValue`: Optional default (e.g. `0` for ints, `""` for strings, `false` for bools, `Vector3.zero` for Vector3). Returned if `PlayerPrefs.HasKey(keyName)` is false.

* **How It Works**:

  1. `keyName = key.ToString()`
  2. If `_cache.TryGetValue(keyName, out cachedValue)`, return `Convert.ChangeType(cachedValue, typeof(T))`.
  3. Otherwise, `type = GetKeyType(key)` (e.g. `"int"`, `"vector3"`).
  4. `value = GetLocalValue(keyName, type, defaultValue)` → that calls `PlayerPrefs.GetInt/GetFloat/GetString` or `JsonUtility.FromJson<Vector3>(…)`.
  5. `_cache[keyName] = value`
  6. Return `(T)Convert.ChangeType(value, typeof(T))`.

* **Return**:
  The value stored under that key, cast to `T`. If not present, returns `defaultValue`.

* **Usage Example**:

  ```csharp
  int currentScore = PrefsHelper.Get<int, GamePrefs>(GamePrefs.Score, 0);
  // If "Score" was never set, returns 0.

  string playerName = PrefsHelper.Get<string, GamePrefs>(GamePrefs.PlayerName, "Guest");
  // If "PlayerName" was set to "Alice", returns "Alice"; else "Guest".

  bool didTutorial = PrefsHelper.Get<bool, GamePrefs>(GamePrefs.IsTutorialComplete, false);

  Vector3 lastPos = PrefsHelper.Get<Vector3, GamePrefs>(GamePrefs.LastPosition, Vector3.zero);
  ```

---

#### `public static async Task Delete<TEnum>(TEnum key, bool cloudSync = true) where TEnum : Enum`

* **Purpose**:
  Remove a single preference from both local `PlayerPrefs` and the in-memory cache; optionally delete the corresponding Firestore document.

* **Parameters**:

  1. `TEnum key`
  2. `bool cloudSync`: If `true` (default), also call `FirebaseHelper.DeleteDocumentAsync("preferences", keyName)` after deleting locally.

* **How It Works**:

  1. `keyName = key.ToString()`
  2. `PlayerPrefs.DeleteKey(keyName)` + `PlayerPrefs.Save()`
  3. `_cache.Remove(keyName)`
  4. If `cloudSync && _autoCloudSync`, call `await FirebaseHelper.DeleteDocumentAsync("preferences", keyName)`.

* **Usage Example**:

  ```csharp
  // Delete the "Score" preference both locally and in Firestore
  await PrefsHelper.Delete(GamePrefs.Score);

  // Delete "PlayerName" only locally, not in cloud
  await PrefsHelper.Delete(GamePrefs.PlayerName, cloudSync: false);
  ```

---

### Bulk Operations

#### `public static Dictionary<TEnum, object> GetAll<TEnum>() where TEnum : Enum`

* **Purpose**:
  Enumerate every enum member of `TEnum`, check if `PlayerPrefs.HasKey(keyName)` is true, and—if so—pull its value (via `GetLocalValue`) and return a dictionary mapping each `TEnum` to its stored object.

* **Returns**:
  A dictionary `Dictionary<TEnum, object>` of all currently‐set preferences (the raw object, not cast to a generic type).

* **How It Works**:

  1. Create an empty result dictionary.
  2. For each `TEnum key in Enum.GetValues(typeof(TEnum))`:
     a. `type = GetKeyType(key)`
     b. `keyName = key.ToString()`
     c. If `PlayerPrefs.HasKey(keyName)`, call `value = GetLocalValue(keyName, type, null)` (passing `defaultValue = null`).
     d. `result[key] = value`.
  3. Return `result`.

* **Usage Example**:

  ```csharp
  // Suppose we set Score=100, PlayerName="Alice", IsTutorialComplete=true
  var allPrefs = PrefsHelper.GetAll<GamePrefs>();
  foreach (var kvp in allPrefs)
  {
      Debug.Log($"Key: {kvp.Key}, Value: {kvp.Value}");
  }
  // Output might be:
  // "Key: Score, Value: 100"
  // "Key: PlayerName, Value: Alice"
  // "Key: IsTutorialComplete, Value: True"
  ```

---

#### `public static async Task SetBulk<TEnum>(Dictionary<TEnum, object> values, bool cloudSync = true) where TEnum : Enum`

* **Purpose**:
  Set multiple preference keys in one call. Each key/value pair calls `Set(...)` with `cloudSync: false` to avoid repeated Firestore writes; after the loop, if `cloudSync == true`, call `SyncAllToCloud()` once.

* **Parameters**:

  1. `Dictionary<TEnum, object> values`: The set of `(key, value)` pairs to set.
  2. `bool cloudSync`: If `true` (default), after setting every key locally, do one bulk push of the entire cache to Firestore via `SyncAllToCloud()`.

* **How It Works**:

  1. For each `kvp` in `values`: `await Set(kvp.Key, kvp.Value, false)` → Writes to local and cache, but does *not* push to Firestore.
  2. After the loop, if `cloudSync && _autoCloudSync`, call `await SyncAllToCloud()` to push every cached key in one by-key loop to Firestore.

* **Usage Example**:

  ```csharp
  var batchValues = new Dictionary<GamePrefs, object>
  {
      { GamePrefs.Score, 2500 },
      { GamePrefs.PlayerName, "Bob" },
      { GamePrefs.IsTutorialComplete, false },
      { GamePrefs.LastPosition, new Vector3(5, 10, 0) }
  };
  await PrefsHelper.SetBulk(batchValues);
  // → Writes all four keys to PlayerPrefs immediately
  // → Then does a single SyncAllToCloud, which pushes Score, PlayerName, IsTutorialComplete, LastPosition to Firestore
  ```

---

#### `public static async Task DeleteAll<TEnum>(bool cloudSync = true) where TEnum : Enum`

* **Purpose**:
  Delete every key defined in `TEnum` from local `PlayerPrefs` (and cache), then—if `cloudSync`—delete them all from Firestore as well.

* **How It Works**:

  1. For each `TEnum key in Enum.GetValues(typeof(TEnum))`, call `await Delete(key, false)` → deletes from PlayerPrefs and cache but does not push to Firestore.
  2. If `cloudSync && _autoCloudSync`, loop again and call `FirebaseHelper.DeleteDocumentAsync("preferences", keyName)` for every key.

* **Usage Example**:

  ```csharp
  // Locally delete Score, PlayerName, IsTutorialComplete, etc.
  // Then delete them all in Firestore
  await PrefsHelper.DeleteAll<GamePrefs>();

  // If you only want to delete locally and not touch cloud data:
  await PrefsHelper.DeleteAll<GamePrefs>(cloudSync: false);
  ```

---

### Cloud Sync

> **Note**: All Firestore operations assume you have already called `FirebaseHelper.InitializeAsync()` somewhere at startup so Firestore is ready.

#### `public static async Task SyncAllToCloud()`

* **Purpose**:
  Push every entry currently in `_cache` (which should mirror all local preferences you’ve accessed or set) into Firestore under `collection = "preferences"`, with a document name = preference key, contents:

  ```jsonc
  {
    "value": <value>,
    "timestamp": <DateTime.UtcNow>
  }
  ```

* **How It Works**:

  ```csharp
  foreach (var kvp in _cache)
      await SyncToCloud(kvp.Key, kvp.Value);
  _lastSyncTime = DateTime.Now;
  ```

  * `SyncToCloud(key, value)` builds a `Dictionary<string, object>` and calls `FirebaseHelper.SetDocumentAsync("preferences", key, data)`.

* **Usage Example**:

  ```csharp
  // Suppose the user pressed “Sync Now” in a settings UI:
  await PrefsHelper.SyncAllToCloud();
  Debug.Log("All local prefs pushed to Firestore.");
  ```

---

#### `public static async Task SyncFromCloud<TEnum>() where TEnum : Enum`

* **Purpose**:
  Pull the latest values from Firestore and overwrite local prefs/cache. For each key in `TEnum`, attempt to read Firestore document `preferences/{keyName}` as a `Dictionary<string, object>`. If it has `"value"`, call `Set(key, value, false)`—i.e. write to local and cache but do *not* push back to Firestore.

* **How It Works**:

  ```csharp
  foreach (TEnum key in Enum.GetValues(typeof(TEnum)))
  {
      string keyName = key.ToString();
      var data = await FirebaseHelper.GetDocumentAsync<Dictionary<string, object>>("preferences", keyName);
      if (data != null && data.TryGetValue("value", out object value))
      {
          await Set(key, value, false);
      }
  }
  _lastSyncTime = DateTime.Now;
  ```

* **Usage Example**:

  ```csharp
  // At app startup, after user logs in to Firebase:
  await PrefsHelper.SyncFromCloud<GamePrefs>();
  // Now every preference (Score, PlayerName, etc.) is updated locally from the cloud.
  ```

---

#### `private static async Task SyncToCloud(string key, object value)`

* **Purpose**:
  Internal helper that builds `{ "value": value, "timestamp": <now> }` and calls `FirebaseHelper.SetDocumentAsync("preferences", key, data)`.

* **Budget**:
  Because Firestore charges per document write, **SyncAllToCloud** will cost n writes if you have n keys in cache. Consider calling it sparingly (e.g. every 5 minutes, or only when user explicitly “Syncs”).

* **Internal Use Only** (not called directly by user code unless you expose it).

---

### JSON Integration

#### `public static async Task SetJson<T, TEnum>(TEnum key, T value, bool cloudSync = true) where TEnum : Enum`

* **Purpose**:
  Convenience method for storing an arbitrary serializable object `value` under `key` as a JSON string. Internally calls:

  1. `string json = JsonHelper.Serialize(value)`
  2. `await Set(key, json, cloudSync)`

* **Usage Example**:
  Suppose you have a custom class:

  ```csharp
  [Serializable]
  public class PlayerStats
  {
      public int level;
      public int experience;
      public List<string> achievements;
  }

  // Somewhere in your code:
  var stats = new PlayerStats { level = 5, experience = 2300, achievements = new List<string>{"FirstBlood","SharpShooter"} };
  await PrefsHelper.SetJson<PlayerStats, GamePrefs>(GamePrefs.ComplexData, stats);
  // → JSON is serialized, then stored under PlayerPrefs.SetString("ComplexData", "<json>"),
  //    and optionally pushed to Firestore.
  ```

#### `public static T GetJson<T, TEnum>(TEnum key, T defaultValue = default) where TEnum : Enum`

* **Purpose**:
  Convenience method for retrieving a JSON-serialized object stored under `key`, deserializing it back into `T`. Internally:

  1. `string json = Get<string, TEnum>(key, null)`
  2. If `json` is null or empty, return `defaultValue`.
  3. Else, `var dict = JsonHelper.DeserializeToDictionary(json)` → a `Dictionary<string, object>`
     *Warning*: The code attempts to cast that dictionary via `(T)Convert.ChangeType(dict, typeof(T))` but that only works if `T` is itself `Dictionary<string, object>` or a type with a matching layout.
     In practice, you may need to do:

     ```csharp
     T obj = JsonUtility.FromJson<T>(json);
     ```

     instead of the current implementation. (As written, `Convert.ChangeType` will fail if `T` is your custom class.)

* **Usage Example**:

  ```csharp
  // If you stored a PlayerStats object under ComplexData:
  PlayerStats stats = PrefsHelper.GetJson<PlayerStats, GamePrefs>(GamePrefs.ComplexData, new PlayerStats());
  ```

---

## Helper Methods (Internal)

These methods do the heavy lifting behind the scenes. You typically don’t call them directly, but it’s helpful to understand how they function.

### `private static string GetKeyType<TEnum>(TEnum key) where TEnum : Enum`

* **Purpose**:
  Examine the enum member’s `MemberInfo` and extract the `[Type("…")]` attribute value. Converts it to lowercase. If missing, throws `InvalidOperationException`.

* **Implementation**:

  ```csharp
  var memberInfo = key.GetType()
                      .GetMember(key.ToString())
                      .FirstOrDefault();

  var attribute = memberInfo?.GetCustomAttribute<TypeAttribute>();
  if (attribute == null)
      throw new InvalidOperationException($"Key {key} is missing TypeAttribute");
  return attribute.Type.ToLowerInvariant();
  ```

### `private static void SetLocalValue(string keyName, object value, string type)`

* **Purpose**:
  Given a string `keyName` and raw `value`, plus a `type` (one of `"int"`, `"float"`, `"string"`, `"bool"`, `"vector2"`, `"vector3"`, `"vector4"`, `"color"`, `"quaternion"`), write to `PlayerPrefs` and call `PlayerPrefs.Save()`.

* **Supported Cases**:

  ```csharp
  switch (type)
  {
      case "int":
          PlayerPrefs.SetInt(key, Convert.ToInt32(value));
          break;
      case "float":
          PlayerPrefs.SetFloat(key, Convert.ToSingle(value));
          break;
      case "string":
          PlayerPrefs.SetString(key, value.ToString());
          break;
      case "bool":
          PlayerPrefs.SetInt(key, Convert.ToBoolean(value) ? 1 : 0);
          break;
      case "vector2":
      case "vector3":
      case "vector4":
      case "color":
      case "quaternion":
          // Serialize the Unity type to JSON string
          PlayerPrefs.SetString(key, JsonUtility.ToJson(value));
          break;
      default:
          throw new ArgumentException($"Unsupported type: {type}");
  }
  PlayerPrefs.Save();
  ```

* **Examples**:

  ```csharp
  SetLocalValue("Score", 500, "int");           // calls PlayerPrefs.SetInt("Score", 500);
  SetLocalValue("MusicVolume", 0.75f, "float");
  SetLocalValue("PlayerName", "Alice", "string");
  SetLocalValue("IsTutorialComplete", true, "bool");
  SetLocalValue("LastPosition", new Vector3(1,2,3), "vector3");  // PlayerPrefs.SetString("LastPosition", "{\"x\":1,\"y\":2,\"z\":3}");
  ```

### `private static object GetLocalValue(string keyName, string type, object defaultValue)`

* **Purpose**:
  Read a value from `PlayerPrefs` (if present) and return it as an `object` of the correct underlying type. If the key is missing, return `defaultValue`.

* **Supported Cases**:

  ```csharp
  if (!PlayerPrefs.HasKey(keyName))
      return defaultValue;

  switch (type)
  {
      case "int":
          return PlayerPrefs.GetInt(keyName, Convert.ToInt32(defaultValue));
      case "float":
          return PlayerPrefs.GetFloat(keyName, Convert.ToSingle(defaultValue));
      case "string":
          return PlayerPrefs.GetString(keyName, defaultValue?.ToString());
      case "bool":
          return PlayerPrefs.GetInt(keyName, Convert.ToBoolean(defaultValue) ? 1 : 0) == 1;
      case "vector2":
          return JsonUtility.FromJson<Vector2>(PlayerPrefs.GetString(keyName));
      case "vector3":
          return JsonUtility.FromJson<Vector3>(PlayerPrefs.GetString(keyName));
      case "vector4":
          return JsonUtility.FromJson<Vector4>(PlayerPrefs.GetString(keyName));
      case "color":
          return JsonUtility.FromJson<Color>(PlayerPrefs.GetString(keyName));
      case "quaternion":
          return JsonUtility.FromJson<Quaternion>(PlayerPrefs.GetString(keyName));
      default:
          throw new ArgumentException($"Unsupported type: {type}");
  }
  ```

* **Examples**:
  If you previously did:

  ```csharp
  SetLocalValue("Score", 100, "int");
  SetLocalValue("LastPosition", new Vector3(5,6,7), "vector3");
  ```

  Then:

  ```csharp
  object scoreObj = GetLocalValue("Score", "int", 0);           // returns boxed int 100
  object posObj   = GetLocalValue("LastPosition", "vector3", null); // returns Vector3(5,6,7)
  ```

---

## Real-World Usage Examples

Below are step-by-step scenarios showing how you might integrate **PrefsHelper** into your Unity project.

### 1. Storing & Retrieving Primitive Types

1. **Define your enum** (already done above):

   ```csharp
   public enum GamePrefs
   {
       [Type("int")]    Score,
       [Type("string")] PlayerName,
       [Type("bool")]   IsTutorialComplete
       // … etc.
   }
   ```

2. **Set & Get an integer**:

   ```csharp
   public class ScoreManager : MonoBehaviour
   {
       async void Start()
       {
           // 1. Set the player's score to 2000, local + cloud (by default)
           await PrefsHelper.Set(GamePrefs.Score, 2000);

           // 2. Later, read it back (with default = 0 if not set)
           int savedScore = PrefsHelper.Get<int, GamePrefs>(GamePrefs.Score, 0);
           Debug.Log("Loaded Score: " + savedScore);
       }
   }
   ```

3. **Set & Get a bool**:

   ```csharp
   public class TutorialManager : MonoBehaviour
   {
       public void CompleteTutorial()
       {
           // Mark tutorial as done
           PrefsHelper.Set(GamePrefs.IsTutorialComplete, true).Forget(); 
           // .Forget() is my hypothetical extension to ignore the Task—
           // or simply do `await PrefsHelper.Set(...)` if calling from an async method
       }

       private void Update()
       {
           // Each frame, check if tutorial is already complete:
           bool done = PrefsHelper.Get<bool, GamePrefs>(GamePrefs.IsTutorialComplete, false);
           if (!done)
           {
               // Show tutorial prompt…
           }
       }
   }
   ```

4. **Set & Get a string**:

   ```csharp
   public class LoginScreen : MonoBehaviour
   {
       public UnityEngine.UI.InputField nameInput;

       public async void OnLoginButton()
       {
           string userName = nameInput.text;
           if (!string.IsNullOrEmpty(userName))
           {
               await PrefsHelper.Set(GamePrefs.PlayerName, userName);
               Debug.Log("PlayerName stored: " + userName);
           }
       }

       private void Start()
       {
           string savedName = PrefsHelper.Get<string, GamePrefs>(GamePrefs.PlayerName, "Guest");
           nameInput.text = savedName;
       }
   }
   ```

---

### 2. Storing & Retrieving Unity Types (`Vector3`, `Color`)

Because `PrefsHelper` serializes Unity types to JSON under the hood, you can do:

1. **Store a `Vector3`**:

   ```csharp
   public class CheckpointSystem : MonoBehaviour
   {
       public void SaveCheckpoint(Vector3 playerPosition)
       {
           // Stores JSON string like "{\"x\":1.23,\"y\":4.56,\"z\":7.89}"
           PrefsHelper.Set(GamePrefs.LastPosition, playerPosition).Forget();
       }
   }
   ```

2. **Retrieve a `Vector3`**:

   ```csharp
   public class PlayerSpawner : MonoBehaviour
   {
       private void Start()
       {
           Vector3 lastPos = PrefsHelper.Get<Vector3, GamePrefs>(GamePrefs.LastPosition, Vector3.zero);
           transform.position = lastPos;
       }
   }
   ```

3. **Store a `Color`**:

   ```csharp
   public class UIManager : MonoBehaviour
   {
       public Color uiTint;

       public void SaveUIColor()
       {
           // uiTint might be (0.1, 0.2, 0.3, 1.0)
           PrefsHelper.Set(GamePrefs.UIColor, uiTint).Forget();
       }
   }
   ```

4. **Retrieve a `Color`**:

   ```csharp
   public class UIManager : MonoBehaviour
   {
       private void Awake()
       {
           Color saved = PrefsHelper.Get<Color, GamePrefs>(GamePrefs.UIColor, Color.white);
           ApplyTint(saved);
       }

       private void ApplyTint(Color c)
       {
           // apply c to your UI Canvas, etc.
       }
   }
   ```

> **Note**: Under the hood, `SetLocalValue("UIColor", new Color(r,g,b,a), "color")` calls `JsonUtility.ToJson(color)` and saves that string. Then `GetLocalValue("UIColor", "color", defaultValue)` calls `JsonUtility.FromJson<Color>(jsonString)` to reconstruct the `Color`.

---

### 3. Complex-Object via JSON

If you have a custom data class that you want to store, use `SetJson` / `GetJson`.

1. **Create a Serializable Class**:

   ```csharp
   [Serializable]
   public class PlayerStats
   {
       public int level;
       public int experience;
       public List<string> achievements;
   }
   ```

2. **Store It**:

   ```csharp
   public class StatsManager : MonoBehaviour
   {
       public async void SaveStats(PlayerStats stats)
       {
           await PrefsHelper.SetJson<PlayerStats, GamePrefs>(GamePrefs.ComplexData, stats);
       }
   }
   ```

3. **Retrieve It**:

   ```csharp
   public class StatsManager : MonoBehaviour
   {
       public PlayerStats LoadStats()
       {
           PlayerStats defaultStats = new PlayerStats() { level = 1, experience = 0, achievements = new List<string>() };
           var savedStats = PrefsHelper.GetJson<PlayerStats, GamePrefs>(GamePrefs.ComplexData, defaultStats);
           return savedStats;
       }
   }
   ```

> **Important Note**: The built-in `GetJson` implementation uses
>
> ```csharp
> var dict = JsonHelper.DeserializeToDictionary(json);
> return (T)Convert.ChangeType(dict, typeof(T));
> ```
>
> which only works if `T` is itself `Dictionary<string,object>`. To deserialize back into your custom `PlayerStats`, you’ll need to modify `GetJson` to use:
>
> ```csharp
> return JsonUtility.FromJson<T>(json);
> ```
>
> rather than `Convert.ChangeType`. Keep that in mind if your class is nontrivial.

---

### 4. Bulk Set/Delete

1. **Set Multiple Keys in One Call**:

   ```csharp
   public class BulkExample : MonoBehaviour
   {
       public async void SubmitAllPrefs()
       {
           var batch = new Dictionary<GamePrefs, object>
           {
               { GamePrefs.Score, 5000 },
               { GamePrefs.PlayerName, "Charlie" },
               { GamePrefs.IsTutorialComplete, true },
               { GamePrefs.LastPosition, new Vector3(10f, 2f, -5f) }
           };

           await PrefsHelper.SetBulk(batch);
           Debug.Log("All preferences set and pushed to cloud.");
       }
   }
   ```

2. **Delete All Keys**:

   ```csharp
   public class WipeData : MonoBehaviour
   {
       public async void OnWipeButtonPressed()
       {
           await PrefsHelper.DeleteAll<GamePrefs>();
           Debug.Log("All preferences locally deleted and removed from cloud.");
       }
   }
   ```

---

### 5. Manual Cloud Sync / Auto Sync

By default, `PrefsHelper._autoCloudSync = true` and every call to `Set(...)` or `Delete(...)` pushes that single key to Firestore. If you’d prefer to:

1. **Disable Automatic Cloud Sync**:
   Simply pass `cloudSync: false` to each `Set`/`Delete`, or set `_autoCloudSync = false` (if you modify the code). Then you can perform a manual bulk sync:

   ```csharp
   public async void ManualSyncButton()
   {
       await PrefsHelper.SyncAllToCloud();
       Debug.Log("Manual: All cached prefs pushed to Firestore.");
   }
   ```

2. **Pull Only Once at Startup**:

   ```csharp
   public class GameStartup : MonoBehaviour
   {
       async void Awake()
       {
           bool success = await FirebaseHelper.InitializeAsync();
           if (success)
           {
               await PrefsHelper.SyncFromCloud<GamePrefs>();
               Debug.Log("Pulled preferences from Firestore.");
           }
       }
   }
   ```

3. **Periodic Sync**:

   ```csharp
   public class AutoSyncManager : MonoBehaviour
   {
       private async void Update()
       {
           if (DateTime.Now - PrefsHelper.LastSyncTime > TimeSpan.FromMinutes(5))
           {
               await PrefsHelper.SyncFromCloud<GamePrefs>();
               Debug.Log("Periodic cloud sync complete.");
           }
       }
   }
   ```

   *(Note: `LastSyncTime` is a private field. You could expose it or mirror it via a public property if you want external code to read it.)*

---

### 6. Migration Scenario

Imagine you add a new key to your `GamePrefs` enum:

```csharp
public enum GamePrefs
{
    [Type("int")] Score,
    [Type("string")] PlayerName,
    [Type("bool")] IsTutorialComplete,

    // NEW in version 2.0:
    [Type("float")] MasterVolume,

    [Type("vector3")] LastPosition,
    [Type("color")] UIColor,
    [Type("string")] ComplexData
}
```

Because you’re using `enum` members, any existing PlayerPrefs under the old keys (`"Score"`, `"PlayerName"`, etc.) remain intact. The new key `MasterVolume` simply won’t exist in `PlayerPrefs` or cloud until someone sets it.

If you need to **migrate** old integer volume (key `"Volume"`) into the new `MasterVolume` float, you could write a migration method:

```csharp
public static async Task MigrateOldVolume()
{
    // Suppose old code used "Volume" as int in PlayerPrefs
    if (PlayerPrefs.HasKey("Volume"))
    {
        int oldVol = PlayerPrefs.GetInt("Volume");
        float newVol = oldVol / 100f; // convert to 0.0–1.0
        await PrefsHelper.Set(GamePrefs.MasterVolume, newVol);

        PlayerPrefs.DeleteKey("Volume");
        PlayerPrefs.Save();
    }
}
```

You could call that at startup to automatically bring over any old data into your new system.

---

## Design Notes & Tips

1. **Mixing Local & Cloud**

   * If you are offline or Firestore is unreachable, `Set(...)` will still write to `PlayerPrefs` and cache; the Firestore write will fail (logged by `FirebaseHelper`). You may want to catch exceptions around `SyncToCloud` or wrap `await Set(...)` in a `try`/`catch`.

2. **Caching Behavior**

   * Once you call `Get(...)`, the returned value is saved in `_cache`. Subsequent `Get(...)` calls don’t hit `PlayerPrefs` again (until you delete or reset that key).
   * If you call `SyncFromCloud` and it writes into local via `Set(key, value, false)`, the cache is updated automatically.

3. **JSON Deserialization for Complex Objects**

   * As noted earlier, the default `GetJson<T>` uses `Convert.ChangeType` on a `Dictionary<string, object>`, which only works if `T` is a dictionary.
   * To truly support arbitrary `T`, replace:

     ```csharp
     var dict = JsonHelper.DeserializeToDictionary(json);
     return (T)Convert.ChangeType(dict, typeof(T));
     ```

     with:

     ```csharp
     return JsonUtility.FromJson<T>(json);
     ```

     so Unity’s JSON utility directly reconstructs `T`.

4. **Type Support**

   * Out of the box, you can store:

     * `"int"`, `"float"`, `"string"`, `"bool"`
     * `"vector2"`, `"vector3"`, `"vector4"`, `"color"`, `"quaternion"`.
   * If you need `Rect`, `Bounds`, or a custom struct, you can extend `SetLocalValue`/`GetLocalValue` to handle them via `JsonUtility.ToJson`/`.FromJson<Rect>`, etc.

5. **Encryption Layer**

   * If you need to store sensitive data (API tokens, user credentials), modify `SetLocalValue` to encrypt the JSON string before calling `PlayerPrefs.SetString(...)` and decrypt in `GetLocalValue`.

6. **Automatic Sync Interval**

   * The code has `_syncInterval = TimeSpan.FromMinutes(5)` and `_lastSyncTime`. You could create a background coroutine that triggers `SyncAllToCloud()` once every `_syncInterval`. Currently, that logic is not implemented—only a placeholder.

7. **Error Handling**

   * All async methods that call Firestore swallow exceptions (by logging), so your code should check return values of `FirebaseHelper.SetDocumentAsync` if needed.

8. **Deleting a Key vs. Resetting to Default**

   * `Delete(key)` truly removes the key from `PlayerPrefs`. If you just want to reset to a default, call `Set(key, defaultValue)` instead.

9. **Naming Conventions**

   * The enum member name (e.g. `Score`) becomes the actual `PlayerPrefs` key. If you want a different key string (e.g. `"player_score"`), you can override by using custom attributes or by changing the code to read a separate `[Key("player_score")]` attribute. As written, it always uses `key.ToString()`.

---

## Summary of Public Members

### Core

* **`Task Set<TEnum>(TEnum key, object value, bool cloudSync = true)`**
  Store a typed value under `key`. Synchronously writes to `PlayerPrefs`, updates cache, and (optionally) pushes to Firestore.

* **`T Get<T, TEnum>(TEnum key, T defaultValue = default)`**
  Retrieve a value of type `T` from local cache or `PlayerPrefs`. Returns `defaultValue` if missing.

* **`Task Delete<TEnum>(TEnum key, bool cloudSync = true)`**
  Remove `key` from `PlayerPrefs`, cache, and (optionally) Firestore.

### Bulk

* **`Dictionary<TEnum, object> GetAll<TEnum>()`**
  Fetch all currently‐set preferences (based on `PlayerPrefs.HasKey`) into a dictionary.

* **`Task SetBulk<TEnum>(Dictionary<TEnum, object> values, bool cloudSync = true)`**
  Set multiple keys in one go; optionally push all changes to Firestore at end.

* **`Task DeleteAll<TEnum>(bool cloudSync = true)`**
  Delete every key in `TEnum` from local and (optionally) cloud.

### Cloud Sync

* **`Task SyncAllToCloud()`**
  Push every cached key/value to Firestore under collection `"preferences"`.

* **`Task SyncFromCloud<TEnum>()`**
  Pull every `TEnum` key from Firestore and overwrite local `PlayerPrefs` and cache.

### JSON Integration

* **`Task SetJson<T, TEnum>(TEnum key, T value, bool cloudSync = true)`**
  Serialize `value` to JSON, then call `Set(key, json)`.

* **`T GetJson<T, TEnum>(TEnum key, T defaultValue = default)`**
  Read a JSON string from `Get<string>(key)`, deserialize to `T`, or return `defaultValue` on error.

---

With this reference and the examples above, you should be able to:

1. **Define** an enum listing all your game preferences, each annotated with `[Type("<…>")]`.
2. **Read/Write** primitives (`int`, `float`, `bool`, `string`) and Unity types (`Vector3`, `Color`, etc.) using `Set(...)` / `Get<…>` without worrying about serialization details.
3. **Push/Pull** values automatically to Firestore for cross-device sync, or do manual bulk syncs.
4. **Work with** complex data structures by serializing them to JSON.
5. **Perform bulk operations** or completely wipe all preferences if needed.
6. **Extend** for encrypted storage, additional types, or migration logic.

Happy coding!
