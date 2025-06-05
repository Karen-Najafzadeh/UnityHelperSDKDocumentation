Below is a comprehensive walkthrough of **ObjectPoolHelper**—a static utility class that implements a full‐featured object pooling system for Unity. It handles:

1. **Runtime Pool Creation**

   * Create and configure pools for any prefab (including those loaded from AssetBundles).
   * Async initialization so you can warm up pools over multiple frames.

2. **Prefab Instantiation & Auto‐Expansion**

   * Pre‐allocate a specified `InitialSize`.
   * Automatically grow by `ExpandBy` up to `MaxSize` if more objects are requested.
   * Each pooled instance is a clone of the original prefab, which can be a simple `GameObject` or a complex hierarchy.

3. **Pool Statistics & Monitoring**

   * Query how many objects exist in a pool, how many are active, how many are available, and the highest concurrent usage (peak).

4. **Component Pooling Support**

   * Each pooled object carries a `PoolableObject` component that knows its `PoolKey`. Calling `PoolableObject.ReturnToPool()` returns it to the correct pool automatically.

5. **Integration with AssetBundleManager**

   * Special method to initialize a pool by loading a prefab from an AssetBundle, then pooling that.

Below, each region of the code is explained in detail, followed by copy‐and‐paste examples showing how to integrate this into a real project.

---

## Table of Contents

