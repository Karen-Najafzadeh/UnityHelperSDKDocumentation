Below is a comprehensive guide to the **SaveSystemHelper** utility. It explains how each part of the helper works, shows real‐world usage examples, and concludes with a concise table of methods and their purposes.

---

## Overview

`SaveSystemHelper` is a static Unity helper designed to handle robust game‐state persistence, multi‐slot save management, automatic backups, optional encryption, and basic cloud synchronization support. Key features include:

1. **Multiple Save Slots**: Save and load under named slots (e.g. `"Slot1"`, `"UserA"`).
2. **Automatic Backups**: Whenever you overwrite an existing save, a timestamped `.bak` backup is created, and only the five most recent backups are kept.
3. **Encryption**: Encrypts JSON payloads on disk using AES (with a fixed key in this example).
4. **Save File Validation & Fallback**: If loading fails (corrupt file), the helper attempts to restore from the newest valid backup.
5. **Cloud Sync Stubs**: Placeholder methods for uploading/downloading to a cloud service (e.g. Firebase, AWS).
6. **Auto‐Save System & Migration**: Although not fully implemented here, the structure allows for easy addition of auto‐save loops and schema migrations.

Under the hood, it stores:

* **`SaveDirectory`**: `<persistentDataPath>/Saves`
* **`BackupDirectory`**: `<persistentDataPath>/Saves/Backups`

When you call `SaveGameAsync(slot, data)`, it:

1. Serializes `data` via `JsonUtility.ToJson(...)` (pretty‐printed).
2. Encrypts the JSON string to bytes.
3. If a save file already exists for that slot, creates a timestamped backup `.bak` file.
4. Writes the encrypted bytes to `slot.sav` in `SaveDirectory`.
5. Cleans up old backups, keeping at most five per slot.

When you call `LoadGameAsync<T>(slot)`, it:

1. Locates `slot.sav`. If missing, returns a new `T()`.
2. Reads and decrypts data.
3. Attempts to deserialize JSON back into `T` via `JsonUtility.FromJson<T>(...)`.
4. If deserialization fails (e.g. corrupted), it tries to restore from the newest valid backup.

---

## 1. Initialization (Static Constructor)

```csharp
static SaveSystemHelper()
{
    Directory.CreateDirectory(SaveDirectory);
    Directory.CreateDirectory(BackupDirectory);
}
```

**What it does**

* As soon as the class is referenced for the first time, the CLR static constructor runs, ensuring that both the main `SaveDirectory` and its `Backups` subfolder exist.

**Behavior**

* If either directory doesn’t exist, it is created.
* If they already exist, nothing happens.

**Why it matters**

* Guarantees that subsequent file‐write operations (save, backup) won’t fail due to missing folders.

**Real‐World Example**

* As soon as any code calls `SaveSystemHelper.SaveGameAsync("Slot1", gameState)`, the static constructor has already created `…/Saves` and `…/Saves/Backups`. You never have to manually set up folders in the file system.

---

## 2. Save Operations

### 2.1 SaveGameAsync

```csharp
public static async Task SaveGameAsync<T>(string slot, T data)
```

**What it does**

1. Serializes the `data` object of type `T` into a pretty‐printed JSON string using `JsonUtility.ToJson(data, prettyPrint: true)`.
2. Encrypts that JSON string into a byte array via `EncryptData(json)`.
3. If a previous save already exists for `slot` (i.e. `<SaveDirectory>/<slot>.sav`), calls `CreateBackupAsync(slot)` to copy it into `…/Backups/<slot>_<timestamp>.bak`.
4. Writes the encrypted bytes to `<SaveDirectory>/<slot>.sav` by calling `await File.WriteAllBytesAsync(path, encrypted)`.
5. Invokes `CleanOldBackupsAsync(slot)` to delete older backups beyond `MaxBackups` (5).

**Parameters**

* `slot`: Unique string identifier for this save slot (e.g. `"Slot1"`, `"ProfileA"`).
* `data`: The object you wish to save (any serializable type).

**Behavior in Detail**

1. **`string path = GetSaveFilePath(slot);`**

   * Constructs `…/Saves/<slot>.sav`.
