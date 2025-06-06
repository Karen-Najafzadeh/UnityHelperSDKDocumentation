Below is a comprehensive guide to the **ParticleHelper** utility. It covers how to set up and use its particle‐system pooling, runtime modification, presets, performance optimization, and cleanup features. For each important method, you’ll find:

* A description of what it does
* Its parameters and behavior
* A real‐world (in‐game) usage example

---

## Overview

`ParticleHelper` is a static helper that centralizes common particle‐system tasks in Unity, specifically:

1. **Pooling**: Create and manage pools of reusable `ParticleSystem` instances to avoid excessive instantiation overhead at runtime.
2. **Playback**: Play particle effects from a pool at a given position/rotation, optionally returning them after a fixed duration.
3. **Modification & Presets**: Define “presets” (configurations) for particle‐system properties (color, size, emission rate, shape) and apply them at runtime. Also expose a simple API to change individual properties on the fly.
4. **Performance**: Automatically detect when active particles finish, then return them to their pool for reuse.
5. **Cleanup**: Provide a way to sweep inactive particle systems and return them to their pools, avoiding “dead” particles accumulating in memory.

Underlying everything is a pair of key collections:

* `_particlePools: Dictionary<string, Queue<ParticleSystem>>`

  * Each “poolKey” (e.g. `"ExplosionFX"`) maps to a queue of inactive `ParticleSystem` objects.
* `_activeParticleSystems: HashSet<ParticleSystem>`

  * Tracks which particle systems have been “played” but not yet returned to their pool.

Additionally, `_presets: Dictionary<string, ParticlePreset>` stores named presets, and you can use `ParticleModifier` to tweak individual properties on the fly.

---

## 1. Pooling System

### 1.1 InitializePool

```csharp
public static void InitializePool(string poolKey, ParticleSystem prefab, int initialSize = 10)
```

**What it does**

* Creates (or expands) a pool under the given `poolKey`.
* Instantiates `initialSize` clones of the provided `prefab` (a `ParticleSystem`), disables them, and enqueues them into `_particlePools[poolKey]`.

**Parameters**

* `poolKey`: A unique string identifier for this pool (e.g. `"ExplosionFX"`, `"MagicSpellFX"`).
* `prefab`: The `ParticleSystem` prefab to clone.
* `initialSize`: Number of clones to pre‐instantiate (default = 10).

**Behavior**

* If `poolKey` does not already exist in `_particlePools`, a new `Queue<ParticleSystem>` is created.
* For `i = 0` … `initialSize - 1`, it:

  1. Calls `Instantiate(prefab)`.
  2. Disables the GameObject (`ps.gameObject.SetActive(false)`).
  3. Enqueues the instance into that pool.

Once initialized, that poolKey can be used with `PlayEffect(...)` to retrieve a ready‐to‐play particle system.

**Real‐World Example**

```csharp
// Suppose you have a “small explosion” particle prefab assigned in the inspector:
[SerializeField] private ParticleSystem smallExplosionPrefab;

private void Start() 
{
    // Prewarm a pool of 20 small explosions for later use:
    ParticleHelper.InitializePool("SmallExplosion", smallExplosionPrefab, 20);
}
```

* In this example, 20 deactivated clones of `smallExplosionPrefab` are created at game start and stored under key `"SmallExplosion"`.
* Whenever you need to spawn a small explosion, you’ll call `ParticleHelper.PlayEffect("SmallExplosion", position)` instead of instantiating a brand‐new prefab at runtime.

---

### 1.2 PlayEffect

```csharp
public static ParticleSystem PlayEffect(
    string poolKey,
    Vector3 position,
    Quaternion rotation = default,
    Transform parent = null,
    float duration = -1f)
```

**What it does**

* Retrieves a `ParticleSystem` instance from the pool identified by `poolKey`.
* If the pool is empty, instantiates a new clone based on the pool’s first prefab.
* Positions, rotates, and optionally parents it.
* Activates it, calls `Play()`, and adds it to `_activeParticleSystems`.
* If `duration > 0`, schedules it to automatically return to the pool after that many seconds.