1. [Overall Structure](#1-overall-structure)
2. [Data Structures & Fields](#2-data-structures--fields)
3. [Public API: Pool Management](#3-public-api-pool-management)
   3.1. [`InitializePoolAsync(string poolKey, GameObject prefab, PoolSettings settings = null)`](#31-initializepoolasyncstring-poolkey-gameobject-prefab-poolsettings-settings--null)
   3.2. [`InitializePoolFromBundleAsync(string poolKey, string bundleName, string prefabName, PoolSettings settings = null)`](#32-initializepoolfrombundleasyncstring-poolkey-string-bundlename-string-prefabname-poolsettings-settings--null)
   3.3. [`Get(string poolKey, Vector3 position = default, Quaternion rotation = default)`](#33-getstring-poolkey-vector3-position--default-quaternion-rotation--default)
   3.4. [`Return(GameObject obj)`](#34-returngameobject-obj)
   3.5. [`PreWarmAsync(string poolKey, int count)`](#35-prewarmasyncstring-poolkey-int-count)
4. [Public API: Pool Information](#4-public-api-pool-information)
   4.1. [`GetPoolStats(string poolKey)`](#41-getpoolstatsstring-poolkey)
   4.2. [`ClearPool(string poolKey)`](#42-clearpoolstring-poolkey)
   4.3. [`ClearAllPools()`](#43-clearallpools)
5. [Internal Class: `ObjectPool`](#5-internal-class-objectpool)
   5.1. **Fields & Properties**
   5.2. **Constructor**
   5.3. **`InitializeAsync()` and `ExpandAsync(int count)`**
   5.4. **`Get(Vector3 position, Quaternion rotation)`**
   5.5. **`Return(GameObject obj)`**
   5.6. **`Clear()`**
   5.7. **`CreateNewObject()`**
6. [Helper Classes & Structs](#6-helper-classes--structs)
   6.1. **`PoolSettings`**
   6.2. **`PoolStats`**
   6.3. **`PoolableObject`**
7. [Real‐World Usage Examples](#7-realworld-usage-examples)
   7.1. Bootstrap & Initialize a Simple Pool
   7.2. Fetching & Returning Objects at Runtime
   7.3. Pre‐Warming a Pool in Advance
   7.4. Loading a Prefab from an AssetBundle into a Pool
   7.5. Querying Pool Statistics for Debugging
   7.6. Clearing Pools on Level Unload
8. [Customization & Tips](#8-customization--tips)
9. [Summary of Public API](#9-summary-of-public-api)

---

## 1. Overall Structure

```csharp
public static class ObjectPoolHelper
{
    private static readonly Dictionary<string, ObjectPool> _pools = new Dictionary<string, ObjectPool>();
    private static readonly PoolSettings _defaultSettings = new PoolSettings { ... };

    #region Pool Management
    public static async Task<bool> InitializePoolAsync(string poolKey, GameObject prefab, PoolSettings settings = null) { … }
    public static async Task<bool> InitializePoolFromBundleAsync(... ) { … }
    public static GameObject Get(string poolKey, Vector3 position = default, Quaternion rotation = default) { … }
    public static void Return(GameObject obj) { … }
    public static async Task PreWarmAsync(string poolKey, int count) { … }
    #endregion

    #region Pool Information
    public static PoolStats GetPoolStats(string poolKey) { … }
    public static void ClearPool(string poolKey) { … }
    public static void ClearAllPools() { … }
    #endregion

    #region Helper Classes
    private class ObjectPool { … }
    public class PoolSettings { … }
    public struct PoolStats { … }
    #endregion
}

public class PoolableObject : MonoBehaviour
{
    public string PoolKey { get; set; }
    public void ReturnToPool() => ObjectPoolHelper.Return(gameObject);
}
```

* **Singleton‐style Static Class**: All pool operations live in `ObjectPoolHelper`. You never instantiate this class directly.
* **`_pools`**: A dictionary mapping pool keys (`string`) → `ObjectPool` instances. Each `ObjectPool` manages a single prefab.
* **`PoolSettings`**: Default and custom settings that define a pool’s size limits, auto‐expansion behavior, etc.
* **`PoolStats`**: Simple struct to return current counts (total, active, available, peak).
* **`PoolableObject`**: A MonoBehaviour you attach to any pooled object (either manually or via `CreateNewObject()`) so the object “knows” its pool key and can return itself.

---

## 2. Data Structures & Fields

```csharp
private static readonly Dictionary<string, ObjectPool> _pools = new Dictionary<string, ObjectPool>();

private static readonly PoolSettings _defaultSettings = new PoolSettings
{
    InitialSize = 10,
    MaxSize = 100,
    ExpandBy = 5,
    AutoExpand = true
};
```

1. **`_pools`**

   * Holds all active pools in the game, keyed by a unique `string poolKey`.
   * When you call `InitializePoolAsync("Enemies", enemyPrefab)`, an `ObjectPool` is created and stored as `_pools["Enemies"]`.

2. **`_defaultSettings`**

   * If you don’t provide custom `PoolSettings` when initializing a pool, it uses these defaults:

     * **`InitialSize = 10`**: Pool starts with 10 inactive instances.
     * **`MaxSize = 100`**: Pool will never exceed 100 total instances (active + available).
     * **`ExpandBy = 5`**: If the pool runs dry (no available objects), it creates 5 new instances at once.
     * **`AutoExpand = true`**: When out of available instances and `TotalCount < MaxSize`, automatically expand by `ExpandBy`. If `AutoExpand` were `false`, `Get()` would simply return `null` when no objects are available.

---

## 3. Public API: Pool Management

### 3.1 `InitializePoolAsync(string poolKey, GameObject prefab, PoolSettings settings = null)`

```csharp
public static async Task<bool> InitializePoolAsync(
    string poolKey,
    GameObject prefab,
    PoolSettings settings = null)
{
    if (_pools.ContainsKey(poolKey))
    {
        Debug.LogWarning($"Pool '{poolKey}' already exists!");
        return false;
    }

    try
    {
        var pool = new ObjectPool(prefab, settings ?? _defaultSettings);
        await pool.InitializeAsync();
        _pools[poolKey] = pool;
        return true;
    }
    catch (Exception ex)
    {
        Debug.LogError($"Failed to initialize pool '{poolKey}': {ex.Message}");
        return false;
    }
}
```

**Purpose**:
Create a brand‐new pool for the given `prefab`, identified by `poolKey`, with optional custom settings.

**Parameters**:

* `poolKey` (`string`): Unique key to identify this pool (e.g., `"BulletPool"`, `"EnemyPool"`).
* `prefab` (`GameObject`): A reference to the prefab you wish to pool (must be a `GameObject` either in the scene or loaded via `Resources/AssetBundle`).
* `settings` (`PoolSettings`, optional): Custom pool parameters (initial size, max size, expansion behavior). If `null`, `_defaultSettings` is used.

**Behavior**:

1. **Check for Existing Pool**

   * If `_pools.ContainsKey(poolKey)`, logs a warning and returns `false`. You cannot create two pools with the same key.

2. **Instantiate a New `ObjectPool`**

   * `new ObjectPool(prefab, settings ?? _defaultSettings)`: Creates an internal pool object but does not yet allocate any instances.

3. **Call `InitializeAsync()` on the `ObjectPool`**

   * `await pool.InitializeAsync()`: This asynchronously expands the pool by `InitialSize` (default 10) over multiple frames (yields each instantiation via `Task.Yield()`).

4. **Store in Dictionary**

   * On successful initialization, store the `ObjectPool` in `_pools[poolKey] = pool`.

5. **Return**

   * Returns `true` on success, `false` on failure (prefab was null, max size reached, or any exception).

**Example**:

```csharp
public class ProjectileManager : MonoBehaviour
{
    [SerializeField] private GameObject _bulletPrefab;

    private async void Start()
    {
        bool success = await ObjectPoolHelper.InitializePoolAsync(
            poolKey: "BulletPool",
            prefab: _bulletPrefab,
            settings: new ObjectPoolHelper.PoolSettings
            {
                InitialSize = 20,
                MaxSize = 200,
                ExpandBy = 10,
                AutoExpand = true
            }
        );

        if (!success)
            Debug.LogError("Failed to initialize BulletPool!");
    }
}
```

* This example allocates 20 bullets on startup (over multiple frames), and configures the pool to hold up to 200 bullets, expanding by 10 whenever necessary.

---

### 3.2 `InitializePoolFromBundleAsync(string poolKey, string bundleName, string prefabName, PoolSettings settings = null)`

```csharp
public static async Task<bool> InitializePoolFromBundleAsync(
    string poolKey, 
    string bundleName, 
    string prefabName,
    PoolSettings settings = null)
{
    try
    {
        // 1. Load the AssetBundle
        var bundle = await AssetBundleManager.LoadBundleAsync(bundleName);
        if (bundle == null) return false;

        // 2. Load the prefab (GameObject) from the bundle
        var prefab = await AssetBundleManager.LoadAssetAsync<GameObject>(
            bundleName, 
            prefabName
        );
        if (prefab == null) return false;

        // 3. Initialize pool with that prefab
        return await InitializePoolAsync(poolKey, prefab, settings);
    }
    catch (Exception ex)
    {
        Debug.LogError($"Failed to initialize pool from bundle: {ex.Message}");
        return false;
    }
}
```

**Purpose**:
Create a new pool whose prefab is loaded dynamically from an AssetBundle at runtime, instead of referencing a prefab in the scene or `Resources` folder.

**Parameters**:

* `poolKey` (`string`): Unique key for the new pool.
* `bundleName` (`string`): The name of the AssetBundle (e.g., `"environmentassets"`).
* `prefabName` (`string`): The asset name inside the bundle (e.g., `"FlyingDronePrefab"`).
* `settings` (`PoolSettings`, optional): Custom pool settings.

**Behavior**:

1. **Load the AssetBundle**

   * `await AssetBundleManager.LoadBundleAsync(bundleName)`: Asynchronously fetches or returns an existing loaded AssetBundle.
   * If the bundle is missing, returns `false`.

2. **Load the Prefab from the Bundle**

   * `await AssetBundleManager.LoadAssetAsync<GameObject>(bundleName, prefabName)`: Retrieves the `GameObject` inside the bundle.
   * If the asset is missing or fails to load, returns `false`.

3. **Call `InitializePoolAsync`** with the loaded prefab.

   * Reuses the logic from **3.1** above to build the pool.

4. **Return**

   * `true` on success, `false` on any failure.

**Example**:

```csharp
public class EnemyLoader : MonoBehaviour
{
    private async void Awake()
    {
        bool success = await ObjectPoolHelper.InitializePoolFromBundleAsync(
            poolKey: "EnemyPool",
            bundleName: "enemiesbundle",
            prefabName: "GoblinPrefab",
            settings: new ObjectPoolHelper.PoolSettings
            {
                InitialSize = 15,
                MaxSize = 50,
                ExpandBy = 5,
                AutoExpand = true
            }
        );

        if (!success)
            Debug.LogError("Could not initialize EnemyPool from AssetBundle!");
    }
}
```

* This asynchronously loads the “enemiesbundle,” retrieves the “GoblinPrefab,” and creates a pool of 15 goblins on startup, ready to spawn.

---

### 3.3 `Get(string poolKey, Vector3 position = default, Quaternion rotation = default)`

```csharp
public static GameObject Get(string poolKey, Vector3 position = default, Quaternion rotation = default)
{
    if (!_pools.TryGetValue(poolKey, out var pool))
    {
        Debug.LogError($"Pool '{poolKey}' not found!");
        return null;
    }
    return pool.Get(position, rotation);
}
```

**Purpose**:
Fetch (or “rent”) a single object from the specified pool, positioning and rotating it in the scene. The returned object is activated (`obj.SetActive(true)` internally) and tracked as “active.”

**Parameters**:

* `poolKey` (`string`): Which pool to retrieve from (e.g., `"BulletPool"`).
* `position` (`Vector3`, optional): The world‐space position where the object should be placed. Defaults to `(0,0,0)`.
* `rotation` (`Quaternion`, optional): The world‐space rotation. Defaults to `Quaternion.identity`.

**Behavior**:

1. **Lookup Pool**

   * If `_pools` does not contain `poolKey`, logs an error and returns `null`.
   * Otherwise, get the `ObjectPool` instance.

2. **Call `pool.Get(position, rotation)`**

   * Internally, if there are available objects, dequeue one.
   * If none are available and `AutoExpand == true` and `TotalCount < MaxSize`, create a new one.
   * If the pool is at max capacity and no objects are free, logs a warning and returns `null`.

3. **Returns**

   * The fetched `GameObject` (active) or `null` if unavailable.

**Example**:

```csharp
public class Weapon : MonoBehaviour
{
    [SerializeField] private Transform _muzzle;

    public void Fire()
    {
        // Request a bullet from “BulletPool”
        GameObject bullet = ObjectPoolHelper.Get("BulletPool", _muzzle.position, _muzzle.rotation);
        if (bullet != null)
        {
            // The bullet prefab likely has a script that drives it forward on Awake/OnEnable
        }
        else
        {
            Debug.LogWarning("No bullets available!");
        }
    }
}
```

* When you call `Fire()`, it instantly reuses or instantiates a bullet at the muzzle’s transform.

---

### 3.4 `Return(GameObject obj)`

```csharp
public static void Return(GameObject obj)
{
    var poolable = obj.GetComponent<PoolableObject>();
    if (poolable == null || string.IsNullOrEmpty(poolable.PoolKey))
    {
        Debug.LogWarning($"Object '{obj.name}' is not poolable!");
        return;
    }

    if (_pools.TryGetValue(poolable.PoolKey, out var pool))
    {
        pool.Return(obj);
    }
    else
    {
        Debug.LogWarning($"Pool '{poolable.PoolKey}' not found! Destroying object instead.");
        GameObject.Destroy(obj);
    }
}
```

**Purpose**:
Return (or “release”) a previously fetched object back into its pool, disabling it and making it available for reuse. Each pooled object must carry a `PoolableObject` component that has its `PoolKey` set.

**Behavior**:

1. **Check for `PoolableObject`**

   * `obj.GetComponent<PoolableObject>()` must exist. If not, log a warning (`"Object 'X' is not poolable!"`) and abort.

2. **Extract `poolable.PoolKey`**

   * If empty or null (`string.IsNullOrEmpty`), same warning and return.

3. **Lookup Pool**

   * If `_pools` contains `poolKey`, call `pool.Return(obj)`.

     * Internally, `Return()` sets `obj.SetActive(false)`, and enqueues it into `_available`.
   * If the pool doesn’t exist (e.g., was cleared), log a warning and call `GameObject.Destroy(obj)` to avoid orphaned objects.

**Example**:

```csharp
public class BulletController : MonoBehaviour
{
    private void OnBecameInvisible()
    {
        // Automatically return this bullet to the pool when it leaves camera frustum
        var poolable = GetComponent<PoolableObject>();
        if (poolable != null)
        {
            poolable.ReturnToPool();
        }
        else
        {
            Destroy(gameObject);
        }
    }
}
```

* When the bullet goes off‐screen, `OnBecameInvisible()` is called. We call `ReturnToPool()` (which internally calls `ObjectPoolHelper.Return(gameObject)`).

---

### 3.5 `PreWarmAsync(string poolKey, int count)`

```csharp
public static async Task PreWarmAsync(string poolKey, int count)
{
    if (!_pools.TryGetValue(poolKey, out var pool))
    {
        Debug.LogError($"Pool '{poolKey}' not found!");
        return;
    }
    await pool.ExpandAsync(count);
}
```

**Purpose**:
Dynamically create (and discard to the available queue) a specified number of extra instances on top of the pool’s current size, so you can avoid hiccups at critical moments.

**Parameters**:

* `poolKey` (`string`): Which pool to pre‐warm (e.g., `"EnemyPool"`).
* `count` (`int`): How many new instances to allocate immediately (up to `MaxSize - TotalCount`).

**Behavior**:

1. **Lookup Pool**

   * If not found, log an error and return.

2. **Call `pool.ExpandAsync(count)`**

   * Internally, this loops `count` times (or until `TotalCount >= MaxSize`), calling `await Task.Yield()` each iteration, instantiating a new object, disabling it, and enqueuing to `_available`.

**Example**:

```csharp
public class PreloadManager : MonoBehaviour
{
    private async void Start()
    {
        // Suppose we already called InitializePoolAsync("EnemyPool", enemyPrefab)
        // Now we want to pre-warm 10 more enemies to avoid stutter during a boss fight
        await ObjectPoolHelper.PreWarmAsync("EnemyPool", 10);
        Debug.Log("Pre-warmed 10 additional enemies!");
    }
}
```

* Calling `PreWarmAsync("EnemyPool", 10)` ensures the pool has 10 extra disabled instances queued up and ready.

---

## 4. Public API: Pool Information

### 4.1 `GetPoolStats(string poolKey)`

```csharp
public static PoolStats GetPoolStats(string poolKey)
{
    if (!_pools.TryGetValue(poolKey, out var pool))
        return default;

    return new PoolStats
    {
        TotalObjects = pool.TotalCount,
        ActiveObjects = pool.ActiveCount,
        AvailableObjects = pool.AvailableCount,
        PeakObjects = pool.PeakCount
    };
}
```

**Purpose**:
Retrieve runtime statistics on how a pool is performing—helpful for debugging memory usage, peak load, etc.

**Parameters**:

* `poolKey` (`string`): The key identifying which pool you want statistics for.

**Return Value**:
A `PoolStats` struct:

* `TotalObjects`: How many instances exist (available + active).
* `ActiveObjects`: How many are currently checked out (in use).
* `AvailableObjects`: How many are idle (in the queue).
* `PeakObjects`: The highest simultaneous `ActiveCount` ever recorded in this pool.

**Example**:

```csharp
public class PoolDebugger : MonoBehaviour
{
    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.P))
        {
            ObjectPoolHelper.PoolStats stats = ObjectPoolHelper.GetPoolStats("BulletPool");
            Debug.Log($"BulletPool → Total: {stats.TotalObjects}, Active: {stats.ActiveObjects}, Available: {stats.AvailableObjects}, Peak: {stats.PeakObjects}");
        }
    }
}
```

* Pressing “P” in play mode logs the current and peak usage of the “BulletPool.”

---

### 4.2 `ClearPool(string poolKey)`

```csharp
public static void ClearPool(string poolKey)
{
    if (_pools.TryGetValue(poolKey, out var pool))
    {
        pool.Clear();
        _pools.Remove(poolKey);
    }
}
```

**Purpose**:
Completely destroy all instances in a specific pool and remove that pool from memory.

**Parameters**:

* `poolKey` (`string`): The key identifying the pool to clear (e.g., `"EnemyPool"`).

**Behavior**:

1. **Lookup Pool**

   * If found, call `pool.Clear()`, which:

     * Iterates over both `_available` and `_active`, calls `GameObject.Destroy(obj)` on each (if not null).
     * Clears the internal data structures.

2. **Remove from `_pools`**

   * Deletes the key from the dictionary.

**Example**:

```csharp
public class LevelUnloader : MonoBehaviour
{
    private void OnDisable()
    {
        // When this level is unloaded, destroy all pooled enemies
        ObjectPoolHelper.ClearPool("EnemyPool");
    }
}
```

* This ensures that when you load a brand new level, you can wipe out the entire “EnemyPool” to free memory.

---

### 4.3 `ClearAllPools()`

```csharp
public static void ClearAllPools()
{
    foreach (var pool in _pools.Values)
    {
        pool.Clear();
    }
    _pools.Clear();
}
```

**Purpose**:
Destroy **every** object in every pool and remove all pools. After this call, `_pools` is empty.

**Behavior**:

1. **Iterate Over All Pools**

   * Call `pool.Clear()` to destroy every GameObject in each pool.

2. **Empty Dictionary**

   * `_pools.Clear()` removes all keys.

**Example**:

```csharp
public class GameManager : MonoBehaviour
{
    public void OnGameExit()
    {
        // Clean up all pools when quitting the game
        ObjectPoolHelper.ClearAllPools();
    }
}
```

* Ensures no pooled objects remain in memory when quitting or reloading the entire game.

---

## 5. Internal Class: `ObjectPool`

```csharp
private class ObjectPool
{
    private readonly GameObject _prefab;
    private readonly PoolSettings _settings;
    private readonly Queue<GameObject> _available;
    private readonly HashSet<GameObject> _active;
    private int _peakCount;

    public int TotalCount => _available.Count + _active.Count;
    public int ActiveCount => _active.Count;
    public int AvailableCount => _available.Count;
    public int PeakCount => _peakCount;

    public ObjectPool(GameObject prefab, PoolSettings settings) { … }
    public async Task InitializeAsync() { … }
    public async Task ExpandAsync(int count) { … }
    public GameObject Get(Vector3 position, Quaternion rotation) { … }
    public void Return(GameObject obj) { … }
    public void Clear() { … }
    private GameObject CreateNewObject() { … }
}
```

### 5.1 Fields & Properties

* **`_prefab`** (`GameObject`):

  * The original template used to `Instantiate(...)` new pool objects. Never modified directly at runtime.

* **`_settings`** (`PoolSettings`):

  * Holds the custom or default pool configuration (initial size, max size, expand by, auto expand).

* **`_available`** (`Queue<GameObject>`):

  * Queue of inactive (pooled) objects ready to be “rented.” Dequeue one on `Get()`, enqueue on `Return()`.

* **`_active`** (`HashSet<GameObject>`):

  * Set of currently “checked out” objects. On `Get()`, we add to `_active`. On `Return()`, we remove from `_active`.

* **`_peakCount`** (`int`):

  * Tracks the highest ever simultaneous count of active objects to help with profiling.
  * Updated inside `Get()` via:

    ```csharp
    _peakCount = Mathf.Max(_peakCount, _active.Count);
    ```

* **Properties** (read‐only):

  * `TotalCount`: Sum of `_available.Count + _active.Count`.
  * `ActiveCount`: How many are currently in use.
  * `AvailableCount`: How many are idle.
  * `PeakCount`: The maximum `ActiveCount` observed historically.

---

### 5.2 Constructor: `public ObjectPool(GameObject prefab, PoolSettings settings)`

```csharp
public ObjectPool(GameObject prefab, PoolSettings settings)
{
    _prefab = prefab;
    _settings = settings;
    _available = new Queue<GameObject>();
    _active = new HashSet<GameObject>();
}
```

* Stores the `prefab` (used to instantiate new instances).
* Copies `settings` (the custom or default `PoolSettings`).
* Initializes empty data structures (`_available`, `_active`).

### 5.3 `InitializeAsync()`

```csharp
public async Task InitializeAsync()
{
    await ExpandAsync(_settings.InitialSize);
}
```

* Simply calls `ExpandAsync(InitialSize)`, which creates the first batch of instances over multiple frames.

### 5.4 `ExpandAsync(int count)`

```csharp
public async Task ExpandAsync(int count)
{
    for (int i = 0; i < count; i++)
    {
        if (TotalCount >= _settings.MaxSize)
            break;

        await Task.Yield(); // yields control to next frame

        var obj = CreateNewObject();
        _available.Enqueue(obj);
    }
}
```

**Purpose**:
Create up to `count` new instances, one per frame (because of `await Task.Yield()`), stopping early if `TotalCount >= MaxSize`. Each new object is disabled and enqueued in `_available`.

**Step-by-Step**:

1. **Loop from `i = 0` to `count - 1`**.
2. **Check Maximum**

   * If `TotalCount >= MaxSize`, break out (never exceed maximum).
3. **`await Task.Yield()`**

   * Yield control so we spread instantiation over multiple frames, avoiding hitches if `count` is large.
4. **`CreateNewObject()`**

   * Instantiates a fresh clone of `_prefab`, disables it, assigns a `PoolableObject` if missing, and returns it.
5. **Enqueue** to `_available`.

**Example**:

* If `InitialSize = 20` and `MaxSize = 100`, calling `InitializeAsync()` will create 20 objects, one per frame.
* If you later call `ExpandAsync(15)`, it will create 15 new ones (total 35) unless that would exceed `MaxSize`.

---

### 5.5 `Get(Vector3 position, Quaternion rotation)`

```csharp
public GameObject Get(Vector3 position, Quaternion rotation)
{
    GameObject obj;

    if (_available.Count == 0 && _settings.AutoExpand && TotalCount < _settings.MaxSize)
    {
        obj = CreateNewObject();
    }
    else if (_available.Count > 0)
    {
        obj = _available.Dequeue();
    }
    else
    {
        Debug.LogWarning($"Pool for {_prefab.name} is empty and cannot expand!");
        return null;
    }

    obj.transform.position = position;
    obj.transform.rotation = rotation;
    obj.SetActive(true);
    _active.Add(obj);

    _peakCount = Mathf.Max(_peakCount, _active.Count);

    return obj;
}
```

**Purpose**:
Check out (rent) one object from the pool, position and rotate it, activate it, and return it. If no object is available, either expand (if allowed) or log a warning and return `null`.

**Step-by-Step**:

1. **No Available & Can Expand**

   * If `_available.Count == 0` and `AutoExpand == true` and `TotalCount < MaxSize`:

     * Call `CreateNewObject()` to produce a fresh instance (disabled by default), skip the queue.

2. **Available Exists**

   * Otherwise, if `_available.Count > 0`, `obj = _available.Dequeue()`.

3. **No Available & Cannot Expand**

   * If no available and either `AutoExpand == false` or `TotalCount >= MaxSize`, log a warning:

     ```csharp
     Debug.LogWarning($"Pool for {_prefab.name} is empty and cannot expand!");
     ```

     and return `null`.

4. **Position & Activate**

   * Set `obj.transform.position = position`, `obj.transform.rotation = rotation`, and `obj.SetActive(true)`.
   * Add `obj` to `_active`.

5. **Update Peak**

   * `_peakCount = Mathf.Max(_peakCount, _active.Count)` tracks the highest concurrent usage.

6. **Return**

   * Returns the activated `GameObject` for use.

**Example**:

```csharp
GameObject spawnedEnemy = ObjectPoolHelper.Get("EnemyPool", spawnPosition, Quaternion.identity);
if (spawnedEnemy != null)
{
    // e.g., initialize health, AI scripts, etc.
}
```

* If the pool is empty but can expand, a new instance is created. Otherwise `null` is returned and you can handle that scenario.

---

### 5.6 `Return(GameObject obj)`

```csharp
public void Return(GameObject obj)
{
    if (_active.Remove(obj))
    {
        obj.SetActive(false);
        _available.Enqueue(obj);
    }
}
```

**Purpose**:
Return (release) a previously checked out object back into the pool. Deactivate it and mark it as available.

**Behavior**:

1. **Remove from `_active`**

   * If the object was indeed rented (present in the `_active` set), remove it from `_active`.
   * If it was not active (perhaps already returned or never from this pool), this code does nothing.

2. **Deactivate & Enqueue**

   * `obj.SetActive(false)` disables the GameObject.
   * `_available.Enqueue(obj)` adds it to the queue for future `Get()` calls.

**Example**:

```csharp
public class Enemy : MonoBehaviour
{
    private void OnDeath()
    {
        // Return this enemy to the pool instead of destroying
        GetComponent<PoolableObject>().ReturnToPool();
    }
}
```

* When the enemy “dies,” instead of `Destroy(gameObject)`, we `ReturnToPool()`, reusing the instance later.

---

### 5.7 `Clear()`

```csharp
public void Clear()
{
    foreach (var obj in _available.Concat(_active))
    {
        if (obj != null)
            GameObject.Destroy(obj);
    }
    _available.Clear();
    _active.Clear();
}
```

**Purpose**:
Destroy all instances in the pool—both active and available—and clear internal data structures.

**Behavior**:

1. **Iterate Over Both Queues**

   * `_available.Concat(_active)` enumerates all GameObjects in the pool.
   * For each, if `obj != null`, call `GameObject.Destroy(obj)`.

     * This removes it from the scene entirely.

2. **Clear Collections**

   * `_available.Clear()` empties the queue.
   * `_active.Clear()` empties the hash set.

**Example**:

```csharp
public class LevelCleanup : MonoBehaviour
{
    private void OnDestroy()
    {
        // When this level is done, completely wipe the bullet pool
        ObjectPoolHelper.ClearPool("BulletPool");
    }
}
```

* Ensures no bullet instances remain in memory when the level unloads.

---

### 5.8 `CreateNewObject()`

```csharp
private GameObject CreateNewObject()
{
    var obj = GameObject.Instantiate(_prefab);
    var poolable = obj.GetComponent<PoolableObject>();
    if (poolable == null)
    {
        poolable = obj.AddComponent<PoolableObject>();
    }
    obj.SetActive(false);
    poolable.PoolKey = /* The key used by the containing ObjectPool */ ;
    return obj;
}
```

**Purpose**:
Instantiate a fresh clone of the `_prefab`, add `PoolableObject` if missing, set its `PoolKey`, deactivate it, and return it for pooling.

**Behavior**:

1. **`Instantiate(_prefab)`**

   * Clones the prefab, including all child objects, scripts, etc.

2. **Ensure `PoolableObject` Exists**

   * If `obj.GetComponent<PoolableObject>()` returns `null`, add the component so the object “knows” its pool key.

3. **Set `PoolKey`**

   * Assign the owning pool’s `poolKey` (captured from the container). This ensures `ReturnToPool()` can find the correct pool.

4. **Deactivate**

   * `obj.SetActive(false)`, so it’s queued in `_available` as an idle instance.

5. **Return**

   * The brand new, inactive GameObject.

**Example**:

* Internally called when `Get()` needs to instantiate more and when pre‐warming.

---

## 6. Helper Classes & Structs

### 6.1 `PoolSettings`

```csharp
public class PoolSettings
{
    public int InitialSize { get; set; }
    public int MaxSize { get; set; }
    public int ExpandBy { get; set; }
    public bool AutoExpand { get; set; }
}
```

* **`InitialSize`**: How many instances to pre‐allocate when the pool is created.
* **`MaxSize`**: The maximum total instances (active + available) the pool may hold.
* **`ExpandBy`**: When more objects are needed and `AutoExpand == true`, how many new instances to create at once.
* **`AutoExpand`**: If `true`, the pool will automatically grow by `ExpandBy` when all existing instances are in use and `TotalCount < MaxSize`. If `false`, once the pool is empty, `Get()` returns `null` without creating new ones.

### 6.2 `PoolStats`

```csharp
public struct PoolStats
{
    public int TotalObjects { get; set; }
    public int ActiveObjects { get; set; }
    public int AvailableObjects { get; set; }
    public int PeakObjects { get; set; }
}
```

* **`TotalObjects`**: Sum of active + available.
* **`ActiveObjects`**: How many are currently checked out (in use).
* **`AvailableObjects`**: How many are idle.
* **`PeakObjects`**: Historical maximum concurrent active count.

### 6.3 `PoolableObject`

```csharp
public class PoolableObject : MonoBehaviour
{
    public string PoolKey { get; set; }

    public void ReturnToPool()
    {
        ObjectPoolHelper.Return(gameObject);
    }
}
```

* A simple component that carries:

  * **`PoolKey`**: The string identifying which pool this object belongs to.
  * **`ReturnToPool()`**: Convenience method to call `ObjectPoolHelper.Return(this.gameObject)`.

In `CreateNewObject()`, `PoolableObject` is added automatically if missing, and `PoolKey` is set so that `ReturnToPool()` knows where to send it.

---

## 7. Real‐World Usage Examples

Below are step‐by‐step examples demonstrating how you might use `ObjectPoolHelper` in a typical Unity game.

### 7.1 Bootstrap & Initialize a Simple Pool

```csharp
using UnityEngine;

public class BulletPoolManager : MonoBehaviour
{
    [SerializeField] private GameObject _bulletPrefab;

    private async void Awake()
    {
        bool success = await ObjectPoolHelper.InitializePoolAsync(
            poolKey: "BulletPool",
            prefab: _bulletPrefab,
            settings: new ObjectPoolHelper.PoolSettings
            {
                InitialSize = 50,
                MaxSize = 500,
                ExpandBy = 50,
                AutoExpand = true
            }
        );

        if (success)
            Debug.Log("BulletPool initialized with 50 bullets.");
        else
            Debug.LogError("Failed to initialize BulletPool!");
    }
}
```

**What Happens**:

* On startup, pre‐allocates 50 inactive bullet instances over 50 frames (one per `Task.Yield()`).
* If at runtime you request more than 50 bullets concurrently, the pool auto‐expands by 50 each time until it hits the 500 max.

---

### 7.2 Fetching & Returning Objects at Runtime

```csharp
public class PlayerGun : MonoBehaviour
{
    [SerializeField] private Transform _muzzleTransform;
    [SerializeField] private float _bulletSpeed = 20f;

    public void Fire()
    {
        // 1. Get a bullet from the pool
        GameObject bullet = ObjectPoolHelper.Get("BulletPool", _muzzleTransform.position, _muzzleTransform.rotation);
        if (bullet == null)
        {
            Debug.LogWarning("No bullets available!");
            return;
        }

        // 2. Configure bullet (e.g., set velocity)
        Rigidbody rb = bullet.GetComponent<Rigidbody>();
        rb.velocity = _muzzleTransform.forward * _bulletSpeed;

        // 3. Set up returning bullet when it hits something or after 5 seconds
        StartCoroutine(ReturnBulletAfterTime(bullet, 5f));
    }

    private IEnumerator ReturnBulletAfterTime(GameObject bullet, float delay)
    {
        yield return new WaitForSeconds(delay);
        // Return to pool automatically
        bullet.GetComponent<PoolableObject>().ReturnToPool();
    }
}
```

* **`ObjectPoolHelper.Get(...)`**: Retrieves an inactive bullet, places it at the muzzle, and activates it.
* **`ReturnBulletAfterTime(...)`**: After 5 seconds, returns the bullet to the pool (disabling it). In a real game, you’d likely return it on collision instead.

---

### 7.3 Pre‐Warming a Pool in Advance

```csharp
public class EnemySpawner : MonoBehaviour
{
    private async void Start()
    {
        // Suppose “EnemyPool” was already initialized with InitialSize=10
        // Now ensure we have 30 available before starting waves
        await ObjectPoolHelper.PreWarmAsync("EnemyPool", 20);
        Debug.Log("EnemyPool pre-warmed with 20 additional instances (total 30).");

        // Begin spawning waves…
        StartCoroutine(SpawnWaves());
    }

    private IEnumerator SpawnWaves()
    {
        // Wave 1: 5 enemies
        for (int i = 0; i < 5; i++)
        {
            GameObject enemy = ObjectPoolHelper.Get("EnemyPool", GetRandomSpawnPos(), Quaternion.identity);
            // configure enemy...
            yield return new WaitForSeconds(0.5f);
        }

        // Later waves…
    }

    private Vector3 GetRandomSpawnPos()
    {
        // Return a random point in the scene
        return new Vector3(UnityEngine.Random.Range(-10f, 10f), 0f, UnityEngine.Random.Range(-10f, 10f));
    }
}
```

* Ensures “EnemyPool” has at least 30 instances (10 initial + 20 prewarmed) before any enemies spawn, so the first wave never experiences on‐the‐fly instantiation hitches.

---

### 7.4 Loading a Prefab from an AssetBundle into a Pool

```csharp
public class BossManager : MonoBehaviour
{
    private async void Awake()
    {
        bool success = await ObjectPoolHelper.InitializePoolFromBundleAsync(
            poolKey: "BossMinions",
            bundleName: "bossassets",
            prefabName: "MinionPrefab",
            settings: new ObjectPoolHelper.PoolSettings
            {
                InitialSize = 5,
                MaxSize = 20,
                ExpandBy = 5,
                AutoExpand = true
            }
        );

        if (!success)
            Debug.LogError("Failed to initialize BossMinions pool from AssetBundle!");
    }

    private void SpawnMinion()
    {
        Vector3 spawnPos = transform.position + UnityEngine.Random.insideUnitSphere * 3f;
        GameObject minion = ObjectPoolHelper.Get("BossMinions", spawnPos, Quaternion.identity);
        if (minion != null)
        {
            // Configure minion behavior…
        }
    }
}
```

* This example loads “bossassets” bundle, fetches “MinionPrefab,” and initializes a pool of 5 minions to begin.
* If you call `SpawnMinion()` 6 times before any are returned, the pool auto‐expands by 5 (to total 10), up to a maximum of 20.

---

### 7.5 Querying Pool Statistics for Debugging

```csharp
public class PoolDebugger : MonoBehaviour
{
    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.I))
        {
            ObjectPoolHelper.PoolStats stats = ObjectPoolHelper.GetPoolStats("EnemyPool");
            if (stats.TotalObjects == 0)
            {
                Debug.Log("EnemyPool not found or empty.");
            }
            else
            {
                Debug.Log($"EnemyPool Stats → Total: {stats.TotalObjects}, Active: {stats.ActiveObjects}, Available: {stats.AvailableObjects}, Peak: {stats.PeakObjects}");
            }
        }
    }
}
```

* Press “I” during play mode to print the current and peak usage of “EnemyPool,” helping you understand whether you need to increase `InitialSize` or `MaxSize`.

---

### 7.6 Clearing Pools on Level Unload

```csharp
public class LevelManager : MonoBehaviour
{
    private void OnDisable()
    {
        // Clear all pools when switching levels
        ObjectPoolHelper.ClearAllPools();
        Debug.Log("All object pools cleared.");
    }
}
```

* Ensures that when the player returns to main menu or loads a new level, all pooled GameObjects are destroyed and memory is freed.

---

## 8. Customization & Tips

1. **Adjust `PoolSettings` to Your Needs**

   * If you expect a large number of concurrent objects (e.g., 200 bullets in a bullet-hell), set `MaxSize` accordingly and bump `ExpandBy` to a higher number to reduce expansion frequency.

2. **Avoid “Stutter” on Expansion**

   * Because `ExpandAsync` does `await Task.Yield()` each iteration, expansion is spread over multiple frames. If you expand by a large number (e.g., `ExpandBy = 100`), that could take 100 frames to finish. Consider smaller increments or pre‐warming in the background during loading screens.

3. **Ensure Prefabs Have `PoolableObject`**

   * If your prefab already has a `PoolableObject` component in the Editor, you avoid runtime `AddComponent(...)`. Just make sure `PoolKey` is filled by `CreateNewObject()`.
   * If your prefab has children with scripts that rely on `OnEnable()`, remember that pooled objects are disabled/enabled repeatedly. Ensure your components respond correctly to those state changes.

4. **Multiple Pools for the Same Prefab**

   * You could (rarely) create two different pools referencing the same prefab under different keys (e.g., `"BulletPool"` and `"ExplosionPool"`). Just be aware that each pool has its own independent instances.

5. **Handling Destruction of Active Objects**

   * If you call `ClearPool`, it destroys everything, including active objects. Ensure you stop referencing those objects afterward. Accessing a destroyed `GameObject` leads to null exceptions.

6. **Integrate With Scene “Reset” Logic**

   * For levels that reset frequently, call `ObjectPoolHelper.ClearAllPools()` or selectively clear pools to avoid leftover objects.

7. **Thread‐Safety & Async Behavior**

   * All asynchronous methods (`InitializePoolAsync`, `InitializePoolFromBundleAsync`, `PreWarmAsync`) use `async Task` and `await Task.Yield()`. This ensures no blocking on the main thread.
   * Don’t call pool methods before the scene has fully loaded or before Unity’s `Start()`—invoke them from `Awake()` or after.

8. **Using `PoolableObject.ReturnToPool()` Internally**

   * By attaching `PoolableObject` to pooled prefabs, you can call `obj.GetComponent<PoolableObject>().ReturnToPool()` anywhere (e.g., on collision, on timer) instead of referencing `ObjectPoolHelper` directly.

---

## 9. Summary of Public API

| Method Signature                                                                                                                                   | Description                                                                                                                                                                         | Example Use                                                                                                                                                                                   |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `public static async Task<bool> InitializePoolAsync(string poolKey, GameObject prefab, PoolSettings settings = null)`                              | Create a new pool for `prefab` under `poolKey`. Pre‐allocates `settings.InitialSize` instances asynchronously. Returns `true` if successful.                                        | `await ObjectPoolHelper.InitializePoolAsync("BulletPool", bulletPrefab, new PoolSettings { InitialSize = 50, MaxSize = 500, ExpandBy = 50, AutoExpand = true });`                             |
| `public static async Task<bool> InitializePoolFromBundleAsync(string poolKey, string bundleName, string prefabName, PoolSettings settings = null)` | Load `bundleName`, fetch `prefabName` as a `GameObject`, then call `InitializePoolAsync(poolKey, prefab, settings)`. Returns `true` on success.                                     | `await ObjectPoolHelper.InitializePoolFromBundleAsync("EnemyPool", "enemyBundle", "SkeletonPrefab", new PoolSettings { InitialSize = 10, MaxSize = 100, ExpandBy = 10, AutoExpand = true });` |
| `public static GameObject Get(string poolKey, Vector3 position = default, Quaternion rotation = default)`                                          | Retrieve a pooled instance from `poolKey`, place it at `position`, rotate to `rotation`, activate it, and return it. If none are available (and no auto‐expansion), returns `null`. | `GameObject bullet = ObjectPoolHelper.Get("BulletPool", muzzlePosition, muzzleRotation); if (bullet != null) { /* use it */ }`                                                                |
| `public static void Return(GameObject obj)`                                                                                                        | Return a previously fetched `obj` (with `PoolableObject` attached) back into its pool, disabling it and marking it as available. If the pool no longer exists, destroys the `obj`.  | In `OnBecameInvisible()` of a bullet: `GetComponent<PoolableObject>().ReturnToPool();`                                                                                                        |
| `public static async Task PreWarmAsync(string poolKey, int count)`                                                                                 | Pre‐allocate (and disable) up to `count` additional instances in the specified pool. Spreads instantiation across frames.                                                           | `await ObjectPoolHelper.PreWarmAsync("EnemyPool", 20);`                                                                                                                                       |
| `public static PoolStats GetPoolStats(string poolKey)`                                                                                             | Returns statistics (`TotalObjects`, `ActiveObjects`, `AvailableObjects`, `PeakObjects`) for the specified pool. Returns `default(PoolStats)` if the pool doesn’t exist.             | `var stats = ObjectPoolHelper.GetPoolStats("BulletPool"); Debug.Log($"Active: {stats.ActiveObjects}, Available: {stats.AvailableObjects}");`                                                  |
| `public static void ClearPool(string poolKey)`                                                                                                     | Destroy all instances (active & available) in the specified pool and remove the pool from the helper.                                                                               | `ObjectPoolHelper.ClearPool("EnemyPool");`                                                                                                                                                    |
| `public static void ClearAllPools()`                                                                                                               | Destroy all instances in all pools and clear the internal dictionary of pools.                                                                                                      | `ObjectPoolHelper.ClearAllPools();`                                                                                                                                                           |

Additionally, the helper classes used by this API:

| Class / Struct                                | Purpose                                                                                                                                               |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `private class ObjectPool`                    | Internal per‐prefab pool that manages creatation, retrieval, return, and clearing of `GameObject` instances.                                          |
| `public class PoolSettings`                   | Encapsulates `InitialSize`, `MaxSize`, `ExpandBy`, and `AutoExpand` for customizing pool behavior.                                                    |
| `public struct PoolStats`                     | Simple data struct containing `TotalObjects`, `ActiveObjects`, `AvailableObjects`, and `PeakObjects`.                                                 |
| `public class PoolableObject : MonoBehaviour` | Component that tags a pooled object with its `PoolKey` (string) and provides a `ReturnToPool()` method calling `ObjectPoolHelper.Return(gameObject)`. |

---

### With this documentation and the provided examples, you have everything you need to:

1. **Create** various pools (bullets, enemies, UI popups) at runtime or via AssetBundles.
2. **Fetch** objects (`Get(...)`) when you need to display or spawn them.
3. **Return** objects (`Return(...)` or `PoolableObject.ReturnToPool()`) when you’re done, making them available for future reuse.
4. **Monitor** pools at runtime to tune your `PoolSettings`.
5. **Clear** pools on level unload or game exit to free up memory.

By using **ObjectPoolHelper**, you reduce `Instantiate`/`Destroy` overhead, avoid spikes in framerate, and keep your code clean with simple, centralized pool management. Enjoy building your optimized, high‐performance Unity game!