2. **`string json = JsonUtility.ToJson(data, true);`**

   * Converts the `data` object into a formatted JSON string.
3. **`byte[] encrypted = EncryptData(json);`**

   * Encrypts JSON via AES key (32‐byte key derived from `EncryptionKey`), returning the ciphertext.
   * If encryption fails, falls back to returning UTF8 bytes of the raw JSON.
4. **Backup existing save**

   * If `<slot>.sav` exists, calls `CreateBackupAsync(slot)`:

     * Copies the old encrypted bytes into `…/Backups/<slot>_<yyyyMMdd_HHmmss>.bak`.
5. **`await File.WriteAllBytesAsync(path, encrypted);`**

   * Writes new encrypted data to file.
6. **`await CleanOldBackupsAsync(slot);`**

   * Enumerates `…/Backups/<slot>_*.bak`, sorts descending (latest first), skips the first 5, and deletes the rest.

**Real‐World Example**

```csharp
[Serializable]
public class PlayerData
{
    public int level;
    public float health;
    public Vector3 position;
}

// ... somewhere in your Save Manager:
public async void OnPlayerRequestedSave()
{
    PlayerData pdata = new PlayerData
    {
        level = 10,
        health = 75.5f,
        position = player.transform.position
    };

    // Save under slot "PlayerSlot1":
    await SaveSystemHelper.SaveGameAsync("PlayerSlot1", pdata);
    Debug.Log("Game saved to slot PlayerSlot1!");
}
```

* If `PlayerSlot1.sav` already exists, a backup called `PlayerSlot1_<timestamp>.bak` is created.
* The new `PlayerSlot1.sav` is written to disk with encrypted JSON.

---

### 2.2 LoadGameAsync

```csharp
public static async Task<T> LoadGameAsync<T>(string slot) where T : new()
```

**What it does**

1. Constructs the path `<SaveDirectory>/<slot>.sav`.
2. If the file does not exist, returns `new T()`.
3. Otherwise, attempts to read all bytes (`await File.ReadAllBytesAsync(path)`).
4. Calls `DecryptData(encrypted)` to obtain the JSON string—if decryption fails, it logs an error and returns fallback to backup restoration.
5. Attempts `JsonUtility.FromJson<T>(json)` to reconstruct the saved object.
6. If JSON parsing fails (e.g. corrupted file), calls `RestoreFromBackupAsync<T>(slot)` to scan backups in `…/Backups`, attempting to load the newest valid backup.
7. If backup restoration succeeds, returns that object; otherwise returns `new T()`.

**Parameters**

* `slot`: The same identifier used in `SaveGameAsync`.

**Behavior in Detail**

* **`if (!File.Exists(path)) return new T();`**

  * No save file means “nothing to load,” so return default‐constructed `T`.
* **`byte[] encrypted = await File.ReadAllBytesAsync(path);`**

  * Reads encrypted content.
* **`string json = DecryptData(encrypted);`**

  * Decrypts using the same AES key. If it fails, logs and proceeds to attempt backup.
* **`T loaded = JsonUtility.FromJson<T>(json);`**

  * If successful, returns `loaded`.
* **Catch exceptions**

  * If any exception—either reading, decrypting, or parsing JSON—occurs, logs the error and calls `RestoreFromBackupAsync<T>(slot)`:

    * Looks up `…/Backups/<slot>_*.bak` files sorted latest→oldest.
    * Tries each one with `File.ReadAllBytesAsync(backup)`, `DecryptData(...)`, and `JsonUtility.FromJson<T>(json)`.
    * Returns the first successful restoration.
  * If no valid backup, returns `new T()`.

**Real‐World Example**

```csharp
public async void OnGameLoadRequest()
{
    // Attempt to load PlayerData from slot "PlayerSlot1"
    PlayerData loadedData = await SaveSystemHelper.LoadGameAsync<PlayerData>("PlayerSlot1");

    // If no save existed or loading failed completely, loadedData is default (level=0, health=0, position=(0,0,0))
    player.transform.position = loadedData.position;
    player.Health = loadedData.health;
    player.Level = loadedData.level;

    Debug.Log($"Loaded PlayerData: level={loadedData.level}, health={loadedData.health}.");
}
```

