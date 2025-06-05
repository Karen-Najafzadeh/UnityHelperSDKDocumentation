Below is a complete reference for **LocalStorageHelper**, a static utility designed to simplify local data persistence, optional encryption, automatic Firebase cloud synchronization, and basic AssetBundle caching. It covers:

1. **Initialization**
2. **Local Storage Operations**
3. **Cloud Sync Integration**
4. **Asset Bundle Integration**
5. **Utilities (Encryption, File I/O, Cache Management)**
6. **Settings Management**
7. **Real-World Usage Examples**

---

## 1. Overview & Purpose

**LocalStorageHelper** centralizes several concerns into one API:

1. **Local Data Persistence**

   * Stores arbitrary data as JSON files under `Application.persistentDataPath/SaveData`.
   * Optionally encrypts sensitive data (AES) before writing to disk.

2. **In-Memory Caching**

   * Keeps deserialized objects in a dictionary (`DataCache`) to avoid repeated disk reads.

3. **Automatic Cloud Sync**

   * After saving locally, can push the same data to a Firestore path: `userdata/{Application.identifier}/{key}`.
   * Can pull down data from Firestore and overwrite local JSON files.

4. **Asset Bundle Metadata Caching**

   * Provides methods to cache an asset bundle’s metadata (version, size, cache date).
   * Helps determine if a bundle is outdated compared to a “latest version” string.

5. **Encryption/Decryption**

   * Uses AES encryption with a static key (`EncryptionKey`) for any string you choose to encrypt.

6. **Auto-Migration Hooks**

   * By virtue of writing JSON files, you can manage versioned schemas or rename keys over time.

7. **Settings Management**

   * Customize whether cloud sync occurs automatically and at what interval.
   * Clear in-memory cache and optionally delete all saved JSON files.

---

## 2. Initialization

### `public static void Initialize()`

**Purpose**
Set up the local storage folder (`SaveData`) and preload any existing JSON files into the in-memory cache.

**What It Does Internally**

1. Checks if the folder path

   ```
   Application.persistentDataPath/SaveData
   ```

   exists. If not, it creates it.
2. Calls `LoadAllLocalData()` to scan for all `*.json` files in that folder. For each file:

   * Reads the full JSON text.
   * Deserializes it into a `Dictionary<string, object>` via `JsonHelper.DeserializeToDictionary(json)`.
   * Stores that dictionary under `DataCache[key]`, where `key` = filename without extension.

**Usage Example**

```csharp
public class GameInitializer : MonoBehaviour
{
    private void Awake()
    {
        // Ensure the SaveData folder exists, and preload any existing files into DataCache.
        LocalStorageHelper.Initialize();
    }
}
```

---

## 3. Local Storage Operations

### 3.1 Saving Data Locally

#### `public static async Task<bool> SaveDataAsync<T>(string key, T data, bool encrypt = false, bool cloudSync = true)`

**Purpose**
Serialize an arbitrary object of type `T` to JSON, optionally encrypt it, then write it to a file `SaveData/{key}.json`. Also updates the in-memory cache and—if requested—pushes to Firebase.

**Parameters**

1. `string key`

   * A unique string identifier. Corresponds to the filename (without “.json”) and the Firestore document ID when syncing.
   * E.g. `"player_profile"`, `"settings"`, `"level_data"`.