**Parameters**

* `poolKey`: The key under which the pool was registered. Must match a previous `InitializePool` call.
* `position`: World‐space position to place the particle system.
* `rotation`: (Optional) Rotation to apply; defaults to `Quaternion.identity`.
* `parent`: (Optional) Transform under which to parent the particle. If non‐null, the particle becomes a child of this `Transform`.
* `duration`: (Optional, default = –1) If > 0, the helper will wait `duration` seconds (using `Task.Delay`) then call `ReturnToPool(ps, poolKey)`. If ≤ 0, you must manually call `ReturnToPool(...)` at some point (or rely on `CleanupInactiveParticles()`).

**Behavior**

1. Looks up `_particlePools[poolKey]`.
2. If the queue is non‐empty:

   * `Dequeue()` a `ParticleSystem` instance.
3. Else (pool was empty):

   * Instantiates a clone of the same prefab used to build the pool:

     ```csharp
     ps = GameObject.Instantiate(_particlePools[poolKey].Peek());
     ```
   * (Note: `Peek()` gives a reference prefab; using it as a template.)
4. Set `ps.transform.position = position;  
   ps.transform.rotation = rotation;  
   if (parent != null) ps.transform.SetParent(parent);`
5. `ps.gameObject.SetActive(true); ps.Play();`
6. `_activeParticleSystems.Add(ps);`
7. If `duration > 0`:

   * Call `ReturnToPoolAfterDuration(ps, poolKey, duration)` (which runs an `async` delay).

Finally, returns that `ParticleSystem` so you can keep a reference if you need to modify it immediately after playing.

**Real‐World Example**

```csharp
// Somewhere in your explosion code:
public void OnEnemyDeath(Vector3 position)
{
    // Play a small explosion at the enemy’s position. Automatically return to pool after 1.5 seconds.
    ParticleSystem ps = ParticleHelper.PlayEffect(
        poolKey: "SmallExplosion",
        position: position,
        rotation: Quaternion.identity,
        parent: null,
        duration: 1.5f
    );

    // Optionally tweak properties right away:
    ps.transform.localScale = Vector3.one * 1.2f;  // Make this particular explosion slightly bigger
}
```

* Because we passed `duration = 1.5f`, after 1.5 seconds the instance will be automatically returned to the “SmallExplosion” pool.

---

### 1.3 ReturnToPool

```csharp
public static void ReturnToPool(ParticleSystem ps, string poolKey)
```

**What it does**

* Immediately stops and deactivates a running `ParticleSystem`.
* Removes it from `_activeParticleSystems` and enqueues it back into `_particlePools[poolKey]`.

**Parameters**

* `ps`: The `ParticleSystem` instance to return.
* `poolKey`: The same key used to create its pool. This ensures it goes back into the correct queue.

**Behavior**

1. `ps.Stop();` stops emission immediately.
2. `ps.gameObject.SetActive(false);` hides the GameObject.
3. `ps.transform.SetParent(null);` (detaches from any parent).
4. If `_particlePools[poolKey]` exists, enqueue `ps`.
5. Remove from `_activeParticleSystems`.

**Real‐World Example**

```csharp
// If you want to manually return a looping “FireEffect” early:
public void ExtinguishFire(ParticleSystem firePs)
{
    ParticleHelper.ReturnToPool(firePs, "FireEffect");
}
```

* In this example, if you had a continuous fire effect from a pool keyed `"FireEffect"`, you stop it and return it immediately when the “fire” condition ends.

---

## 2. Particle Modification

### 2.1 ApplyPreset

```csharp
public static void ApplyPreset(ParticleSystem ps, string presetName)
```

**What it does**

* Looks up a named `ParticlePreset` in `_presets`.
* Copies its stored properties into the `ParticleSystem.MainModule`, `EmissionModule`, `ShapeModule`, etc.
* Allows you to define a variety of “preset” configurations at startup, then apply them to any pooled or non‐pooled system at runtime.