* If `PlayerSlot1.sav` exists and is intact, `loadedData` has the saved values.
* If the file is missing, `loadedData` is `new PlayerData()` (all fields default).
* If the file is corrupted, it tries to restore from a backup (e.g. `PlayerSlot1_20250310_142530.bak`).

---

### 2.3 DoesSaveExist

```csharp
public static bool DoesSaveExist(string slot)
```

**What it does**

* Checks for the existence of `<SaveDirectory>/<slot>.sav`.
* Returns `true` if the file is present; otherwise `false`.

**Usage Example**

```csharp
if (SaveSystemHelper.DoesSaveExist("PlayerSlot1"))
{
    Debug.Log("PlayerSlot1 has a saved game—presenting 'Load' button.");
}
else
{
    Debug.Log("No save detected for PlayerSlot1.");
}
```

---

### 2.4 DeleteSaveAsync

```csharp
public static async Task DeleteSaveAsync(string slot)
```

**What it does**

1. Finds `<SaveDirectory>/<slot>.sav`.
2. If it exists, calls `CreateBackupAsync(slot)` one last time (so you still have a backup even if you delete).
3. Deletes the `.sav` file.

**Real‐World Example**

```csharp
public async void OnPlayerWantsReset()
{
    // Player clicked “Delete Save”:
    await SaveSystemHelper.DeleteSaveAsync("PlayerSlot1");
    Debug.Log("Deleted PlayerSlot1 save (backup preserved).");
}
```

* Even though the main `.sav` is removed, you can later restore from a backup in `…/Backups` if necessary.

---

## 3. Backup Management

### 3.1 CreateBackupAsync (private)

```csharp
private static async Task CreateBackupAsync(string slot)
```

**What it does**

* Copies the existing `<slot>.sav` file to a new backup file named `<slot>_<yyyyMMdd_HHmmss>.bak` inside `BackupDirectory`.
* The timestamp uses the system time in `"yyyyMMdd_HHmmss"` format (e.g. `20250310_142530`).

**Behavior**

1. **`string sourcePath = GetSaveFilePath(slot);`**

   * If `<slot>.sav` doesn’t exist, just returns.
2. **`string timestamp = DateTime.Now.ToString("yyyyMMdd_HHmmss");`**
3. **`string backupPath = Path.Combine(BackupDirectory, $"{slot}_{timestamp}.bak");`**
4. **`byte[] data = await File.ReadAllBytesAsync(sourcePath);`**
5. **`await File.WriteAllBytesAsync(backupPath, data);`**

**When it runs**

* Called whenever `SaveGameAsync` is overwriting an existing save.
* Called in `DeleteSaveAsync` to preserve data before deletion.

---

### 3.2 RestoreFromBackupAsync (private)

```csharp
private static async Task<T> RestoreFromBackupAsync<T>(string slot) where T : new()
```

**What it does**

* Finds all backup files matching `<slot>_*.bak` in descending order (newest first).
* For each backup, attempts to read, decrypt, and parse JSON back into `T`.
* Returns the first successfully deserialized instance.
* If none succeed, returns `default` (which is `null` for reference types or `new T()` if `T` is a value type).

**Behavior**

1. **`var backups = Directory.GetFiles(BackupDirectory, $"{slot}_*.bak") .OrderByDescending(f => f).ToList();`**
2. Loop through `backups`:

   * `byte[] encrypted = await File.ReadAllBytesAsync(backup);`
   * `string json = DecryptData(encrypted);`
   * Attempt `T candidate = JsonUtility.FromJson<T>(json);`
   * If successful, `return candidate;`
   * If any exception, skip to next.
3. If loop finishes without success, `return default;`.

**Usage Example**

* Primarily called internally by `LoadGameAsync` when the main `.sav` is corrupt.

---

### 3.3 CleanOldBackupsAsync (private)

```csharp
private static async Task CleanOldBackupsAsync(string slot)
```

**What it does**

* After a successful `SaveGameAsync(slot, data)`, it ensures that at most `MaxBackups` (5) backups exist for that slot.
* It lists `…/Backups/<slot>_*.bak`, orders them descending (latest first), and calls `File.Delete(...)` on any beyond the first 5.