2. `T data`

   * The object to persist. Can be any serializable type (plain C# classes marked `[Serializable]`, collections, primitives, etc.).

3. `bool encrypt` (default `false`)

   * If `true`, the JSON string will be AES-encrypted (see EncryptionKey) before writing to disk.
   * Decryption is required when reading it back.

4. `bool cloudSync` (default `true`)

   * If `true` and `AutoCloudSync == true`, after saving locally, the same data is pushed to Firebase using `SyncToCloudAsync(key, data)`.

**Return Value**

* `Task<bool>`: resolves to `true` if save (and optional cloud sync) succeeded; `false` on any exception.

**How It Works**

1. Serialize `data` to JSON:

   ```csharp
   string json = JsonHelper.Serialize(data);
   ```
2. If `encrypt` is `true`, replace `json` with `EncryptString(json)`.
3. Store `data` in the in-memory dictionary:

   ```csharp
   DataCache[key] = data;
   ModifiedKeys.Add(key);
   ```
4. Compute file path:

   ```csharp
   string filePath = Path.Combine(Application.persistentDataPath, "SaveData", $"{key}.json");
   ```
5. Write the (possibly-encrypted) JSON to disk:

   ```csharp
   await File.WriteAllTextAsync(filePath, json);
   ```
6. If `cloudSync && AutoCloudSync`, call `await SyncToCloudAsync(key, data)`.

**Real-World Example**

```csharp
public class ProfileManager : MonoBehaviour
{
    [Serializable]
    private class PlayerProfile
    {
        public string playerName;
        public int level;
        public int experience;
    }

    private async void SaveProfile()
    {
        var profile = new PlayerProfile
        {
            playerName = "Alice",
            level = 7,
            experience = 2300
        };

        bool success = await LocalStorageHelper.SaveDataAsync("player_profile", profile,
                                                              encrypt: true,    // encrypt on disk
                                                              cloudSync: true   // also push to Firestore
                                                             );
        if (success)
            Debug.Log("Profile saved locally and synced to cloud.");
        else
            Debug.LogError("Failed to save profile.");
    }
}
```

---

### 3.2 Loading Data Locally

#### `public static async Task<T> LoadDataAsync<T>(string key, bool encrypted = false, bool useCache = true)`

**Purpose**
Read a JSON file `SaveData/{key}.json` from disk, optionally decrypt it, deserialize it into an object of type `T`, update the cache, and return the result. If `useCache == true` and that `key` is already in `DataCache`, return the cached object immediately without touching the file system.

**Parameters**

1. `string key`

   * Corresponds to the filename (without “.json”).

2. `bool encrypted` (default `false`)

   * If `true`, after reading the file’s contents, pass through `DecryptString(...)`.

3. `bool useCache` (default `true`)

   * If `true` and `DataCache` already has an entry for this `key`, simply cast and return that object rather than reloading from disk.

**Return Value**

* `Task<T>`: the deserialized object of type `T`. If the file does not exist (or any error occurs), returns `default(T)` (e.g. `null` for reference types, `0` for numerics).

**How It Works**

1. If `useCache` and `DataCache.TryGetValue(key, out object cachedData)` succeeds, cast to `T` and return.
2. Compute file path:

   ```csharp
   string filePath = Path.Combine(Application.persistentDataPath, "SaveData", $"{key}.json");
   ```
3. If `!File.Exists(filePath)`, return `default(T)`.
4. `string json = await File.ReadAllTextAsync(filePath)`.
5. If `encrypted == true`, do `json = DecryptString(json)`.
6. Deserialize using Unity’s `JsonUtility`:

   ```csharp
   T data = JsonUtility.FromJson<T>(json);
   ```
7. Store `DataCache[key] = data` and return `data`.

**Real-World Example**

```csharp
public class ProfileManager : MonoBehaviour
{
    private async void LoadProfile()
    {
        var profile = await LocalStorageHelper.LoadDataAsync<PlayerProfile>("player_profile", 
                                                                            encrypted: true, 
                                                                            useCache: true);
        if (profile != null)
        {
            Debug.Log($"Welcome back, {profile.playerName}! Level: {profile.level}");
        }
        else
        {
            Debug.Log("No saved profile found.");
        }
    }
}
```

---

## 4. Cloud Sync Integration

> **Note**: All cloud operations rely on `FirebaseHelper` being initialized and authenticated elsewhere in your app. In valid usage, call something like:
>
> ```csharp
> await FirebaseHelper.InitializeAsync();
> LocalStorageHelper.Initialize();
> ```

### 4.1 Pushing Local Data to Cloud

#### `private static async Task<bool> SyncToCloudAsync<T>(string key, T data)`

**Purpose**
Upload `data` under Firestore document path:

```
collection: "userdata/{Application.identifier}"
document ID: key
content: { "value": <data>, "timestamp": <UTC now> }
```

This only happens if at least `SyncInterval` time has passed since `LastSyncTime`.

**Parameters**

1. `string key`

   * Document ID in Firestore.

2. `T data`

   * The object to push. `FirebaseHelper.SetDocumentAsync(...)` will internally convert it to a `Dictionary<string, object>` via `JsonHelper.Serialize` and `JsonHelper.DeserializeToDictionary`, then call Firestore `.SetAsync(...)`.

**Return Value**

* `Task<bool>`: `true` if cloud write succeeded or if sync is skipped due to interval not yet passed. Returns `false` on exception or failed Firestore call.

**Behavior**

1. If `(DateTime.Now - LastSyncTime) < SyncInterval`, return `true` immediately (skip to avoid excessive writes).
2. Else, call:

   ```csharp
   bool success = await FirebaseHelper.SetDocumentAsync($"userdata/{Application.identifier}", key, data);
   ```
3. If `success`, update `LastSyncTime = DateTime.Now` and return `true`.
4. On exception or `success == false`, log and return `false`.

**Real-World Example**

```csharp
public class SettingsManager : MonoBehaviour
{
    private async void SaveSettings(GameSettings settings)
    {
        // This will only push to Firestore if 5+ minutes have elapsed
        bool synced = await LocalStorageHelper.SaveDataAsync("settings", settings, encrypt: false, cloudSync: true);
        if (synced)
            Debug.Log("Settings saved locally and (possibly) to cloud.");
        else
            Debug.LogError("Failed to sync settings to cloud.");
    }
}
```

---

### 4.2 Pulling Data from Cloud

#### `public static async Task<bool> PullFromCloudAsync(string key)`

**Purpose**
Fetch Firestore document:

```
collection: "userdata/{Application.identifier}"
document ID: key
```

Deserialize its data to a `Dictionary<string, object>`, then write that dictionary to the local JSON file (overwriting). Finally, store that dictionary in `DataCache`.

**Parameters**

1. `string key`

   * The Firestore document ID to fetch. Also corresponds to the local JSON file name.

**Return Value**

* `Task<bool>`:

  * `true` if the document existed and was pulled & saved
  * `false` if document did not exist or on any error.

**How It Works**

1. Call:

   ```csharp
   var cloudData = await FirebaseHelper.GetDocumentAsync<Dictionary<string, object>>(
                       $"userdata/{Application.identifier}", key);
   ```
2. If `cloudData == null`, return `false`.
3. Convert that `Dictionary<string, object>` back to a JSON string:

   ```csharp
   string json = JsonHelper.Serialize(cloudData);
   ```
4. Write `json` to local file `SaveData/{key}.json` (via `await File.WriteAllTextAsync(...)`).
5. Update `DataCache[key] = cloudData`.
6. Return `true`.

**Real-World Example**

```csharp
public class ProgressManager : MonoBehaviour
{
    private async void SyncProgress()
    {
        bool pulled = await LocalStorageHelper.PullFromCloudAsync("player_progress");
        if (pulled)
            Debug.Log("Progress pulled from cloud and saved locally.");
        else
            Debug.Log("No remote progress found (or error).");
    }
}
```

---

## 5. Asset Bundle Integration

These methods track metadata about any AssetBundle you load via `AssetBundleManager`, for simple “is outdated?” checks.

### 5.1 Caching an Asset Bundle

#### `public static async Task<bool> CacheAssetBundleAsync(string bundleName, string version)`

**Purpose**

1. Ensure that the requested AssetBundle (by `bundleName`) is loaded (via `AssetBundleManager`).
2. Record metadata—`version` (string), `CacheDate` (now), and `Size` (in bytes)—in a small helper class `BundleMetadata`.
3. Save that metadata under local JSON key:

   ```
   "bundle_{bundleName}_meta"
   ```

**Parameters**

1. `string bundleName`

   * Name of the bundle (matching what `AssetBundleManager` expects).

2. `string version`

   * A version tag (e.g. `"1.2.3"`, `"rev_A1"`) that you compare later to see if a newer bundle is available.

**Return Value**

* `Task<bool>`:

  * `true` on success (bundle loaded + metadata saved locally),
  * `false` on error (e.g. bundle failed to load).

**How It Works**

1. Call `var bundle = await AssetBundleManager.LoadBundleAsync(bundleName)`. If that returns `null`, return `false`.
2. Build `BundleMetadata metadata = new BundleMetadata { Version = version, CacheDate = DateTime.UtcNow, Size = <file length> }`.

   * The file length is fetched via `new FileInfo(AssetBundleManager.GetBundleFilePath(bundleName)).Length`.
3. Call `await SaveDataAsync($"bundle_{bundleName}_meta", metadata)`—no encryption by default.
4. Return `true`.

**Example**

```csharp
public class BundleManager : MonoBehaviour
{
    private async void DownloadAndCacheBundles()
    {
        string bundleName = "environment_textures";
        string latestVersion = "2.0.0";

        bool success = await LocalStorageHelper.CacheAssetBundleAsync(bundleName, latestVersion);
        if (success)
            Debug.Log($"Successfully cached {bundleName} v{latestVersion}.");
        else
            Debug.LogError($"Failed to cache bundle {bundleName}.");
    }
}
```

---

### 5.2 Checking if a Bundle Is Outdated

#### `public static async Task<bool> IsBundleOutdatedAsync(string bundleName, string latestVersion)`

**Purpose**
Compare the locally cached bundle metadata’s `Version` against a new `latestVersion` string. If they differ (or no metadata exists), return `true` (meaning you should re-download the bundle). Otherwise, return `false`.

**Parameters**

1. `string bundleName`
2. `string latestVersion`

**Return Value**

* `Task<bool>`:

  * `true` if (a) no metadata found or (b) `metadata.Version != latestVersion`
  * `false` if metadata exists and `metadata.Version == latestVersion`

**How It Works**

1. Call `var metadata = await LoadDataAsync<BundleMetadata>($"bundle_{bundleName}_meta")`.
2. If `metadata == null`, return `true` (no local cache).
3. If `metadata.Version != latestVersion`, return `true`.
4. Else return `false`.

**Example**

```csharp
public class BundleManager : MonoBehaviour
{
    private async void CheckBundles()
    {
        string bundleName = "environment_textures";
        string latestVersion = "2.1.0";

        bool isOutdated = await LocalStorageHelper.IsBundleOutdatedAsync(bundleName, latestVersion);
        if (isOutdated)
        {
            Debug.Log($"{bundleName} is outdated. Downloading new version...");
            // ... trigger a re-download via AssetBundleManager, then call CacheAssetBundleAsync again
        }
        else
        {
            Debug.Log($"{bundleName} is up-to-date.");
        }
    }
}
```

---

## 6. Utilities

### 6.1 File Path Helper

#### `private static string GetLocalFilePath(string key)`

**Purpose**
Compute the full path on disk for a given `key`, namely:

```
<Application.persistentDataPath>/SaveData/{key}.json
```

**Returns**
A string containing the absolute path.

**Example**

```csharp
string filePath = LocalStorageHelper.GetLocalFilePath("player_profile");
// Might return "/Users/username/Library/Application Support/MyGame/SaveData/player_profile.json"
```

---

### 6.2 Encryption / Decryption

LocalStorageHelper uses a symmetric AES key to encrypt any JSON string you choose. The same key is used to decrypt.

#### `private static string EncryptString(string text)`

**Purpose**
Encrypt an input UTF-8 string using AES (128-bit or 256-bit depending on `EncryptionKey.Length`) in CBC mode with a random IV. Returns a Base64 string that concatenates IV + ciphertext.

**What It Does**

1. Create an `Aes` instance and set `aes.Key = EncryptionKey`.
2. Generate a random `aes.IV`.
3. Create an `ICryptoTransform encryptor = aes.CreateEncryptor()`.
4. Convert `text` → UTF-8 bytes.
5. `encrypted = encryptor.TransformFinalBlock(textBytes, 0, textBytes.Length)`.
6. Allocate `result = new byte[aes.IV.Length + encrypted.Length]` and copy `IV` first, then `encrypted`.
7. Return `Convert.ToBase64String(result)`.

On error, logs and returns the original plaintext.

**Example**

```csharp
string plain = "{\"playerName\":\"Alice\",\"level\":5}";
string encrypted = LocalStorageHelper.EncryptString(plain);
Debug.Log($"Encrypted: {encrypted}");
// Later: DecryptString(encrypted) returns the original JSON.
```

---

#### `private static string DecryptString(string encryptedText)`

**Purpose**
Reverse of `EncryptString`: expects a Base64 string that contains `[ IV ][ ciphertext ]`. Splits out the first `aes.IV.Length` bytes as IV, uses the same `EncryptionKey`, decrypts the remainder, and returns the UTF-8 plaintext.

**What It Does**

1. `byte[] fullData = Convert.FromBase64String(encryptedText)`.
2. Create an `Aes` instance, set `aes.Key = EncryptionKey`.
3. Extract `iv = fullData[0..aes.IV.Length]`.
4. Extract `encrypted = fullData[aes.IV.Length..]`.
5. `aes.IV = iv`.
6. `decryptor = aes.CreateDecryptor()`.
7. `decrypted = decryptor.TransformFinalBlock(encrypted, 0, encrypted.Length)`.
8. Return `UTF8.GetString(decrypted)`.

On exception, logs and returns the original `encryptedText`.

**Example**

```csharp
string decrypted = LocalStorageHelper.DecryptString(encrypted);
Debug.Log($"Decrypted JSON: {decrypted}");
// Should match the original JSON string that was passed to EncryptString.
```

---

### 6.3 Loading All Local Data into Cache

#### `private static void LoadAllLocalData()`

**Purpose**
Scan `SaveData/*.json` folder and, for each JSON file found, read its contents, parse via `JsonHelper.DeserializeToDictionary(json)`, and store under `DataCache[key]` (where `key` = filename without extension).

**How It Works**

1. `string[] files = Directory.GetFiles(LocalSavePath, "*.json")`.
2. For each `file` in `files`:

   * `string key = Path.GetFileNameWithoutExtension(file)`.
   * `string json = File.ReadAllText(file)`.
   * `var dict = JsonHelper.DeserializeToDictionary(json)`.
   * `DataCache[key] = dict`.

This means that on initialization, every existing JSON file’s top-level structure is stored in cache as a `Dictionary<string, object>`. If your saved files represent more complex classes, you’ll need to cast or re-deserialize in `LoadDataAsync`.

**Example**
Call `LocalStorageHelper.Initialize()` at startup, which invokes this method.

---

## 7. Settings Management

### 7.1 ConfigureSync

#### `public static void ConfigureSync(bool autoSync, TimeSpan? interval = null)`

**Purpose**
Adjust the helper’s behavior for automatic cloud synchronization. When `autoSync = false`, calls to `SaveDataAsync(..., cloudSync: true)` will still skip cloud sync because `AutoCloudSync` is now `false`. Optionally change the `SyncInterval` (minimum elapsed time between two consecutive Firestore writes).

**Parameters**

1. `bool autoSync`

   * If `true`, local saves may trigger cloud sync (depending on interval).
   * If `false`, no calls to `SyncToCloudAsync` will occur.

2. `TimeSpan? interval` (optional)

   * If non-null, sets the new `SyncInterval`. Default is 5 minutes.

**Usage Example**

```csharp
public class SyncConfigurator : MonoBehaviour
{
    private void Start()
    {
        // Disable automatic cloud syncing; we’ll do it manually.
        LocalStorageHelper.ConfigureSync(autoSync: false);

        // Alternatively, sync at most once every 15 minutes:
        LocalStorageHelper.ConfigureSync(autoSync: true, interval: TimeSpan.FromMinutes(15));
    }
}
```

---

### 7.2 ClearCache

#### `public static void ClearCache(bool deleteFiles = false)`

**Purpose**
Empty the in-memory `DataCache` and `ModifiedKeys` sets. If `deleteFiles == true`, also delete all `*.json` under `SaveData` (and recreate the folder).

**Parameters**

* `bool deleteFiles` (default `false`):

  * If `true`, physically remove the entire `SaveData` directory and recreate it empty.
  * If `false`, only clears memory, leaving files on disk.

**Usage Example**

```csharp
public class ResetSystem : MonoBehaviour
{
    public void OnUserRequestsFactoryReset()
    {
        // Clear all cached and on-disk data:
        LocalStorageHelper.ClearCache(deleteFiles: true);
        Debug.Log("All local storage cleared.");
    }
}
```

---

## 8. Helper Classes

### 8.1 BundleMetadata

```csharp
private class BundleMetadata
{
    public string Version { get; set; }
    public DateTime CacheDate { get; set; }
    public uint Size { get; set; }
}
```

* **`Version`**: a string representing the bundle’s version (whatever convention you choose, e.g. `"1.0.0"`).
* **`CacheDate`**: the UTC timestamp when you cached the bundle.
* **`Size`**: file size in bytes (uint) of the stored bundle.

Used by `CacheAssetBundleAsync` to track whether a local bundle is stale.

---

## 9. Real-World Usage Examples

Below are some end-to-end scenarios demonstrating how you might employ **LocalStorageHelper** in a typical Unity project.

### 9.1 Saving Player Profile with Encryption + Cloud Sync

```csharp
[Serializable]
public class PlayerProfile
{
    public string playerName;
    public int level;
    public int coins;
}

public class ProfileController : MonoBehaviour
{
    private async void Start()
    {
        // Must have Firebase initialized somewhere before we try to sync
        await FirebaseHelper.InitializeAsync();

        // Ensure LocalStorageHelper is ready
        LocalStorageHelper.Initialize();

        // Attempt to pull any existing profile from cloud (if user has one)
        bool pulled = await LocalStorageHelper.PullFromCloudAsync("player_profile");
        if (pulled)
        {
            PlayerProfile loaded = await LocalStorageHelper.LoadDataAsync<PlayerProfile>(
                                     "player_profile", 
                                     encrypted: true, 
                                     useCache: true);
            if (loaded != null)
                Debug.Log($"Loaded profile for {loaded.playerName}, Level {loaded.level}, Coins {loaded.coins}");
        }
        else
        {
            Debug.Log("No existing cloud profile; using defaults.");
        }
    }

    public async void SaveButtonPressed(string playerName, int level, int coins)
    {
        var profile = new PlayerProfile { playerName = playerName, level = level, coins = coins };
        // Encrypt on disk, push to cloud, and update in-memory cache
        bool saved = await LocalStorageHelper.SaveDataAsync("player_profile", profile,
                                                            encrypt: true, cloudSync: true);
        if (saved)
            Debug.Log("Profile saved locally (encrypted) and synced to cloud.");
    }
}
```

**Explanation**

1. On `Start()`, we call `FirebaseHelper.InitializeAsync()` (else cloud calls will fail).
2. `LocalStorageHelper.Initialize()` ensures the `SaveData` folder exists and loads any existing JSON files into memory.
3. `PullFromCloudAsync("player_profile")` attempts to fetch Firestore document `userdata/{bundleId}/player_profile`. If found, we write that JSON to disk and update cache.
4. Then we call `LoadDataAsync<PlayerProfile>("player_profile", encrypted: true)`, which reads the encrypted JSON file, decrypts, and deserializes into our `PlayerProfile` object.
5. When the user hits “Save,” we package the new `PlayerProfile` and call `SaveDataAsync("player_profile", profile, encrypt: true, cloudSync: true)`. This writes the encrypted JSON to disk and pushes the same object to Firestore.

---

### 9.2 Saving Game Settings Without Cloud Sync

```csharp
[Serializable]
public class GameSettings
{
    public float masterVolume;
    public float musicVolume;
    public float sfxVolume;
    public bool invertYAxis;
}

public class SettingsMenu : MonoBehaviour
{
    public async void OnApplySettings(float masterVol, float musicVol, float sfxVol, bool invertY)
    {
        var settings = new GameSettings
        {
            masterVolume = masterVol,
            musicVolume = musicVol,
            sfxVolume = sfxVol,
            invertYAxis = invertY
        };

        // Only save locally; do not touch cloud
        bool saved = await LocalStorageHelper.SaveDataAsync("game_settings", settings,
                                                            encrypt: false, cloudSync: false);
        if (saved)
            Debug.Log("Settings saved locally (no cloud sync).");
    }

    private async void OnEnable()
    {
        // Load settings from local, defaults if none exist
        GameSettings defaultSettings = new GameSettings { masterVolume = 1f, musicVolume = 0.8f, sfxVolume = 0.8f, invertYAxis = false };
        var loaded = await LocalStorageHelper.LoadDataAsync<GameSettings>("game_settings",
                                                                          encrypted: false,
                                                                          useCache: true);
        if (loaded != null)
        {
            // Apply loaded settings to UI sliders, etc.
        }
        else
        {
            // Use defaults
        }
    }
}
```

**Explanation**

* We explicitly pass `cloudSync: false` to skip any Firestore calls.
* Encryption is disabled (`encrypt: false`), since these settings are not sensitive.

---

### 9.3 Caching & Checking AssetBundle Versions

```csharp
public class BundleUpdater : MonoBehaviour
{
    private async void CheckAndUpdateBundle(string bundleName, string latestVersion)
    {
        bool isOutdated = await LocalStorageHelper.IsBundleOutdatedAsync(bundleName, latestVersion);
        if (isOutdated)
        {
            Debug.Log($"{bundleName} is outdated (expected v{latestVersion}). Downloading...");
            // Force re-download and re-cache
            await AssetBundleManager.UnloadBundle(bundleName);
            var bundle = await AssetBundleManager.LoadBundleAsync(bundleName);
            if (bundle != null)
            {
                // Cache metadata under "bundle_{bundleName}_meta"
                await LocalStorageHelper.CacheAssetBundleAsync(bundleName, latestVersion);
                Debug.Log($"Cached {bundleName} v{latestVersion}");
            }
            else
            {
                Debug.LogError($"Failed to download {bundleName}.");
            }
        }
        else
        {
            Debug.Log($"{bundleName} is already up-to-date (v{latestVersion}).");
        }
    }
}
```

**Explanation**

1. Call `IsBundleOutdatedAsync(bundleName, latestVersion)`.
2. If `true`, unload any existing local bundle, re-download via `AssetBundleManager`, then call `CacheAssetBundleAsync(...)` to record metadata.
3. Next time you call `IsBundleOutdatedAsync` with the same `latestVersion`, it will return `false`.

---

### 9.4 Manual & Periodic Cloud Sync

```csharp
public class SyncScheduler : MonoBehaviour
{
    private void Start()
    {
        // Disable auto sync; we’ll sync manually or on a timer
        LocalStorageHelper.ConfigureSync(autoSync: false);

        // Kick off periodic syncing every 10 minutes
        InvokeRepeating(nameof(PeriodicSync), 0f, 600f);
    }

    private async void PeriodicSync()
    {
        // Suppose we want to sync "player_profile" every 10 minutes
        bool pulled = await LocalStorageHelper.PullFromCloudAsync("player_profile");
        if (pulled)
            Debug.Log("Periodic pull completed.");

        // Next, push any modified keys to cloud
        // But since autoSync is false, we need to manually push each ModifiedKey:
        foreach (string key in new List<string>(LocalStorageHelper.ModifiedKeys))
        {
            if (LocalStorageHelper.DataCache.TryGetValue(key, out object obj))
            {
                // Must cast obj back to T to call SyncToCloudAsync<T>, but we have it as object.
                // If we know its type, e.g. "player_profile" → PlayerProfile, we cast:
                if (key == "player_profile" && obj is PlayerProfile profile)
                {
                    await FirebaseHelper.SetDocumentAsync($"userdata/{Application.identifier}", key, profile);
                    Debug.Log($"Pushed {key} to cloud manually.");
                }
                // Repeat for each known key that requires typed data.
            }
        }
        LocalStorageHelper.ModifiedKeys.Clear();
    }
}
```

**Explanation**

* We first call `ConfigureSync(autoSync: false)` to disable any automatic cloud writes on each save.
* Then in `PeriodicSync()`, we manually call `PullFromCloudAsync("player_profile")`.
* To push whatever is in `ModifiedKeys`, we look up each key in `DataCache`, cast to its expected type, then call `FirebaseHelper.SetDocumentAsync(...)` directly.
* Finally, clear `ModifiedKeys`.

---

## 10. Summary of Public Members

* **Initialization**

  * `void Initialize()`

* **Local Storage**

  * `Task<bool> SaveDataAsync<T>(string key, T data, bool encrypt = false, bool cloudSync = true)`
  * `Task<T> LoadDataAsync<T>(string key, bool encrypted = false, bool useCache = true)`

* **Cloud Sync**

  * `Task<bool> PullFromCloudAsync(string key)`

* **Asset Bundle Caching**

  * `Task<bool> CacheAssetBundleAsync(string bundleName, string version)`
  * `Task<bool> IsBundleOutdatedAsync(string bundleName, string latestVersion)`

* **Settings Management**

  * `void ConfigureSync(bool autoSync, TimeSpan? interval = null)`
  * `void ClearCache(bool deleteFiles = false)`

---

With this documentation and the usage examples, you have a full picture of how **LocalStorageHelper** can be used to:

* Persist arbitrary data (JSON files under `SaveData/`)
* Encrypt sensitive data at rest
* Keep everything cached in memory for fast access
* Automatically (or manually) push/pull data to/from Firebase Firestore
* Track and compare AssetBundle versions for on-demand bundle updates
* Easily wipe local cache or switch sync settings at runtime