**Parameters**

* `ps`: The target `ParticleSystem` instance.
* `presetName`: Key into `_presets`. If that key is found, apply color, size, speed, maxParticles, emissionRate, shapeType.

**Behavior**

* `if (_presets.TryGetValue(presetName, out var preset)) { … }`.
* `var main = ps.main; main.startColor = preset.StartColor; main.startSize = preset.StartSize; main.startSpeed = preset.StartSpeed; main.maxParticles = preset.MaxParticles; var emission = ps.emission; emission.rateOverTime = preset.EmissionRate; var shape = ps.shape; shape.shapeType = preset.ShapeType;`

You can expand this to copy more properties (e.g. `gravityModifier`, `lifetime`, etc.) as needed.

**Real‐World Example**

```csharp
// 1) Define some presets at game initialization
void Awake()
{
    // “Fireball” preset: bright orange, large size, high emission rate
    ParticleHelper._presets["Fireball"] = new ParticleHelper.ParticlePreset
    {
        StartColor = new Color(1f, 0.5f, 0f),
        StartSize = 0.5f,
        StartSpeed = 2f,
        MaxParticles = 200,
        EmissionRate = 50f,
        ShapeType = ParticleSystemShapeType.Sphere
    };

    // “Smoke” preset
    ParticleHelper._presets["Smoke"] = new ParticleHelper.ParticlePreset
    {
        StartColor = new Color(0.2f, 0.2f, 0.2f, 0.8f),
        StartSize = 1.5f,
        StartSpeed = 0.5f,
        MaxParticles = 100,
        EmissionRate = 10f,
        ShapeType = ParticleSystemShapeType.Cone
    };
}

// 2) Later, when you acquire a pooled particle system:
ParticleSystem ps = ParticleHelper.PlayEffect("MagicSpell", position);

// 3) Immediately apply the “Fireball” preset to it:
ParticleHelper.ApplyPreset(ps, "Fireball");

// The effect now uses the Fireball color, size, speed, etc.
```

* By calling `ApplyPreset`, you instantly overwrite the core settings of the `ParticleSystem` before or after `Play()`.

---

### 2.2 ModifyParticles

```csharp
public static void ModifyParticles(ParticleSystem ps, ParticleModifier modifier)
```

**What it does**

* Takes a `ParticleModifier` object that may specify one or more of: `StartColor`, `StartSize`, `StartSpeed`, `EmissionRate`.
* If any field in `modifier` is non‐null, it updates the corresponding property on the `main` or `emission` modules of `ps`.

**Parameters**

* `ps`: The `ParticleSystem` to modify.
* `modifier`: A `ParticleModifier` instance; its nullable properties (`float?` and `Color?`) indicate which fields to change.

**Behavior**

* `var main = ps.main; if (modifier.StartColor.HasValue) main.startColor = modifier.StartColor.Value; if (modifier.StartSize.HasValue) main.startSize = modifier.StartSize.Value; …`
* `if (modifier.EmissionRate.HasValue) ps.emission.rateOverTime = modifier.EmissionRate.Value;`

This is useful for tweaking a few properties at runtime without needing a full preset.

**Real‐World Example**

```csharp
// Suppose you grabbed a base “Smoke” effect but want to fade it out gradually:
ParticleSystem smokePs = ParticleHelper.PlayEffect("SmokePool", position);

// After 1 second, reduce its emission rate to zero to let the existing particles dissipate:
await Task.Delay(1000);
ParticleHelper.ModifyParticles(smokePs, new ParticleHelper.ParticleModifier
{
    EmissionRate = 0f  // reduce emission to zero
});

// Optionally, after another 2 seconds, return it to the pool:
await Task.Delay(2000);
ParticleHelper.ReturnToPool(smokePs, "SmokePool");
```

* Here, we dynamically zero out the emission rather than immediately stopping the particle system.

---

## 3. Performance Optimization & Cleanup

### 3.1 CleanupInactiveParticles

```csharp
public static void CleanupInactiveParticles()
```

**What it does**