**Behavior**

1. **`var backups = Directory.GetFiles(BackupDirectory, $"{slot}_*.bak") .OrderByDescending(f => f).Skip(MaxBackups);`**
2. For each `backup` in that skip list:

   * Attempts `await Task.Run(() => File.Delete(backup));`
   * Logs a warning if deletion fails.

**Why it matters**

* Prevents unlimited accumulation of backup files, capping storage footprint at roughly `MaxBackups * sizeOfSingleSave`.

---

## 4. Cloud Sync (Stubs)

> **Note:** These methods are placeholders. You must integrate your preferred cloud service (Firebase, AWS S3, PlayFab, etc.) to actually upload/download data.

### 4.1 UploadToCloudAsync

```csharp
public static async Task<bool> UploadToCloudAsync(string slot)
```

**What it does**

* Reads the bytes of `<slot>.sav`.
* (Placeholder) Uploads them to your cloud service under a path like `saves/<slot>`.
* Returns `true` if upload “succeeds,” `false` otherwise.

**Behavior in This Example**

1. If `<slot>.sav` doesn’t exist, returns `false`.
2. Reads file into `byte[] data`.
3. /\* TODO: use FirebaseHelper or similar to upload `data` \*/
4. Returns `true` (since no actual cloud code is present).

**Real‐World Example (Pseudo)**

```csharp
public static async Task<bool> UploadToCloudAsync(string slot)
{
    string path = GetSaveFilePath(slot);
    if (!File.Exists(path)) return false;

    byte[] data = await File.ReadAllBytesAsync(path);
    // Example: await FirebaseHelper.UploadFileAsync(data, $"saves/{slot}.sav");
    return true;
}
```

### 4.2 DownloadFromCloudAsync

```csharp
public static async Task<bool> DownloadFromCloudAsync(string slot)
```

**What it does**

* (Placeholder) Downloads the backup `/cloud/saves/<slot>` as `byte[] data`.
* Writes `data` to `<slot>.sav` in `SaveDirectory`.
* Returns `true` if download “succeeds,” `false` otherwise.

**Behavior in This Example**

* Doesn’t actually download; just yields once (`await Task.Yield()`) and returns `true`.

**Real‐World Example (Pseudo)**

```csharp
public static async Task<bool> DownloadFromCloudAsync(string slot)
{
    // Example: byte[] data = await FirebaseHelper.DownloadFileAsync($"saves/{slot}.sav");
    // await File.WriteAllBytesAsync(GetSaveFilePath(slot), data);
    await Task.Yield();
    return true;
}
```

---

## 5. Helper Methods

### 5.1 GetSaveFilePath

```csharp
private static string GetSaveFilePath(string slot)
```

**What it does**

* Constructs and returns the full path for a given slot’s main save file:

  ```
  Path.Combine(SaveDirectory, $"{slot}.sav")
  ```

**Example**

* If `Application.persistentDataPath` is
  `C:/Users/Player/AppData/LocalLow/MyGame`, then
  `SaveDirectory = C:/…/MyGame/Saves`, so
  `GetSaveFilePath("Player1")` →
  `C:/…/MyGame/Saves/Player1.sav`.

---

### 5.2 EncryptData

```csharp
private static byte[] EncryptData(string data)
```

**What it does**

1. Converts the input `data` string to a UTF8 byte array.
2. Creates a new `Aes` instance with:

   * Key = first 32 bytes of `EncryptionKey.PadRight(32)`.
   * IV = 16 zero bytes (NOT secure—placeholder).
3. Uses `CreateEncryptor()` to encrypt the raw UTF8 bytes into ciphertext.
4. Returns the encrypted byte array.
5. If any exception occurs, logs an error and returns UTF8 bytes of the plaintext fallback.

**Why it matters**

* On disk, your save file is “obfuscated” so that users can’t casually open/modify it.
* In production, you should use a randomly generated IV per encryption and store it alongside the ciphertext, rather than the zero‐IV used here.

---

### 5.3 DecryptData

```csharp
private static string DecryptData(byte[] data)
```

