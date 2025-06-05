Below is a complete breakdown of **DebugHelper**—a static utility class for Unity that delivers enhanced logging, performance timing, visual debugging primitives, memory‐profiling, and an FPS counter. You can sprinkle calls throughout your game code to diagnose issues, measure hotspots, visualize data in‐scene, and track memory/FPS in development builds or the Editor.

---

## Table of Contents

1. [Overview & Purpose](#overview--purpose)
2. [Internal Data & Fields](#internal-data--fields)
3. [Logging System](#logging-system)

   * 3.1. `Log(string message, string category = "General", LogLevel level = LogLevel.Info)`
   * 3.2. `SetCategoryLevel(string category, LogLevel level)`
   * 3.3. `ShouldLog(string category, LogLevel level)`
4. [Performance Monitoring](#performance-monitoring)

   * 4.1. `StartTimer(string id)`
   * 4.2. `StopTimer(string id)`
5. [Visual Debugging Primitives](#visual-debugging-primitives)

   * 5.1. `DrawPersistentSphere(Vector3 position, float radius, Color color)`
   * 5.2. `DrawPersistentBox(Vector3 center, Vector3 size, Color color)`
6. [Memory Tracking](#memory-tracking)

   * 6.1. `LogMemoryStats()`
7. [FPS Counter](#fps-counter)

   * 7.1. `UpdateFPS()`
   * 7.2. `GetAverageFPS()`
8. [LogLevel Enum](#loglevel-enum)
9. [Putting It All Together: Real‐World Usage Examples](#putting-it-all-together-real-world-usage-examples)

   * 9.1. Configuring & Using Category Levels
   * 9.2. Measuring Function Execution Time
   * 9.3. Drawing Persistent Debug Shapes in‐Scene
   * 9.4. Logging Memory Usage on Demand
   * 9.5. Displaying Average FPS in a Debug UI
10. [Best Practices & Tips](#best-practices--tips)
11. [Summary of Public API](#summary-of-public-api)

---

## 1. Overview & Purpose

```csharp
/// <summary>
/// A comprehensive debug helper for Unity that provides enhanced logging,
/// performance monitoring, and visual debugging tools.
///
/// Features:
/// - Conditional logging
/// - Performance monitoring
/// - Visual debugging
/// - Stack trace analysis
/// - Memory tracking
/// - FPS counter
/// </summary>
public static class DebugHelper { … }
```

**DebugHelper** centralizes common diagnostic tasks:

* **Logging** with categories and minimum log‐levels, so in development builds or the Editor you can filter out low‐priority messages.
* **Performance Monitoring** via simple `StartTimer`/`StopTimer` pairs (using `System.Diagnostics.Stopwatch`), logging elapsed milliseconds under a “Performance” category.
* **Visual Debugging** draws persistent lines (spheres, boxes) in the Scene View to highlight positions, bounds, etc., without disappearing after a single frame.
* **Memory Tracking** to snapshot Unity’s profiler metrics (allocated memory, reserved memory, Mono heap/used).
* **FPS Counting** to calculate a moving‐average framerate over the last second of gameplay.

All operational methods are tagged with `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]`, so they only compile into Editor or Development builds, not Shipping–this avoids runtime overhead in production.

---

## 2. Internal Data & Fields

```csharp
// Log settings
private static bool _enableLogging = true;
private static readonly Dictionary<string, LogLevel> _categoryLevels = new Dictionary<string, LogLevel>();
private static readonly StringBuilder _logBuilder = new StringBuilder();

// Performance monitoring
private static readonly Dictionary<string, Stopwatch> _stopwatches = new Dictionary<string, Stopwatch>();

// FPS tracking
private static readonly Queue<float> _fpsBuffer = new Queue<float>();
private static readonly int _fpsBufferSize = 60;
private static float _lastFpsUpdate;
private static readonly float _fpsUpdateInterval = 0.5f;
```

1. **Logging Fields**

   * `_enableLogging` (unused in current code but represents a global toggle).
   * `_categoryLevels`: maps a string “category” → minimum `LogLevel`. If a category is set to `Warning`, only `Warning` and `Error` messages under that category will print.
   * `_logBuilder`: a `StringBuilder` used by memory‐stats logging to build a multi‐line string efficiently.

2. **Performance Monitoring**

   * `_stopwatches`: maps a string `id` → a `Stopwatch`. When you call `StartTimer("MyTask")`, a new stopwatch is created (or restarted) for that key; `StopTimer("MyTask")` logs the elapsed time via `Log(..., category:"Performance", level:LogLevel.Debug)`.

3. **FPS Tracking**

   * `_fpsBuffer`: a queue of the last N instantaneous FPS samples (sampled every `_fpsUpdateInterval` seconds).
   * `_fpsBufferSize = 60`: stores up to 60 samples (roughly 30 seconds of history if sampling every 0.5 s, although by default this covers \~30 s of gameplay).
   * `_lastFpsUpdate`: tracks the time when the last FPS sample was taken.
   * `_fpsUpdateInterval = 0.5f`: sample FPS every 0.5 seconds.

---

## 3. Logging System

### 3.1 Method: `Log(string message, string category = "General", LogLevel level = LogLevel.Info)`

```csharp
[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
public static void Log(string message, string category = "General", LogLevel level = LogLevel.Info)
{
    if (!ShouldLog(category, level))
        return;

    var formattedMessage = $"[{category}] {message}";
    
    switch (level)
    {
        case LogLevel.Debug:
            Debug.Log(formattedMessage);
            break;
        case LogLevel.Info:
            Debug.Log(formattedMessage);
            break;
        case LogLevel.Warning:
            Debug.LogWarning(formattedMessage);
            break;
        case LogLevel.Error:
            Debug.LogError(formattedMessage);
            break;
    }
}
```

**Purpose**:
Print a message to the Unity Console with a `[Category]` prefix and route it to the appropriate `Debug.Log*` method based on `level`. Because of the `[Conditional]` attributes, calls to `Log` are compiled out entirely unless you’re in the Editor or a Development Build.

**Parameters**:

* `message` (`string`): The core text you want to log.
* `category` (`string`, default `"General"`): A grouping tag. Examples: `"AI"`, `"UI"`, `"Network"`.
* `level` (`LogLevel`, default `Info`): One of `Debug`, `Info`, `Warning`, or `Error`. Controls logging severity.

**Behavior**:

1. **Check `ShouldLog(category, level)`**

   * If the category is configured to only allow `Warning`+ and you call `Log(..., LogLevel.Info)`, the method returns immediately without printing.

2. **Format the Message**

   * Prefix with `"[Category] "` so it’s easy to filter by category in the Console.

3. **Select Unity’s Logging Method**

   * `LogLevel.Debug` and `LogLevel.Info` both call `Debug.Log(...)`.
   * `LogLevel.Warning` calls `Debug.LogWarning(...)`.
   * `LogLevel.Error` calls `Debug.LogError(...)`.

Because this is decorated with `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]`, any `DebugHelper.Log(...)` call in code will be omitted by the compiler unless one of those symbols is defined. In a Release build, no logging code remains.

---

### 3.2 Method: `SetCategoryLevel(string category, LogLevel level)`

```csharp
public static void SetCategoryLevel(string category, LogLevel level)
{
    _categoryLevels[category] = level;
}
```

**Purpose**:
Adjust the minimum log level for a given category. Calls to `Log(...)` will be suppressed if `level` is below the configured threshold.

**Example**:

```csharp
DebugHelper.SetCategoryLevel("AI", LogLevel.Warning);
```

* After this call, any `DebugHelper.Log("some AI detail", "AI", LogLevel.Info)` or `DebugLevel.Debug` in category `"AI"` will be skipped. Only `Warning` or `Error` in `"AI"` will appear.

---

### 3.3 Method: `ShouldLog(string category, LogLevel level)`

```csharp
private static bool ShouldLog(string category, LogLevel level)
{
    return !_categoryLevels.TryGetValue(category, out var minLevel) || level >= minLevel;
}
```

**Purpose**:
Check if a message of `level` under `category` should be printed. Returns `true` if:

* The category is not found in `_categoryLevels` (meaning no minimum was set, so everything passes).
* Or the supplied `level >= minLevel` configured for that category.

Otherwise, returns `false`, and `Log` will skip printing.

---

## 4. Performance Monitoring

These methods let you bracket a piece of code—call `StartTimer("MyKey")` before, then `StopTimer("MyKey")` after. The elapsed time (in ms) is logged under category `"Performance"` at `LogLevel.Debug`.

### 4.1 Method: `StartTimer(string id)`

```csharp
[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
public static void StartTimer(string id)
{
    if (!_stopwatches.TryGetValue(id, out var sw))
    {
        sw = new Stopwatch();
        _stopwatches[id] = sw;
    }
    
    sw.Restart();
}
```

**Purpose**:
Start or restart a stopwatch for the given string key `id`.

**Behavior**:

1. **Lookup** in `_stopwatches`.
2. If not found, create a new `Stopwatch` and store it.
3. Call `sw.Restart()`, which both resets `Elapsed` to zero and starts timing.

**Example**:

```csharp
DebugHelper.StartTimer("LoadAssets");
// … code that loads assets …
DebugHelper.StopTimer("LoadAssets");
```

---

### 4.2 Method: `StopTimer(string id)`

```csharp
[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
public static void StopTimer(string id)
{
    if (_stopwatches.TryGetValue(id, out var sw))
    {
        sw.Stop();
        Log($"{id}: {sw.ElapsedMilliseconds}ms", "Performance", LogLevel.Debug);
    }
}
```

**Purpose**:
Stop timing for `id` and log the elapsed milliseconds.

**Behavior**:

1. **Lookup** in `_stopwatches`.
2. If found, call `sw.Stop()`.
3. Use `DebugHelper.Log` to print a message like `"LoadAssets: 123ms"` under the `"Performance"` category at `Debug` level.

Because `Log` is conditional, this debug timing only appears in Editor/Development builds.

---

## 5. Visual Debugging Primitives

Sometimes you want to draw shapes in the Scene view that persist (instead of just one frame) to mark positions, bounds, or areas of interest. These methods use `Debug.DrawLine` with `duration = float.MaxValue`, ensuring they remain visible until you manually clear them or stop the editor.

### 5.1 Method: `DrawPersistentSphere(Vector3 position, float radius, Color color)`

```csharp
[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
public static void DrawPersistentSphere(Vector3 position, float radius, Color color)
{
    Debug.DrawLine(position + Vector3.up * radius, position - Vector3.up * radius, color, float.MaxValue);
    Debug.DrawLine(position + Vector3.right * radius, position - Vector3.right * radius, color, float.MaxValue);
    Debug.DrawLine(position + Vector3.forward * radius, position - Vector3.forward * radius, color, float.MaxValue);
}
```

**Purpose**:
Visually mark a spherical region’s center by drawing three orthogonal diameter lines: up/down, left/right, forward/backward.

**Parameters**:

* `position`: Center of the sphere.
* `radius`: Radius of the sphere—defines how far each axis‐aligned line extends.
* `color`: The color of the lines (e.g., `Color.red`).

**Behavior**:

1. **Up/Down Line**

   * Draw a line from `(position + Vector3.up * radius)` to `(position - Vector3.up * radius)`.

2. **Right/Left Line**

   * Draw a line from `(position + Vector3.right * radius)` to `(position - Vector3.right * radius)`.

3. **Forward/Backward Line**

   * Draw a line from `(position + Vector3.forward * radius)` to `(position - Vector3.forward * radius)`.

4. **Duration** = `float.MaxValue`, so the lines persist indefinitely (or until you close the scene or manually clear them).

**Example**:

```csharp
public class EnemyAI : MonoBehaviour
{
    private void OnDrawGizmos()
    {
        // Mark the detection radius of this enemy in green
        DebugHelper.DrawPersistentSphere(transform.position, 3f, Color.green);
    }
}
```

* Even though `DrawPersistentSphere` is called every frame, the lines are drawn once and remain until the editor refreshes or enters Play Mode.

---

### 5.2 Method: `DrawPersistentBox(Vector3 center, Vector3 size, Color color)`

```csharp
[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
public static void DrawPersistentBox(Vector3 center, Vector3 size, Color color)
{
    Vector3 halfSize = size * 0.5f;
    Debug.DrawLine(center + new Vector3(-halfSize.x, -halfSize.y, -halfSize.z), 
                   center + new Vector3( halfSize.x, -halfSize.y, -halfSize.z), color, float.MaxValue);
    Debug.DrawLine(center + new Vector3(-halfSize.x, -halfSize.y,  halfSize.z), 
                   center + new Vector3( halfSize.x, -halfSize.y,  halfSize.z), color, float.MaxValue);
    Debug.DrawLine(center + new Vector3(-halfSize.x,  halfSize.y, -halfSize.z), 
                   center + new Vector3( halfSize.x,  halfSize.y, -halfSize.z), color, float.MaxValue);
    Debug.DrawLine(center + new Vector3(-halfSize.x,  halfSize.y,  halfSize.z), 
                   center + new Vector3( halfSize.x,  halfSize.y,  halfSize.z), color, float.MaxValue);

    Debug.DrawLine(center + new Vector3(-halfSize.x, -halfSize.y, -halfSize.z), 
                   center + new Vector3(-halfSize.x,  halfSize.y, -halfSize.z), color, float.MaxValue);
    Debug.DrawLine(center + new Vector3( halfSize.x, -halfSize.y, -halfSize.z), 
                   center + new Vector3( halfSize.x,  halfSize.y, -halfSize.z), color, float.MaxValue);
    Debug.DrawLine(center + new Vector3(-halfSize.x, -halfSize.y,  halfSize.z), 
                   center + new Vector3(-halfSize.x,  halfSize.y,  halfSize.z), color, float.MaxValue);
    Debug.DrawLine(center + new Vector3( halfSize.x, -halfSize.y,  halfSize.z), 
                   center + new Vector3( halfSize.x,  halfSize.y,  halfSize.z), color, float.MaxValue);

    Debug.DrawLine(center + new Vector3(-halfSize.x, -halfSize.y, -halfSize.z), 
                   center + new Vector3(-halfSize.x, -halfSize.y,  halfSize.z), color, float.MaxValue);
    Debug.DrawLine(center + new Vector3( halfSize.x, -halfSize.y, -halfSize.z), 
                   center + new Vector3( halfSize.x, -halfSize.y,  halfSize.z), color, float.MaxValue);
    Debug.DrawLine(center + new Vector3(-halfSize.x,  halfSize.y, -halfSize.z), 
                   center + new Vector3(-halfSize.x,  halfSize.y,  halfSize.z), color, float.MaxValue);
    Debug.DrawLine(center + new Vector3( halfSize.x,  halfSize.y, -halfSize.z), 
                   center + new Vector3( halfSize.x,  halfSize.y,  halfSize.z), color, float.MaxValue);
}
```

**Purpose**:
Draw a wireframe box (cube) defined by `center` and `size` (width, height, depth) that persists in the Scene.

**Parameters**:

* `center`: Center of the cube.
* `size`: Full dimensions `(width, height, depth)`.
* `color`: Color of the edges.

**Behavior**:

1. **Compute `halfSize = size * 0.5f`**.

2. **Draw 12 Edges** via `Debug.DrawLine(...)`:

   * 4 edges for the bottom face (y = –halfSize.y)
   * 4 edges for the top face (y = +halfSize.y)
   * 4 vertical edges connecting top and bottom.

3. **Duration** always `float.MaxValue`, so they remain visible until manually cleared or scene closed.

**Example**:

```csharp
public class BoundingBoxVisualizer : MonoBehaviour
{
    public Vector3 boxCenter;
    public Vector3 boxSize;

    private void Update()
    {
        // Show a persistent wireframe around "boxCenter" with dimensions "boxSize"
        DebugHelper.DrawPersistentBox(boxCenter, boxSize, Color.yellow);
    }
}
```

* The wireframe appears in Editor’s Scene view (and optionally Game view if gizmos are enabled) and stays until you stop the editor.

---

## 6. Memory Tracking

### 6.1 Method: `LogMemoryStats()`

```csharp
[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
public static void LogMemoryStats()
{
    var stats = new StringBuilder();
    stats.AppendLine("Memory Statistics:");
    stats.AppendLine($"Total Allocated: {UnityEngine.Profiling.Profiler.GetTotalAllocatedMemoryLong() / 1024 / 1024}MB");
    stats.AppendLine($"Total Reserved: {UnityEngine.Profiling.Profiler.GetTotalReservedMemoryLong() / 1024 / 1024}MB");
    stats.AppendLine($"Mono Heap: {UnityEngine.Profiling.Profiler.GetMonoHeapSizeLong() / 1024 / 1024}MB");
    stats.AppendLine($"Mono Used: {UnityEngine.Profiling.Profiler.GetMonoUsedSizeLong() / 1024 / 1024}MB");
    Log(stats.ToString(), "Memory", LogLevel.Debug);
}
```

**Purpose**:
Query Unity’s built‐in profiler for four memory metrics (in bytes), convert them to MB, and log them under category `"Memory"` at `Debug` level. Only present in development or Editor builds.

**Metrics**:

1. **`GetTotalAllocatedMemoryLong()`**

   * The total memory Unity has allocated from the OS (in bytes).

2. **`GetTotalReservedMemoryLong()`**

   * The total memory Unity has reserved for internal use (includes fragmentation).

3. **`GetMonoHeapSizeLong()`**

   * Total size of the Mono GC heap.

4. **`GetMonoUsedSizeLong()`**

   * Portion of the Mono heap currently in use.

**Example**:

```csharp
public class MemoryDebugger : MonoBehaviour
{
    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.M))
        {
            DebugHelper.LogMemoryStats();
        }
    }
}
```

* When you press “M,” the Console prints something like:

  ```
  [Memory] Memory Statistics:
  Total Allocated: 120MB
  Total Reserved: 256MB
  Mono Heap: 60MB
  Mono Used: 45MB
  ```

---

## 7. FPS Counter

### 7.1 Method: `UpdateFPS()`

```csharp
[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
public static void UpdateFPS()
{
    if (Time.unscaledTime > _lastFpsUpdate + _fpsUpdateInterval)
    {
        _lastFpsUpdate = Time.unscaledTime;
        float fps = 1f / Time.unscaledDeltaTime;
        _fpsBuffer.Enqueue(fps);
        if (_fpsBuffer.Count > _fpsBufferSize)
            _fpsBuffer.Dequeue();
    }
}
```

**Purpose**:
Called once per frame—typically from a MonoBehaviour’s `Update()`—to sample instantaneous `1 / deltaTime` every `_fpsUpdateInterval` seconds (default 0.5 s). Stores samples in `_fpsBuffer` (max size 60), forming a rolling buffer of recent FPS values.

**Behavior**:

1. If `Time.unscaledTime > _lastFpsUpdate + 0.5f`:

   * Update `_lastFpsUpdate = Time.unscaledTime`.
   * Compute instantaneous `fps = 1f / Time.unscaledDeltaTime`.
   * Enqueue into `_fpsBuffer`.
   * If buffer exceeds 60 samples, dequeue the oldest.

**Integration**:

* In a MonoBehaviour:

  ```csharp
  public class FPSDisplay : MonoBehaviour
  {
      private void Update()
      {
          DebugHelper.UpdateFPS();
      }
  }
  ```

* Call `UpdateFPS()` every frame so that sampling occurs at 0.5‐second intervals.

---

### 7.2 Method: `GetAverageFPS()`

```csharp
public static float GetAverageFPS()
{
    if (_fpsBuffer.Count == 0)
        return 0f;
    float sum = 0f;
    foreach (var fps in _fpsBuffer)
        sum += fps;
    return sum / _fpsBuffer.Count;
}
```

**Purpose**:
Compute and return the average of all FPS samples currently stored in `_fpsBuffer`. If the buffer is empty (e.g., during the first few frames), returns `0f`.

**Example**:

```csharp
public class FPSDisplay : MonoBehaviour
{
    private void Update()
    {
        DebugHelper.UpdateFPS();
        float avgFps = DebugHelper.GetAverageFPS();
        Debug.Log($"Average FPS: {avgFps:F1}");
    }
}
```

* Logs something like `Average FPS: 58.7` every frame (or you could throttle it to log less frequently).

---

## 8. LogLevel Enum

```csharp
public enum LogLevel
{
    Debug,
    Info,
    Warning,
    Error
}
```

* **`Debug` (0)**: Lowest severity; for verbose, minute‐to‐minute details.
* **`Info` (1)**: General informational messages.
* **`Warning` (2)**: Indicates something suspicious that might warrant attention.
* **`Error` (3)**: Serious issues that require immediate debugging.

When you call `DebugHelper.Log("...", "CategoryName", LogLevel.Warning)`, the method routes to `Debug.LogWarning` in Unity. Moreover, `SetCategoryLevel("CategoryName", LogLevel.Warning)` would suppress any subsequent calls below `Warning`.

---

## 9. Putting It All Together: Real‐World Usage Examples

Below are scenarios illustrating how to integrate **DebugHelper** into your Unity project for improved debugging and performance insights.

---

### 9.1 Configuring & Using Category Levels

```csharp
public class GameManager : MonoBehaviour
{
    private void Awake()
    {
        // Only log Info+ for General, Debug+ for AI
        DebugHelper.SetCategoryLevel("General", LogLevel.Info);
        DebugHelper.SetCategoryLevel("AI", LogLevel.Debug);

        DebugHelper.Log("Game started", "General", LogLevel.Info);
        DebugHelper.Log("This is a verbose AI check", "AI", LogLevel.Debug);
    }

    private void Update()
    {
        // Suppose this is inside an AI decision loop
        DebugHelper.Log("Checking pathfinding", "AI", LogLevel.Debug);
        // … AI code …
        DebugHelper.Log("Path found", "AI", LogLevel.Info);
    }
}
```

* **Result**:

  * In the Editor or Development build:

    * “Game started” (General/Info) prints, because `Info >= Info`.
    * “This is a verbose AI check” prints, because `Debug >= Debug`.
    * “Checking pathfinding” and “Path found” also print.
  * If you tried `DebugHelper.Log("MinorGeneralDetail", "General", LogLevel.Debug)`, it would be suppressed, since `"General"` was set to `LogLevel.Info` (so Debug < Info).

* **In a Release build**: All calls to `DebugHelper.Log` compile out, no overhead.

---

### 9.2 Measuring Function Execution Time

```csharp
public class LevelLoader : MonoBehaviour
{
    private async void Start()
    {
        DebugHelper.StartTimer("LoadLevel");
        await LoadLevelAsync("Level1");   // Some custom async method
        DebugHelper.StopTimer("LoadLevel");
    }

    private async Task LoadLevelAsync(string levelName)
    {
        // Simulate a load operation
        await Task.Delay(1500);
    }
}
```

* **What Happens**:

  * On the first line, `DebugHelper.StartTimer("LoadLevel")` begins a `Stopwatch`.
  * After `LoadLevelAsync` completes (simulated 1.5 s delay), `DebugHelper.StopTimer("LoadLevel")` stops the stopwatch and logs:

    ```
    [Performance] LoadLevel: 1503ms
    ```
  * Only in Editor/Development builds—omitted in production.

---

### 9.3 Drawing Persistent Debug Shapes In‐Scene

```csharp
public class SpawnAreaVisualizer : MonoBehaviour
{
    public Vector3 areaCenter = Vector3.zero;
    public Vector3 areaSize = new Vector3(10f, 0f, 10f);

    private void Update()
    {
        // Draw the spawn area as a yellow box that never disappears
        DebugHelper.DrawPersistentBox(areaCenter, areaSize, Color.yellow);

        // Additionally, mark the center with a small green sphere
        DebugHelper.DrawPersistentSphere(areaCenter, 0.2f, Color.green);
    }
}
```

* **What You See**:

  * In the Scene view of the Editor or Game view (if “Gizmos” is on), a stable yellow wireframe box, centered at `(0,0,0)` with dimensions `(10,0,10)`.
  * A small green line‐based crosshair (three lines) marking the exact center.
  * Because of `float.MaxValue` duration, these lines stay until you stop Play Mode. They won’t flicker each frame, since `Debug.DrawLine` with `float.MaxValue` draws them once.

---

### 9.4 Logging Memory Usage on Demand

```csharp
public class MemoryCheckUI : MonoBehaviour
{
    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.M))
        {
            DebugHelper.LogMemoryStats();
        }
    }
}
```

* **What Happens**:

  * Pressing “M” in the Editor or Development Build logs a multi‐line message:

    ```
    [Memory] Memory Statistics:
    Total Allocated: 125MB
    Total Reserved: 256MB
    Mono Heap: 60MB
    Mono Used: 45MB
    ```

  * In a Release build, the call to `LogMemoryStats()` is omitted.

---

### 9.5 Displaying Average FPS in a Debug UI

```csharp
using UnityEngine;
using UnityEngine.UI;

public class FPSDisplay : MonoBehaviour
{
    [SerializeField] private Text _fpsText;

    private void Update()
    {
        // 1. Sample a new FPS value
        DebugHelper.UpdateFPS();

        // 2. Read the average over the buffer
        float avgFps = DebugHelper.GetAverageFPS();

        // 3. Display in a UI Text (only if >0 sample exists)
        if (avgFps > 0f)
            _fpsText.text = $"FPS: {avgFps:F1}";
        else
            _fpsText.text = "FPS: ---";
    }
}
```

* **What You See**:

  * A UI Text (e.g., anchored top‐left) that updates every frame with the average FPS across the last 60 samples (sampled every 0.5 s).
  * If no samples yet (first 0.5 s), shows “FPS: ---”.

---

## 10. Best Practices & Tips

1. **Always Call Logging & Debug Methods in Editor/Dev Only**

   * Because each method is decorated with `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]`, any calls are stripped in Release builds—no performance penalty. Use these freely in development.

2. **Use Categories to Filter Output**

   * Before flooding your console, call `DebugHelper.SetCategoryLevel("MyCategory", LogLevel.Warning)` to suppress low‐level logs. Then adjust as needed.

3. **Time‐Consuming Operations**

   * Wrap expensive loops or loading routines with `StartTimer`/`StopTimer` to measure exactly how long they take—ideal for profiling in Editor without attaching the full Profiler.

4. **Persistent Visualizations vs. Transient**

   * If you want a shape to appear only for one frame, use `Debug.DrawLine(..., duration: 0f)` directly. The “Persistent” variants here use `float.MaxValue` so they remain. Remember to remove them (e.g., by stopping Play Mode) when you no longer need them.

5. **Clear Flickering Lines**

   * Because `DrawPersistentBox` or `Sphere` is typically called every frame in `Update()`, you won’t see flicker. If you call them only once, they stay visible forever. To remove them early, you’ll need to call `SceneView.RepaintAll()` in a custom Editor script or manually stop Play Mode.

6. **Memory Logging Granularity**

   * `GetTotalAllocatedMemoryLong()` and related calls measure Unity’s memory allocation in bytes. Dividing by `1024×1024` yields MB. Keep in mind that garbage collection spikes or asset loading can increase “Reserved” memory even if “Allocated” stays stable.

7. **FPS Sampling Interval**

   * By default, `UpdateFPS()` samples once every 0.5 seconds. You can change `_fpsUpdateInterval` if you want faster or slower updates (e.g., 0.2 s for more responsive but noisier readings).

8. **Thread Safety**

   * All of these methods run on the main Unity thread. Do not call from background threads. The `[Conditional]` attributes simply remove calls at compile time, not at runtime via preprocessor.

9. **Avoid Overuse in Production**

   * Even though logging is stripped out, calls to `UpdateFPS()` and memory/stopwatch stems incur a tiny overhead in Development builds. Wrap them judiciously around the parts you truly need to inspect.

---

## 11. Summary of Public API

| Method Signature                                                                                                                                                      | Description                                                                                                                           |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")] public static void Log(string message, string category = "General", LogLevel level = LogLevel.Info)` | Print `"[Category] message"` to Console at the appropriate `Debug.Log*` based on `level`, if `level >=` configured for that category. |
| `public static void SetCategoryLevel(string category, LogLevel level)`                                                                                                | Define the minimum `LogLevel` for messages under `category`. Messages below that level are suppressed.                                |
| `private static bool ShouldLog(string category, LogLevel level)`                                                                                                      | Returns `true` if `level >=` the minimum for `category`, or if `category` not configured.                                             |
| `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")] public static void StartTimer(string id)`                                                            | Begin or restart a stopwatch identified by `id` for performance measurement.                                                          |
| `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")] public static void StopTimer(string id)`                                                             | Stop the stopwatch for `id` and log `"{id}: {ms}ms"` under `"Performance"` at `LogLevel.Debug`.                                       |
| `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")] public static void DrawPersistentSphere(Vector3 position, float radius, Color color)`                | Draw three orthogonal diameter lines (X, Y, Z) to mark a sphere’s center and radius, with infinite duration in Scene/Editor view.     |
| `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")] public static void DrawPersistentBox(Vector3 center, Vector3 size, Color color)`                     | Draw the 12 edges of a box centered at `center` with dimensions `size`, with infinite duration in Scene/Editor view.                  |
| `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")] public static void LogMemoryStats()`                                                                 | Query Unity’s Profiler for allocated, reserved, Mono heap, and Mono used memory (in bytes), convert to MB, and log under `"Memory"`.  |
| `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")] public static void UpdateFPS()`                                                                      | Sample `1 / unscaledDeltaTime` every `_fpsUpdateInterval` seconds and enqueue into a buffer of size `_fpsBufferSize`.                 |
| `public static float GetAverageFPS()`                                                                                                                                 | Return the average of all FPS samples in `_fpsBuffer`.                                                                                |
| `public enum LogLevel { Debug, Info, Warning, Error }`                                                                                                                | Severity levels for logs; `Debug < Info < Warning < Error`.                                                                           |

Use **DebugHelper** to instrument your game’s code during development, pinpoint performance‐sapping sections, visualize spatial data in the Scene, and keep tabs on memory/FPS—all without polluting Release builds or incurring unnecessary runtime cost outside of development contexts.