* Iterates through `_activeParticleSystems`. For each `ps` that is no longer alive (i.e. `ps.IsAlive() == false`), it finds the correct pool in `_particlePools` that contains it and calls `ReturnToPool(ps, thatPoolKey)`.

**Behavior**

1. `foreach (var ps in _activeParticleSystems.ToArray())`:

   * `if (!ps.IsAlive())`:

     * Search each `poolKey`/`Queue<ParticleSystem>` in `_particlePools` and check `pool.Value.Contains(ps)`.
     * When found, call `ReturnToPool(ps, pool.Key)` and break.

**When to call it**

* Typically, call `CleanupInactiveParticles()` each frame (or every few frames) from a manager script—e.g., in `Update()`—so that finished particle systems automatically get returned.

**Real‐World Example**

```csharp
public class ParticleManager : MonoBehaviour
{
    private void Update()
    {
        // Every frame, sweep for inactive particles and return them to their pools
        ParticleHelper.CleanupInactiveParticles();
    }
}
```

* In many projects, you attach a small “ParticleManager” MonoBehaviour to an empty GameObject. Its `Update()` calls `ParticleHelper.CleanupInactiveParticles()` every frame. As soon as a particle system has finished playing (no live particles), it’s detected and returned to the pool automatically.

---

## 4. Helper Methods

### 4.1 ReturnToPoolAfterDuration (private)

```csharp
private static async void ReturnToPoolAfterDuration(ParticleSystem ps, string poolKey, float duration)
```

**What it does**

* Internally used by `PlayEffect(...)` when you pass a `duration > 0`.
* Awaits `Task.Delay(duration * 1000)` milliseconds, then calls `ReturnToPool(ps, poolKey)`.

**Behavior**

* Because it’s `async void`, it runs alongside the main thread without blocking.
* If you pass, for example, `duration = 2.5f`, after about 2.5 seconds it will deactivate and return that `ps` to its pool.

**Real‐World Note**

* Avoid using `async void` outside helper methods—here it’s acceptable because it’s purely a “fire‐and‐forget” return scheduler.
* If you need more precise control (e.g., you want to wait until all ongoing particles are gone), it’s better to rely on `CleanupInactiveParticles()` instead of a fixed duration.

---

## 5. Helper Classes

### 5.1 ParticlePreset

```csharp
public class ParticlePreset
{
    public Color StartColor { get; set; }
    public float StartSize { get; set; }
    public float StartSpeed { get; set; }
    public int MaxParticles { get; set; }
    public float EmissionRate { get; set; }
    public ParticleSystemShapeType ShapeType { get; set; }
}
```

* **`StartColor`**: The initial main module color (e.g. `new Color(1f, 0.2f, 0.2f)` for red).
* **`StartSize`**: The initial particle size.
* **`StartSpeed`**: How fast particles are emitted initially.
* **`MaxParticles`**: Maximum number of simultaneous particles.
* **`EmissionRate`**: `emission.rateOverTime`—how many particles per second.
* **`ShapeType`**: The `ParticleSystemShapeType` (e.g. `Sphere`, `Cone`, `Box`) to use in the `ShapeModule`.

*Define these at startup in a dictionary:*

```csharp
ParticleHelper._presets["BloodSplatter"] = new ParticleHelper.ParticlePreset
{
   StartColor = new Color(0.6f, 0f, 0f),
   StartSize = 0.3f,
   StartSpeed = 3f,
   MaxParticles = 50,
   EmissionRate = 30f,
   ShapeType = ParticleSystemShapeType.Cone
};
```

### 5.2 ParticleModifier

```csharp
public class ParticleModifier
{
    public Color? StartColor { get; set; }
    public float? StartSize { get; set; }
    public float? StartSpeed { get; set; }
    public float? EmissionRate { get; set; }
}
```

* Each property is nullable (`Color?` or `float?`).
* When you pass a `ParticleModifier` to `ModifyParticles`, only those fields that have a `Value` will be copied.

*Usage example:*