**What it does**

1. Creates a new `Aes` with the same fixed Key and zero IV.
2. Uses `CreateDecryptor()` to decrypt the ciphertext `data` into plaintext bytes.
3. Converts decrypted bytes to a UTF8 string.
4. If decryption fails, logs an error and returns `Encoding.UTF8.GetString(data)`—i.e. treats the data as unencrypted JSON.

**Real‐World Note**

* Because IV is always zero, any two saved games identical in content produce identical ciphertext. A production‐grade system should randomly generate a unique IV for each encryption and store it (e.g. prefix the ciphertext with the IV bytes, then call `Aes.IV = iv` before decrypting).

---

## 6. Putting It All Together: Full Workflow Example

Below is a typical save/load workflow in a game’s “GameManager” MonoBehaviour:

```csharp
public class GameManager : MonoBehaviour
{
    [Serializable]
    public class GameState
    {
        public int playerLevel;
        public float playerHealth;
        public Vector3 playerPosition;
        public List<string> inventoryItems;
    }

    private void Start()
    {
        // 1. On startup, attempt to load slot "MainSave"
        _ = LoadGameState();
    }

    private async Task LoadGameState()
    {
        GameState state = await SaveSystemHelper.LoadGameAsync<GameState>("MainSave");

        // If no save or restored from backup, state fields are either default or loaded values.
        playerTransform.position = state.playerPosition;
        playerHealth = state.playerHealth;
        playerLevel = state.playerLevel;
        inventory = new List<string>(state.inventoryItems ?? new List<string>());

        Debug.Log($"Loaded GameState: level={state.playerLevel}, health={state.playerHealth}");
    }

    public async void OnAutoSaveTrigger() // e.g., called every few minutes
    {
        GameState state = new GameState
        {
            playerLevel = playerLevel,
            playerHealth = playerHealth,
            playerPosition = playerTransform.position,
            inventoryItems = inventory.ToList()
        };

        await SaveSystemHelper.SaveGameAsync("MainSave", state);
        Debug.Log("Game auto‐saved to slot MainSave");
    }

    public async void OnPlayerRequestsSave()
    {
        // Save manually as well
        await OnAutoSaveTrigger();
    }

    public async void OnPlayerRequestsLoad()
    {
        await LoadGameState();
    }

    public async void OnDeleteProfile()
    {
        // Backup is still preserved
        await SaveSystemHelper.DeleteSaveAsync("MainSave");
        Debug.Log("Deleted MainSave slot (but a backup remains).");
    }

    public async void OnCloudSyncNow()
    {
        // Upload local save to cloud
        bool success = await SaveSystemHelper.UploadToCloudAsync("MainSave");
        Debug.Log($"Cloud upload success: {success}");

        // (Later) Download from cloud (overwrites local)
        success = await SaveSystemHelper.DownloadFromCloudAsync("MainSave");
        if (success) Debug.Log("Cloud download succeeded; you may want to re‐load now.");
    }
}
```

**Explanation of Workflow**

1. **`Start()`**

   * Immediately calls `LoadGameState()`, which uses `LoadGameAsync<GameState>("MainSave")`.
   * If no save exists, returns default `GameState` (`playerLevel=0`, etc.).
2. **`OnAutoSaveTrigger()`**

   * Packs current game variables into a `GameState` instance and calls `SaveGameAsync("MainSave", state)`.
   * On first save, no existing file → no backup. On subsequent saves, creates a timestamped backup before overwriting.
   * Automatically cleans any backups older than the five most recent.
3. **`OnPlayerRequestsLoad()`**

   * Simply calls `LoadGameState()` again to restore game state from file.
4. **`OnDeleteProfile()`**

   * Deletes the current `MainSave.sav` (after creating one last backup). The local file is gone but backups remain.
5. **`OnCloudSyncNow()`**

   * Placeholder call to upload/download. In production, implement the TODO blocks with your service’s SDK (Firebase Storage, Azure Blob, etc.).

---

## 7. Method Summary Table

