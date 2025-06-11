**Generic Factory with Integrated Pooling in Unity**

This document explains the **`Factory<TProduct>`** class that combines a generic factory pattern with **object pooling** support for GameObjects, plus extensive real-world Unity examples illustrating each method.

---

## Table of Contents

1. [Overview](#overview)
2. [API Reference](#api-reference)

   * `RegisterProduct`
   * `RegisterPooledProduct`
   * `UnregisterProduct`
   * `CreateProduct`
   * `ReturnProduct`
   * `PreWarmPoolAsync`
   * `GetPoolStats`
   * `GetRegisteredTypes` & `GetPooledTypes`
3. [Example Scenarios](#example-scenarios)

   1. Non‑Pooled Enemy Spawning
   2. Pooled Bullet System
   3. Pooled Visual Effects (VFX)
   4. UI Window Pooling
   5. NPC Crowd Manager
4. [Best Practices & Edge Cases](#best-practices)
5. [Full Code Listing](#full-code)

---

## 1. Overview

`Factory<TProduct>` extends the classic Factory pattern by integrating **Unity GameObject pooling** via `ObjectPoolHelper`. It allows you to:

* Register simple creators (`Func<TProduct>`) for one‑off instantiation.
* Register pooled GameObject types to reuse instances and reduce GC.
* Create, return, and pre‑warm pools asynchronously.
* Inspect current pool statistics.

Applicable when you want both flexible factory creation and high‑performance pooling in a single abstraction.

---

## 2. API Reference

### `void RegisterProduct(string productType, Func<TProduct> creator)`

Registers a non‑pooled creator. Logs a warning if `productType` already exists.

```csharp
factory.RegisterProduct("Goblin", () => new Goblin());
```

---

### `Task RegisterPooledProduct(string productType, GameObject prefab, PoolSettings settings = null)`

Asynchronously initializes a pool for the given prefab, then registers a creator that **fetches** from the pool. Only valid when `TProduct == GameObject`.

```csharp
await factory.RegisterPooledProduct(
    "Bullet",
    bulletPrefab,
    new PoolSettings { initialSize = 20, maxSize = 100 }
);
```

---

### `void UnregisterProduct(string productType)`

Removes the creator. If pooled, also **clears** its entire pool.

```csharp
factory.UnregisterProduct("Bullet"); // Destroys pool
```

---

### `TProduct CreateProduct(string productType)`

Returns a new or pooled instance. Logs an error if `productType` not registered.

```csharp
GameObject vfx = factory.CreateProduct("ExplosionVFX");
```

---

### `void ReturnProduct(string productType, TProduct product)`

Returns a pooled GameObject back to its pool. Warns if the type isn’t pooled.

```csharp
factory.ReturnProduct("Bullet", bulletInstance);
```

---

### `Task PreWarmPoolAsync(string productType, int count)`

Fills the pool with `count` extra instances ahead of time.

```csharp
await factory.PreWarmPoolAsync("EnemyNPC", 10);
```

---

### `PoolStats GetPoolStats(string productType)`

Retrieves current statistics: active, inactive, total counts.

```csharp
var stats = factory.GetPoolStats("Bullet");
Debug.Log($"Bullets: active={stats.activeCount}, pooled={stats.inactiveCount}");
```

---

### `IEnumerable<string> GetRegisteredTypes()` & `GetPooledTypes()`

List all keys for simple and pooled registrations.

```csharp
foreach (var key in factory.GetRegisteredTypes()) Debug.Log(key);
foreach (var key in factory.GetPooledTypes()) Debug.Log("Pooled: " + key);
```

---

## 3. Example Scenarios

### 3.1 Non‑Pooled Enemy Spawning

```csharp
// Enemy classes
public class Enemy : MonoBehaviour {}
public class Goblin : Enemy {}
public class Orc : Enemy {}

// Factory setup
public class EnemyFactory : Factory<GameObject> {
    public EnemyFactory(Transform parent) {
        RegisterProduct("Goblin", () =>
            GameObject.Instantiate(goblinPrefab, parent));
        RegisterProduct("Orc", () =>
            GameObject.Instantiate(orcPrefab, parent));
    }
}
```

**Usage**:

```csharp
var factory = new EnemyFactory(spawnRoot);
var goblin = factory.CreateProduct("Goblin");
```

### 3.2 Pooled Bullet System

```csharp
// Setup factory in Shooter
var bulletFactory = new Factory<GameObject>();
await bulletFactory.RegisterPooledProduct(
    "Bullet",
    bulletPrefab,
    new PoolSettings { initialSize = 30, maxSize = 200 }
);
```

**Firing**:

```csharp
void Fire() {
    var bullet = bulletFactory.CreateProduct("Bullet");
    bullet.transform.position = muzzlePosition;
    bullet.GetComponent<Rigidbody>().velocity = direction * speed;
}
```

**Returning** (e.g., on collision):

```csharp
void OnBulletCollision(GameObject bullet) {
    bulletFactory.ReturnProduct("Bullet", bullet);
}
```

### 3.3 Pooled Visual Effects (VFX)

```csharp
await vfxFactory.RegisterPooledProduct(
    "ExplosionVFX",
    explosionPrefab,
    new PoolSettings { initialSize = 5, maxSize = 20 }
);

void SpawnExplosion(Vector3 pos) {
    var vfx = vfxFactory.CreateProduct("ExplosionVFX");
    vfx.transform.position = pos;
    StartCoroutine(ExpireVFX(vfx));
}

IEnumerator ExpireVFX(GameObject vfx) {
    yield return new WaitForSeconds(1);
    vfxFactory.ReturnProduct("ExplosionVFX", vfx);
}
```

### 3.4 UI Window Pooling

```csharp
// For heavy UI windows reuse
await uiFactory.RegisterPooledProduct(
    "InventoryWindow", inventoryWindowPrefab
);

void ShowInventory() {
    var window = uiFactory.CreateProduct("InventoryWindow");
    window.GetComponent<InventoryUI>().Open();
}

void CloseInventory(GameObject window) {
    uiFactory.ReturnProduct("InventoryWindow", window);
}
```

### 3.5 NPC Crowd Manager

```csharp
// Pre‑warm 50 NPCs for city crowd
await npcFactory.RegisterPooledProduct(
    "CitizenNPC",
    citizenPrefab,
    new PoolSettings { initialSize = 50, maxSize = 200 }
);

// Spawn 20 random citizens
for (int i = 0; i < 20; i++) {
    var npc = npcFactory.CreateProduct("CitizenNPC");
    npc.transform.position = RandomPointInCity();
}
```

---

## 4. Best Practices & Edge Cases

* **TProduct constraint**: Only GameObjects may be pooled.
* **Error Handling**: Always check `CreateProduct` return value for `null`.
* **Pool Growth**: Set sensible `maxSize` to prevent runaway allocations.
* **Clean‑up**: Call `UnregisterProduct` when no longer needed to free memory.
* **Threading**: All Unity API calls remain on the main thread; `RegisterPooledProduct` awaits pool initialization but does not spawn on background threads.

---

## 5. Full Code Listing

```csharp
public abstract class Factory<TProduct> where TProduct : class {
    protected readonly Dictionary<string, Func<TProduct>> _creators = new Dictionary<string, Func<TProduct>>();
    protected readonly HashSet<string> _pooledTypes = new HashSet<string>();

    public void RegisterProduct(string productType, Func<TProduct> creator) { /* ... */ }
    public async Task RegisterPooledProduct(string productType, GameObject prefab, PoolSettings settings = null) { /* ... */ }
    public void UnregisterProduct(string productType) { /* ... */ }
    public virtual TProduct CreateProduct(string productType) { /* ... */ }
    public virtual void ReturnProduct(string productType, TProduct product) { /* ... */ }
    public virtual async Task PreWarmPoolAsync(string productType, int count) { /* ... */ }
    public virtual PoolStats GetPoolStats(string productType) { /* ... */ }
    public IEnumerable<string> GetRegisteredTypes() => _creators.Keys;
    public IEnumerable<string> GetPooledTypes() => _pooledTypes;
}
```

With these examples, you can integrate both factories and high‑performance pooling cleanly into your Unity projects.