```csharp
var modifier = new ParticleHelper.ParticleModifier
{
    StartColor = Color.green,       // change color to green
    EmissionRate = 5f               // slow down emission
};

// Apply to an existing particle system:
ParticleHelper.ModifyParticles(firePs, modifier);
```

---

## 6. Putting It All Together

Below is a sample workflow showcasing these methods in a hypothetical combat scenario:

```csharp
public class CombatEffectsManager : MonoBehaviour
{
    [SerializeField] private ParticleSystem explosionPrefab;
    [SerializeField] private ParticleSystem smokePrefab;

    private void Start()
    {
        // 1. Initialize pools for two effect types
        ParticleHelper.InitializePool("Explosion", explosionPrefab, initialSize: 15);
        ParticleHelper.InitializePool("Smoke", smokePrefab, initialSize: 10);

        // 2. Define presets
        ParticleHelper._presets["BigExplosion"] = new ParticleHelper.ParticlePreset
        {
            StartColor = new Color(1f, 0.4f, 0f),
            StartSize = 2f,
            StartSpeed = 4f,
            MaxParticles = 300,
            EmissionRate = 100f,
            ShapeType = ParticleSystemShapeType.Sphere
        };

        ParticleHelper._presets["SmallSmoke"] = new ParticleHelper.ParticlePreset
        {
            StartColor = new Color(0.2f, 0.2f, 0.2f),
            StartSize = 1f,
            StartSpeed = 0.5f,
            MaxParticles = 80,
            EmissionRate = 20f,
            ShapeType = ParticleSystemShapeType.Cone
        };
    }

    private void Update()
    {
        // 3. Continuously clean up any finished particle systems
        ParticleHelper.CleanupInactiveParticles();
    }

    public async void OnEnemyKilled(Vector3 position)
    {
        // 4. Play a “BigExplosion” effect at enemy position:
        ParticleSystem explosionPs = ParticleHelper.PlayEffect(
            "Explosion",
            position,
            Quaternion.identity,
            parent: null,
            duration: 2f  // auto‐return to pool after 2 seconds
        );

        // 5. Immediately apply the “BigExplosion” preset to configure color, emission, etc.
        ParticleHelper.ApplyPreset(explosionPs, "BigExplosion");

        // 6. After 0.5s, spawn some smoke rising from the same position:
        await Task.Delay(500);
        ParticleSystem smokePs = ParticleHelper.PlayEffect(
            "Smoke",
            position + Vector3.up * 0.5f,
            Quaternion.identity,
            parent: null,
            duration: 3f
        );
        ParticleHelper.ApplyPreset(smokePs, "SmallSmoke");

        // 7. After 1.5s, reduce smoke emission so it fades out:
        await Task.Delay(1500);
        ParticleHelper.ModifyParticles(smokePs, new ParticleHelper.ParticleModifier
        {
            EmissionRate = 0f  // let existing smoke dissipate naturally
        });

        // 8. Finally, manually return explosionPs if you want to kill it early:
        //    (optional—otherwise it auto‐returned in 2s)
        // ParticleHelper.ReturnToPool(explosionPs, "Explosion");
    }
}
```

**Explanation of workflow**

1. **Start()**

   * Two pools are created: `"Explosion"` (15 initial clones) and `"Smoke"` (10 clones).
   * Two named presets (`"BigExplosion"`, `"SmallSmoke"`) are registered for later use.
2. **Update()**

   * Each frame, `CleanupInactiveParticles()` checks `_activeParticleSystems`. If any particle systems have finished emitting and are no longer alive, they’re returned to their appropriate pool.
3. **OnEnemyKilled(Vector3)**

   * Calls `PlayEffect("Explosion", position, …, duration: 2f)`. The explosion is activated and will be automatically returned after 2 seconds.
   * Applies the `"BigExplosion"` preset (orange color, large size, high emission).
   * After 0.5 seconds, spawns a smoke system at a slightly raised position, applies the `"SmallSmoke"` preset, and schedules it to auto‐return after 3 seconds.
   * After 1.5 seconds (while smoke is still active), calls `ModifyParticles(smokePs, new ParticleModifier { EmissionRate = 0f })`—this stops new smoke but lets existing particles fade out.