| Method                                               | Purpose                                                                                                                                |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Static Constructor**                               | Ensures `SaveDirectory` and `BackupDirectory` both exist before any other operations.                                                  |
| `SaveGameAsync<T>(string slot, T data)`              | Serialize `data` to JSON, encrypt it, back up existing slot, write encrypted bytes to `<slot>.sav`, then clean old backups.            |
| `LoadGameAsync<T>(string slot) where T : new()`      | Read `<slot>.sav`, decrypt, parse JSON into `T`; if it fails, attempt to restore from the newest valid backup; else return `new T()`.  |
| `DoesSaveExist(string slot)`                         | Returns `true` if `<slot>.sav` exists on disk; otherwise `false`.                                                                      |
| `DeleteSaveAsync(string slot)`                       | Back up current `<slot>.sav` (if present), then delete the main save file.                                                             |
| **(Private) CreateBackupAsync(string slot)**         | Copies existing `<slot>.sav` into `BackupDirectory` as `<slot>_<timestamp>.bak`.                                                       |
| **(Private) RestoreFromBackupAsync<T>(string slot)** | Scans `BackupDirectory/<slot>_*.bak` newest→oldest; attempts to decrypt/deserialize each into `T`; returns first success or `default`. |
| **(Private) CleanOldBackupsAsync(string slot)**      | Keeps only the five most recent `<slot>_*.bak` backups; deletes the rest.                                                              |
| **(Cloud Stub) UploadToCloudAsync(string slot)**     | Placeholder: read `<slot>.sav` and “upload” it to cloud storage. Returns `true` by default.                                            |
| **(Cloud Stub) DownloadFromCloudAsync(string slot)** | Placeholder: “download” cloud copy into `<slot>.sav`. Returns `true` by default.                                                       |
| **Helper) GetSaveFilePath(string slot)**             | Returns the full local path: `<persistentDataPath>/Saves/<slot>.sav`.                                                                  |
| **Helper) EncryptData(string data)**                 | Encrypts the given JSON string via AES (32‐byte key, zero IV), returning ciphertext bytes. Fallbacks to raw UTF8 on error.             |
| **Helper) DecryptData(byte\[] data)**                | Decrypts AES ciphertext bytes to UTF8 string. If decryption fails, treats `data` as raw JSON and returns `UTF8.GetString(data)`.       |

---

## 8. Best Practices & Tips

1. **Use Consistent Slot Names**

   * Pick stable slot identifiers (e.g. `"Player1"`, `"ProfileA"`) so backups and loads always match.
2. **Keep EncryptionKey Secret & IV Random**

   * Instead of a fixed `"YOUR_ENCRYPTION_KEY_HERE"`, generate a secure random key at build time.
   * Replace the zero‐IV with a randomly generated IV stored alongside the ciphertext (e.g. prefix the ciphertext).
3. **Backups Prevent Data Loss**

   * Whenever a save might become corrupted (e.g. power failure during write), you can fall back to one of the `.bak` files.
4. **Backup Retention**

   * Adjust `MaxBackups` if you want more or fewer history copies.
5. **Cloud Sync Integration**

   * Replace the `TODO` blocks with actual calls (e.g., `FirebaseHelper.UploadFileAsync(...)` or `AWS S3` SDK).
   * Consider versioning or using metadata to detect if the cloud copy is newer than local.
6. **Auto‐Save**

   * In a real project, you might start a coroutine that calls `SaveGameAsync` every few minutes or at key checkpoints.
7. **Migration Hooks**

   * You can extend this class to detect a “version” field inside your saved JSON, then apply transformations when needed.

---

With this documentation and the method table above, you have everything needed to:

* **Initialize** and manage multiple save slots with a single line (`SaveGameAsync("SlotName", data)`).
* **Load** safely, handling corruption by falling back to backups.
* **Delete** while preserving backups.
* **Back up** older saves automatically, with a cap on how many are retained.
* **Encrypt** saved data to add a layer of protection (although replace the zero‐IV for production).
* **Extend** to integrate with your chosen cloud service.

Feel free to adapt or extend `SaveSystemHelper` to fit your project’s specific requirements, such as adding a “last‐modified timestamp” in the JSON, integrating PlayerPrefs for quick small values, or using compression (e.g., GZip) before encryption.