---

## 7. Method Summary

| Method                                                      | Purpose                                                                                                                                           |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `InitializePool(string, ParticleSystem, int)`               | Pre‐instantiate and enqueue `initialSize` disabled clones of the provided `ParticleSystem` prefab under a pool key.                               |
| `PlayEffect(string, Vector3, Quaternion, Transform, float)` | Dequeue (or instantiate) a `ParticleSystem`, position/rotate it, activate and `Play()`. Optionally schedule auto‐return after `duration` seconds. |
| `ReturnToPool(ParticleSystem, string)`                      | Immediately `Stop()`, deactivate, detach, and enqueue a `ParticleSystem` back into its pool.                                                      |
| `ApplyPreset(ParticleSystem, string)`                       | Copied a stored `ParticlePreset`’s properties (color, size, speed, emission, shape) into a `ParticleSystem`.                                      |
| `ModifyParticles(ParticleSystem, ParticleModifier)`         | Change select runtime properties (start color/size/speed/emission) on a `ParticleSystem`.                                                         |
| `CleanupInactiveParticles()`                                | Iterates through `_activeParticleSystems`, and for each one that `IsAlive() == false`, returns it to the correct pool.                            |
| **Helper classes**                                          |                                                                                                                                                   |
| `ParticlePreset`                                            | Holds a set of properties (color, size, speed, maxParticles, emissionRate, shapeType) to apply as a preset.                                       |
| `ParticleModifier`                                          | Nullable properties (`Color?`, `float?`) for on‐the‐fly modifications to an existing `ParticleSystem`.                                            |

---

## 8. Best Practices & Tips

1. **Prewarm Pools Wisely**

   * If you know roughly how many simultaneous particles you’ll need—e.g., a maximum of 10 simultaneous “Explosion” effects—initialize your pool with `initialSize` = 10 or 15 to avoid runtime instantiation spikes.
2. **Always Call CleanupInactiveParticles**

   * To avoid “leaking” active particles (i.e. they never get returned), place `ParticleHelper.CleanupInactiveParticles()` in a MonoBehaviour’s `Update()` or a `Coroutine` that runs every 0.5–1 second.
3. **Automatic Return vs. Cleanup**

   * `duration > 0` lets you auto‐return after a fixed time, but that assumes you know how long the particle effect lasts. For variable‐length effects, rely on `IsAlive()`.
   * If you call `PlayEffect` without a `duration`, the system never auto‐returns it—you must either call `ReturnToPool(...)` yourself or depend on `CleanupInactiveParticles()`.
4. **Presets vs. Manual Tweaks**

   * Use `ApplyPreset` to apply a complete configuration. If you only need to tweak 1–2 properties, use `ModifyParticles(...)`.
5. **Parenting Considerations**

   * If you pass a non‐null `parent` to `PlayEffect`, the particle instance becomes a child of that transform. When you call `ReturnToPool(...)`, it automatically does `ps.transform.SetParent(null)`. If you need it re‐parented later, do so after you call `PlayEffect`.
6. **Threading Note**

   * `ReturnToPoolAfterDuration` uses `Task.Delay(...)`, which runs on a thread‐pool thread. Because it’s `async void`, it will return to the main thread automatically when calling `ReturnToPool(...)`. Avoid heavy CPU work inside this helper method.

---

With these explanations, code samples, and best practices, you should have everything needed to:

* **Initialize** your particle pools at startup
* **Play** effects at runtime without incurring `Instantiate`/`Destroy` overhead
* **Apply** standardized presets (fire, smoke, sparks) or tweak properties on the fly
* **Clean up** finished particle systems transparently

Feel free to extend or customize any part of `ParticleHelper`—for instance, add additional preset fields (gravity, lifetime, color over time curves) or build a more advanced scheduling system. But as is, this helper provides a flexible, performance‐minded foundation for most Unity particle workflows.
