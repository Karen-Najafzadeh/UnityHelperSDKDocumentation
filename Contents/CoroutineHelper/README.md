Below is a comprehensive breakdown of **CoroutineHelper**—a static class that wraps Unity’s coroutine system and .NET Tasks to provide advanced lifecycle management, parallel and sequential execution, error handling, and cancellation support. Each region of the code is explained in detail, followed by realistic usage examples you can plug directly into your Unity project.

---

## Table of Contents

1. [Overall Architecture](#1-overall-architecture)
2. [Initialization](#2-initialization)
3. [Coroutine Management](#3-coroutine-management)

   * 3.1. `StartManagedCoroutine`
   * 3.2. `StopCoroutineWithId`
   * 3.3. `StopAllCoroutines`
4. [Task Integration](#4-task-integration)

   * 4.1. `ToCoroutine(Task)`
   * 4.2. `ToCoroutine<T>(Task<T>, Action<T>)`
5. [Sequence Management](#5-sequence-management)

   * 5.1. `Sequence(params IEnumerator[])`
   * 5.2. `Parallel(params IEnumerator[])`
6. [Error Handling](#6-error-handling)

   * 6.1. `WrapWithErrorHandler`
7. [CoroutineRunner (MonoBehaviour)](#7-coroutinerunner-monobehaviour)
8. [Real‐World Usage Examples](#8-realworld-usage-examples)

   * 8.1. Managing a Repeating Coroutine with Cancellation
   * 8.2. Converting a `Task` to a Coroutine for Asynchronous File I/O
   * 8.3. Running Game Logic in Sequence and in Parallel
   * 8.4. Handling Errors Gracefully in Coroutines
   * 8.5. Stopping Everything on Scene Unload
9. [Best Practices & Tips](#9-best-practices--tips)

---

## 1. Overall Architecture

```csharp
public static class CoroutineHelper
{
    private static readonly Dictionary<string, IEnumerator> _activeCoroutines 
        = new Dictionary<string, IEnumerator>();

    private static readonly Dictionary<string, CancellationTokenSource> _cancellationSources 
        = new Dictionary<string, CancellationTokenSource>();

    private static CoroutineRunner _runner;
    // … methods follow …
}
```

* **`_activeCoroutines`**

  * Maps a user‐provided `string id` → the wrapped `IEnumerator` that is currently running.
  * Allows you to stop a coroutine by referencing its unique ID.

* **`_cancellationSources`**

  * Maps the same `string id` → a `CancellationTokenSource`.
  * Although this code does not directly use the tokens inside the wrapped coroutine, storing the CTS lets you cancel any awaiting tasks or custom logic that checks `token.IsCancellationRequested`.

* **`_runner`**

  * A singleton instance of the internal `CoroutineRunner` MonoBehaviour.
  * Responsible for actually calling `StartCoroutine` and `StopCoroutine` on the `IEnumerator`s provided.

After these fields, the static class is organized into regions:

1. **Initialization**: Ensures there is a `CoroutineRunner` GameObject in the scene.
2. **Coroutine Management**: Methods to start, stop, and globally stop coroutines by ID.
3. **Task Integration**: Utility methods to wrap a .NET `Task` or `Task<T>` as a Unity `IEnumerator`.
4. **Sequence Management**: Helpers to run multiple coroutines in series or in parallel.
5. **Error Handling**: A private helper that wraps any user‐provided `IEnumerator` so exceptions are caught and passed to a callback.
6. **`CoroutineRunner` MonoBehaviour**: An internal component that ensures coroutines continue across scene loads and allows cleanup on destroy.

---

## 2. Initialization

### 2.1 Method: `private static void EnsureInitialized()`

```csharp
private static void EnsureInitialized()
{
    if (_runner == null)
    {
        var go = new GameObject("CoroutineRunner");
        UnityEngine.Object.DontDestroyOnLoad(go);
        _runner = go.AddComponent<CoroutineRunner>();
    }
}
```

* **Purpose**: Guarantee that there is a `CoroutineRunner` in the scene to host all managed coroutines.

* **Behavior**:

  1. If `_runner` is already non‐null, do nothing (already initialized).
  2. Otherwise, create a new `GameObject` named `"CoroutineRunner"`, call `DontDestroyOnLoad` so it persists across scene changes, and add the `CoroutineRunner` component to it.
  3. Cache that new component in `_runner`.

* **When It’s Called**:

  * Automatically invoked at the top of any public method that starts or stops coroutines (`StartManagedCoroutine`, `StopCoroutineWithId`, `StopAllCoroutines`). You do not need to call it explicitly.

**Real‐World Example**:

You never need to manually create a `CoroutineRunner`—`CoroutineHelper.StartManagedCoroutine("MyId", SomeCoroutine())` will call `EnsureInitialized()` for you. If you examine the scene at runtime, you’ll see a persistent `GameObject` named `"CoroutineRunner"` carrying a `CoroutineRunner` component.

---

## 3. Coroutine Management

### 3.1 Method: `public static void StartManagedCoroutine(string id, IEnumerator routine, Action<Exception> onError = null)`

```csharp
/// <summary>
/// Start a coroutine with error handling and cancellation support
/// </summary>
public static void StartManagedCoroutine(
    string id,
    IEnumerator routine,
    Action<Exception> onError = null)
{
    EnsureInitialized();

    if (_activeCoroutines.ContainsKey(id))
        StopCoroutineWithId(id);

    var cts = new CancellationTokenSource();
    _cancellationSources[id] = cts;
    
    var wrappedRoutine = WrapWithErrorHandler(routine, onError);
    _activeCoroutines[id] = wrappedRoutine;
    _runner.StartCoroutine(wrappedRoutine);
}
```

**Purpose**:
Start a Unity‐style coroutine identified by a unique string `id`, wrapping it in error handling and assigning a `CancellationTokenSource` so you can cancel supporting tasks if needed.

**Parameters**:

* `string id`

  * A unique key you choose to identify this coroutine—for example, `"SpawnEnemies"`, `"AutoSave"`, or `"UIFadeSequence"`.
  * If you call `StartManagedCoroutine` again with the same `id`, the previous coroutine is stopped first to prevent duplicates.

* `IEnumerator routine`

  * The actual coroutine to run. Could be any `IEnumerator` method (e.g., `MyCoroutine()`).

* `Action<Exception> onError = null`

  * An optional callback invoked if an exception is thrown anywhere inside the coroutine.
  * If `onError` is null, exceptions will be silently swallowed by `WrapWithErrorHandler` (but you could log them inside your own handler).

**How It Works**:

1. **`EnsureInitialized()`**

   * Creates the `CoroutineRunner` GameObject if not already present.

2. **Stop Existing Instance**

   * If `_activeCoroutines` already contains `id`, calls `StopCoroutineWithId(id)` to halt and cleanup the old instance before starting a new one.

3. **Create `CancellationTokenSource`**

   * Instantiate a new `CancellationTokenSource`, store it in `_cancellationSources[id]`.
   * Your coroutine can inspect `cts.Token.IsCancellationRequested` at any yield point to terminate early if you implement that, though the default `WrapWithErrorHandler` does not check it.

4. **Wrap Routine with Error Handler**

   * Calls the private `WrapWithErrorHandler(routine, onError)` which returns a new `IEnumerator` that internally runs your routine step by step, catches exceptions, and finally invokes `onError`.
   * Storing this wrapped version (rather than the raw `routine`) ensures that exceptions are captured.

5. **Track and Start**

   * Add `wrappedRoutine` to `_activeCoroutines[id]` and call `_runner.StartCoroutine(wrappedRoutine)`.
   * The coroutine begins execution on the next frame.

**Example Usage**:

```csharp
public class EnemySpawner : MonoBehaviour
{
    private void Start()
    {
        // Begin an endless spawn loop, identified by "SpawnLoop"
        CoroutineHelper.StartManagedCoroutine(
            "SpawnLoop",
            SpawnLoopCoroutine(),
            onError: ex => Debug.LogError($"SpawnLoop errored: {ex}")
        );
    }

    private IEnumerator SpawnLoopCoroutine()
    {
        while (true)
        {
            // Spawn an enemy every 5 seconds
            SpawnEnemyAtRandomLocation();
            yield return new WaitForSeconds(5f);
        }
    }

    private void OnDisable()
    {
        // Stop the spawn loop if this object is disabled or destroyed
        CoroutineHelper.StopCoroutineWithId("SpawnLoop");
    }

    private void SpawnEnemyAtRandomLocation()
    {
        // Implementation omitted
    }
}
```

* The `"SpawnLoop"` coroutine runs indefinitely, spawning enemies every 5 seconds.
* If an exception occurs inside `SpawnLoopCoroutine()`, it is caught and passed to `onError`.
* When the object is disabled, `StopCoroutineWithId("SpawnLoop")` halts it and cleans up.

---

### 3.2 Method: `public static void StopCoroutineWithId(string id)`

```csharp
/// <summary>
/// Stop a coroutine by ID and clean up resources
/// </summary>
public static void StopCoroutineWithId(string id)
{
    if (_activeCoroutines.TryGetValue(id, out var routine))
    {
        _runner.StopCoroutine(routine);
        _activeCoroutines.Remove(id);
    }

    if (_cancellationSources.TryGetValue(id, out var cts))
    {
        cts.Cancel();
        cts.Dispose();
        _cancellationSources.Remove(id);
    }
}
```

**Purpose**:
Stop a previously started managed coroutine (identified by `id`) and cancel any associated `CancellationTokenSource`.

**Behavior**:

1. **Stop the Coroutine**

   * If `_activeCoroutines` contains `id`, call `_runner.StopCoroutine(routine)` to halt it instantly.
   * Remove that entry from `_activeCoroutines`.

2. **Clean Up the `CancellationTokenSource`**

   * If `_cancellationSources` contains `id`, call `cts.Cancel()` and `cts.Dispose()` to signal cancellation and free resources.
   * Remove the CTS entry from `_cancellationSources`.

**When to Use**:

* Whenever you want to halt a long‐running or repeating coroutine early—e.g., when a level ends, the player dies, or a UI closes.

**Example Usage**:

```csharp
public class AutoSaver : MonoBehaviour
{
    private void OnEnable()
    {
        // Start an autosave coroutine
        CoroutineHelper.StartManagedCoroutine(
            "AutoSave",
            AutoSaveRoutine(),
            ex => Debug.LogError($"AutoSave failed: {ex}")
        );
    }

    private void OnDisable()
    {
        // Stop autosaving when this object is disabled or scene unloads
        CoroutineHelper.StopCoroutineWithId("AutoSave");
    }

    private IEnumerator AutoSaveRoutine()
    {
        while (true)
        {
            yield return new WaitForSeconds(60f); // Every 60 seconds
            SaveGame();
        }
    }

    private void SaveGame()
    {
        // Implementation omitted
    }
}
```

* This ensures that if the player quits to the main menu (disabling this component), the autosave loop halts.

---

### 3.3 Method: `public static void StopAllCoroutines()`

```csharp
/// <summary>
/// Stop all active coroutines and clean up
/// </summary>
public static void StopAllCoroutines()
{
    foreach (var id in _activeCoroutines.Keys.ToList())
        StopCoroutineWithId(id);

    _activeCoroutines.Clear();
    foreach (var cts in _cancellationSources.Values)
    {
        cts.Cancel();
        cts.Dispose();
    }
    _cancellationSources.Clear();
}
```

**Purpose**:
Halt every managed coroutine currently running (across all IDs), and cancel/dispose all associated cancellation tokens.

**Behavior**:

1. **Stop Each Active Coroutine**

   * Iterates over a copy of `_activeCoroutines.Keys` (using `ToList()` to avoid modifying the dictionary while iterating).
   * Calls `StopCoroutineWithId(id)` for each key.

2. **Clear Dictionaries**

   * After halting all coroutines, calls `_activeCoroutines.Clear()`.
   * Then iterates over every `CancellationTokenSource` in `_cancellationSources.Values`, calling `Cancel()` and `Dispose()` on each, finishing with `_cancellationSources.Clear()`.

**When to Use**:

* Useful when unloading a scene or shutting down a system and you want to ensure there are no stray coroutines continuing to run in the background.

**Example Usage**:

```csharp
public class LevelManager : MonoBehaviour
{
    private void OnDestroy()
    {
        // When the level manager is destroyed (e.g., on level load/unload),
        // stop all coroutines that may have been started via CoroutineHelper
        CoroutineHelper.StopAllCoroutines();
    }
}
```

* By putting `StopAllCoroutines()` in `OnDestroy()`, you guarantee no managed coroutines linger after the object or scene is torn down.

---

## 4. Task Integration

CoroutineHelper provides two overloads of `ToCoroutine` that let you await a `.NET Task` inside Unity’s coroutine system, bridging async/await tasks with Unity’s `IEnumerator` semantics.

### 4.1 Method: `public static IEnumerator ToCoroutine(Task task)`

```csharp
/// <summary>
/// Convert a Task to a Coroutine
/// </summary>
public static IEnumerator ToCoroutine(Task task)
{
    while (!task.IsCompleted)
        yield return null;

    if (task.IsFaulted)
        throw task.Exception;
}
```

**Purpose**:
Wrap a non‐generic `Task` (e.g., `Task` returned from an async file I/O, web request, etc.) so that you can `yield return` it in a coroutine. The calling code will wait until the task is finished; if the task faults, the exception is re‐thrown inside the coroutine.

**Behavior**:

1. **Wait for Completion**

   * While `task.IsCompleted == false`, yield `null` each frame. This effectively pauses the coroutine until the task finishes (either successfully, by fault, or by cancellation).

2. **Error Propagation**

   * If `task.IsFaulted == true`, rethrow `task.Exception` so that it can be caught by `WrapWithErrorHandler` (if used) or bubble up otherwise.

**Example Usage**:

```csharp
public class FileLoader : MonoBehaviour
{
    private void Start()
    {
        CoroutineHelper.StartManagedCoroutine(
            "LoadConfig",
            LoadConfigCoroutine(),
            ex => Debug.LogError($"Config load failed: {ex}")
        );
    }

    private IEnumerator LoadConfigCoroutine()
    {
        // Let’s say ReadConfigAsync returns Task<string>
        Task<string> readTask = ConfigService.ReadConfigAsync("settings.json");

        // Wait for the task using ToCoroutine
        yield return CoroutineHelper.ToCoroutine(readTask);

        // If no exception, the task is completed
        string configJson = readTask.Result;
        ProcessConfig(configJson);
    }

    private void ProcessConfig(string json)
    {
        // Implementation omitted
    }
}
```

* By calling `yield return CoroutineHelper.ToCoroutine(readTask)`, the coroutine will pause until `ReadConfigAsync` finishes. If that task throws, `LoadConfigCoroutine` will rethrow the exception, and the `onError` provided to `StartManagedCoroutine` will be invoked.

---

### 4.2 Method: `public static IEnumerator ToCoroutine<T>(Task<T> task, Action<T> onComplete)`

```csharp
/// <summary>
/// Convert a Task<T> to a Coroutine with result
/// </summary>
public static IEnumerator ToCoroutine<T>(Task<T> task, Action<T> onComplete)
{
    yield return ToCoroutine(task);
    
    if (!task.IsFaulted && !task.IsCanceled)
        onComplete?.Invoke(task.Result);
}
```

**Purpose**:
Wrap a `Task<T>` so that you can not only wait for its completion but also receive its result in a callback. The coroutine yields until the task completes; if there was no fault or cancellation, `onComplete(task.Result)` is invoked.

**Behavior**:

1. **Wait via the First Overload**

   * `yield return ToCoroutine(task)` waits for the task to complete and rethrows if it faults.

2. **Invoke Callback with Result**

   * If `!task.IsFaulted && !task.IsCanceled`, call `onComplete(task.Result)`.
   * This allows you to continue with the returned value once the task is done.

**Example Usage**:

```csharp
public class WebRequester : MonoBehaviour
{
    private void Start()
    {
        CoroutineHelper.StartManagedCoroutine(
            "FetchData",
            FetchDataCoroutine(),
            ex => Debug.LogError($"FetchData failed: {ex}")
        );
    }

    private IEnumerator FetchDataCoroutine()
    {
        // Suppose WebService.GetDataAsync returns Task<MyData>
        Task<MyData> dataTask = WebService.GetDataAsync("https://api.example.com/data");

        // Wait and then receive the result via callback
        yield return CoroutineHelper.ToCoroutine<MyData>(
            dataTask,
            resultData => HandleReceivedData(resultData)
        );
    }

    private void HandleReceivedData(MyData data)
    {
        // Process the received data
    }
}
```

* This pattern avoids `task.Wait()` (which blocks the main thread) and ties async network calls to your coroutine logic.

---

## 5. Sequence Management

Sometimes you need to run multiple coroutines either one after the other (sequence) or all at once (parallel). CoroutineHelper offers two simple utilities:

### 5.1 Method: `public static IEnumerator Sequence(params IEnumerator[] coroutines)`

```csharp
/// <summary>
/// Run multiple coroutines in sequence
/// </summary>
public static IEnumerator Sequence(params IEnumerator[] coroutines)
{
    foreach (var routine in coroutines)
        yield return _runner.StartCoroutine(routine);
}
```

**Purpose**:
Yield‐return each of the provided coroutines one after another. The next coroutine only starts after the previous one finishes.

**Behavior**:

1. **Loop Through Each `IEnumerator` in `coroutines[]`**
2. **For Each**: call `_runner.StartCoroutine(routine)` and then `yield return` that `Coroutine` handle.

   * This means Unity will wait until `routine` finishes before moving on.

**Example Usage**:

```csharp
public class LevelIntro : MonoBehaviour
{
    private void Start()
    {
        CoroutineHelper.StartManagedCoroutine(
            "IntroSequence",
            CameraHelper.Sequence(
                FadeOutCurtains(),
                PlayIntroDialogue(),
                FadeInHUD()
            ),
            ex => Debug.LogError($"IntroSequence error: {ex}")
        );
    }

    private IEnumerator FadeOutCurtains()
    {
        // Implementation omitted, fade out UI curtains over 2s
        yield return new WaitForSeconds(2f);
    }

    private IEnumerator PlayIntroDialogue()
    {
        // Implementation omitted, play audio and wait until done
        yield return new WaitForSeconds(5f);
    }

    private IEnumerator FadeInHUD()
    {
        // Implementation omitted, fade in HUD over 1s
        yield return new WaitForSeconds(1f);
    }
}
```

* This ensures the curtains fade out, then the dialogue plays, then the HUD fades in—in strict order.

---

### 5.2 Method: `public static IEnumerator Parallel(params IEnumerator[] coroutines)`

```csharp
/// <summary>
/// Run multiple coroutines in parallel
/// </summary>
public static IEnumerator Parallel(params IEnumerator[] coroutines)
{
    var routines = new List<Coroutine>();
    foreach (var routine in coroutines)
        routines.Add(_runner.StartCoroutine(routine));

    foreach (var routine in routines)
        yield return routine;
}
```

**Purpose**:
Start all provided coroutines at once (in parallel), then yield until each one has finished. The calling code waits until **all** have completed before moving on.

**Behavior**:

1. **Start Each `IEnumerator`**

   * Calls `_runner.StartCoroutine(routine)` for each `routine` and collects the returned `Coroutine` handles in a `List<Coroutine>`.

2. **Wait for All**

   * Loops through that list and `yield return routine` on each. Since Unity’s `Coroutine` handle yields until that coroutine is done, the code effectively waits for all the tasks to finish (in the order they were added, though they run concurrently).

**Example Usage**:

```csharp
public class SimultaneousActions : MonoBehaviour
{
    private void Start()
    {
        CoroutineHelper.StartManagedCoroutine(
            "SimulActions",
            CoroutineHelper.Parallel(
                MoveToPosition(new Vector3(10, 0, 0)),
                FadeOutMusic(1f),
                LoadLevelDataAsync()
            ),
            ex => Debug.LogError($"SimulActions error: {ex}")
        );
    }

    private IEnumerator MoveToPosition(Vector3 target)
    {
        while (Vector3.Distance(transform.position, target) > 0.1f)
        {
            transform.position = Vector3.MoveTowards(transform.position, target, 5f * Time.deltaTime);
            yield return null;
        }
    }

    private IEnumerator FadeOutMusic(float duration)
    {
        AudioSource music = FindObjectOfType<AudioSource>();
        float startVol = music.volume;
        float elapsed = 0f;
        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            music.volume = Mathf.Lerp(startVol, 0f, elapsed / duration);
            yield return null;
        }
        music.Stop();
    }

    private IEnumerator LoadLevelDataAsync()
    {
        Task levelTask = LevelService.LoadLevelDataAsync("Level1");
        yield return CoroutineHelper.ToCoroutine(levelTask);
        // Once done, proceed (no callback needed here)
    }
}
```

* `MoveToPosition`, `FadeOutMusic`, and `LoadLevelDataAsync` all run at the same time.
* The main “SimulActions” coroutine continues only after all three have completed.

---

## 6. Error Handling

### 6.1 Method: `private static IEnumerator WrapWithErrorHandler(IEnumerator routine, Action<Exception> onError)`

```csharp
private static IEnumerator WrapWithErrorHandler(
    IEnumerator routine,
    Action<Exception> onError)
{
    bool hasError = false;
    Exception error = null;
    bool isComplete = false;

    while (!isComplete)
    {
        object current = null;
        try
        {
            if (!(isComplete = !routine.MoveNext()))
                current = routine.Current;
        }
        catch (Exception ex)
        {
            hasError = true;
            error = ex;
            isComplete = true;
        }

        if (!isComplete)
            yield return current;
    }
    
    if (hasError && onError != null)
    {
        try
        {
            onError(error);
        }
        catch (Exception ex)
        {
            Debug.LogException(ex);
        }
    }
}
```

**Purpose**:
Wrap any user‐provided coroutine `routine` so that exceptions thrown inside it are caught and delivered to `onError`, rather than crashing Unity or leaving silent failures.

**How It Works**:

1. **Local Flags**

   * `hasError` tracks whether an exception occurred.
   * `error` stores the caught `Exception`.
   * `isComplete` indicates when the inner `routine` has fully finished (naturally or due to an exception).

2. **Advancing the Routine**

   * In each loop iteration, attempt `routine.MoveNext()`.
   * If `MoveNext()` throws, catch it, set `hasError = true; error = ex; isComplete = true`. That stops further iteration.
   * If `MoveNext()` returns `false`, it means the routine has ended normally, so `isComplete = true`.

3. **Yielding**

   * If `isComplete` is still `false`, retrieve `routine.Current` into `current` and `yield return current`. This causes the wrapper to yield whatever the inner routine yielded (e.g., `WaitForSeconds(1)`, `null`, `WaitUntil(...)`, etc.).

4. **After Completion**

   * Once the loop exits, if `hasError == true && onError != null`, attempt to call `onError(error)`.
   * If `onError` itself throws, catch that in a `try/catch` and log via `Debug.LogException`.

**Why It’s Important**:

* In Unity, if an exception is thrown inside a coroutine and not caught, Unity logs it but continues running—some subsequent `yield return`s may be skipped or the entire coroutine may silently stop. Wrapping with `WrapWithErrorHandler` ensures you control how exceptions are handled and can provide user feedback or cleanup.

**Example Scenario**:

```csharp
private IEnumerator FragileRoutine()
{
    // Do some work
    yield return new WaitForSeconds(1f);
    // Suppose this line throws NullReferenceException
    GameObject go = null;
    go.transform.position = Vector3.zero;
    yield return null;
}

private void StartFragile()
{
    CoroutineHelper.StartManagedCoroutine(
        "FragileID",
        FragileRoutine(),
        onError: ex => Debug.LogError($"FragileRoutine encountered: {ex.Message}")
    );
}
```

* Without the wrapper, Unity would log a stack trace but you wouldn’t have a centralized place to react to the error.
* With the wrapper and `onError`, you can notify the player, revert state, or attempt recovery.

---

## 7. CoroutineRunner (MonoBehaviour)

```csharp
/// <summary>
/// MonoBehaviour that runs coroutines
/// </summary>
internal class CoroutineRunner : MonoBehaviour
{
    private void OnDestroy()
    {
        CoroutineHelper.StopAllCoroutines();
    }
}
```

* **Purpose**:

  * A minimal `MonoBehaviour` attached to a persistent `GameObject` named `"CoroutineRunner"`.
  * It exists solely to host coroutine execution (`StartCoroutine`/`StopCoroutine`) for all methods in `CoroutineHelper`.

* **`OnDestroy()`**:

  * When the `CoroutineRunner` is destroyed (which only happens if you manually destroy the `"CoroutineRunner"` GameObject), calls `CoroutineHelper.StopAllCoroutines()`.
  * This ensures no lingering managed coroutines continue after the runner goes away.

* **Why It’s Internal**:

  * It’s not meant to be used directly by game code. All public methods in `CoroutineHelper` call `EnsureInitialized()`, which creates this component if needed.

**Real‐World Note**:

* Because `CoroutineRunner` is marked with `DontDestroyOnLoad`, it stays alive across scene loads—so your managed coroutines continue even if you load a new scene.

---

## 8. Real‐World Usage Examples

Below are several practical scenarios demonstrating how to utilize **CoroutineHelper** in common game scenarios.

### 8.1 Managing a Repeating Coroutine with Cancellation

```csharp
public class EnemyAI : MonoBehaviour
{
    private void OnEnable()
    {
        // Every 2 seconds, check for a player and update path
        CoroutineHelper.StartManagedCoroutine(
            id: "PathfindingLoop",
            routine: PathfindingLoop(),
            onError: ex => Debug.LogError($"AI error: {ex.Message}")
        );
    }

    private void OnDisable()
    {
        // Halt the pathfinding loop when this AI is no longer active
        CoroutineHelper.StopCoroutineWithId("PathfindingLoop");
    }

    private IEnumerator PathfindingLoop()
    {
        while (true)
        {
            UpdatePathToPlayer();
            yield return new WaitForSeconds(2f);
        }
    }

    private void UpdatePathToPlayer()
    {
        // Implementation omitted
    }
}
```

* **What Happens**:

  * `PathfindingLoop` runs indefinitely, updating the enemy’s path every 2 seconds.
  * If `UpdatePathToPlayer()` throws an exception, it’s caught by `WrapWithErrorHandler` and printed via `onError`.
  * When the enemy is disabled or destroyed, `StopCoroutineWithId("PathfindingLoop")` automatically cancels and stops the coroutine.

---

### 8.2 Converting a Task to a Coroutine for Asynchronous File I/O

```csharp
public class SaveLoadManager : MonoBehaviour
{
    private void Start()
    {
        // Load save data at startup
        CoroutineHelper.StartManagedCoroutine(
            id: "LoadSaveData",
            routine: LoadSaveDataCoroutine(),
            onError: ex => Debug.LogError($"LoadSaveData error: {ex}")
        );
    }

    private IEnumerator LoadSaveDataCoroutine()
    {
        // Suppose ReadFromDiskAsync returns Task<SaveData>
        Task<SaveData> loadTask = SaveService.ReadFromDiskAsync("player.sav");

        // Wait until the task completes, using our helper
        yield return CoroutineHelper.ToCoroutine<SaveData>(
            loadTask,
            data => {
                // After loading, apply to game state
                ApplySaveData(data);
            }
        );
    }

    private void ApplySaveData(SaveData data)
    {
        // Implementation omitted
    }
}
```

* **What Happens**:

  * `ReadFromDiskAsync("player.sav")` runs on a background thread.
  * The coroutine yields each frame until the `Task<SaveData>` finishes.
  * Once done, `ApplySaveData(data)` is invoked with the result.
  * If any exception occurs in the `Task`, the coroutine rethrows it, caught by `onError`.

---

### 8.3 Running Game Logic in Sequence and in Parallel

```csharp
public class LevelController : MonoBehaviour
{
    private void Start()
    {
        CoroutineHelper.StartManagedCoroutine(
            id: "LevelIntroSequence",
            routine: IntroSequence(),
            onError: ex => Debug.LogError($"IntroSequence failed: {ex}")
        );
    }

    private IEnumerator IntroSequence()
    {
        // Phase 1 & 2 in parallel: Fade out UI and load first wave simultaneously
        yield return CoroutineHelper.Parallel(
            FadeOutUI(1f),
            SpawnFirstWave()
        );

        // Once those complete, wait 2 seconds, then fade in camera
        yield return new WaitForSeconds(2f);
        yield return FadeInCamera(1f);

        // Phase 3: Start main gameplay loop
        StartGameplayLoop();
    }

    private IEnumerator FadeOutUI(float duration)
    {
        // Dummy implementation
        float elapsed = 0f;
        CanvasGroup cg = FindObjectOfType<CanvasGroup>();
        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            cg.alpha = Mathf.Lerp(1f, 0f, elapsed / duration);
            yield return null;
        }
    }

    private IEnumerator SpawnFirstWave()
    {
        // Dummy implementation
        for (int i = 0; i < 5; i++)
        {
            SpawnEnemy();
            yield return new WaitForSeconds(0.5f);
        }
    }

    private IEnumerator FadeInCamera(float duration)
    {
        // Dummy implementation
        float elapsed = 0f;
        Camera cam = Camera.main;
        float startFOV = cam.fieldOfView;
        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            cam.fieldOfView = Mathf.Lerp(startFOV, 60f, elapsed / duration);
            yield return null;
        }
    }

    private void StartGameplayLoop()
    {
        // Implementation omitted
    }

    private void SpawnEnemy()
    {
        // Implementation omitted
    }
}
```

* **What Happens**:

  * The UI fade‐out and first wave spawning run simultaneously (Parallel).
  * Only after both finish does the coroutine wait an additional 2 seconds, then fade in the camera.
  * Finally, it calls `StartGameplayLoop()`.

---

### 8.4 Handling Errors Gracefully in Coroutines

```csharp
public class DataProcessor : MonoBehaviour
{
    private void Start()
    {
        CoroutineHelper.StartManagedCoroutine(
            "ProcessData",
            ProcessDataCoroutine(),
            ex => {
                Debug.LogError($"Data processing encountered an error: {ex}");
                ShowErrorUI(ex);
            }
        );
    }

    private IEnumerator ProcessDataCoroutine()
    {
        // Step 1: Validate
        if (!ValidateInputs())
            throw new InvalidOperationException("Invalid inputs for data processing.");

        yield return null;

        // Step 2: Heavy computation (simulate with WaitForSeconds)
        yield return new WaitForSeconds(2f);
        
        // Step 3: Simulate another possible error
        bool success = DoComputation();
        if (!success)
            throw new Exception("Computation failed in unknown manner.");

        // Step 4: Finish
        ShowSuccessUI();
    }

    private bool ValidateInputs()
    {
        // Dummy false condition to simulate an error
        return false;
    }

    private bool DoComputation()
    {
        return true;
    }

    private void ShowErrorUI(Exception ex)
    {
        // Display error popup
    }

    private void ShowSuccessUI()
    {
        // Display success popup
    }
}
```

* **What Happens**:

  * In `ProcessDataCoroutine`, `ValidateInputs()` returns `false`, so an `InvalidOperationException` is thrown.
  * `WrapWithErrorHandler` catches it and then calls the `onError` callback (`ShowErrorUI(ex)`), so the user sees a popup instead of the game silently failing.

---

### 8.5 Stopping Everything on Scene Unload

```csharp
public class SceneCleanup : MonoBehaviour
{
    private void OnDestroy()
    {
        // As soon as this scene’s root is destroyed, kill all managed coroutines
        CoroutineHelper.StopAllCoroutines();
    }
}
```

* **What Happens**:

  * When you unload the scene (e.g., `SceneManager.LoadScene("NextScene")`), `OnDestroy` of `SceneCleanup` triggers.
  * Every coroutine started via `CoroutineHelper` in that scene (or any persistent runner) is stopped immediately to avoid “orphaned” logic that might reference destroyed objects.

---

## 9. Best Practices & Tips

1. **Always Use Unique IDs**

   * Choose descriptive, unique string IDs for your coroutines (e.g., `"PlayerSpawn"`, `"AutoSave"`, `"FadeMusic"`) to avoid accidental overwriting or stopping the wrong coroutine.

2. **Avoid Long‐Lived Coroutines Without Clean‐Up**

   * If your coroutine loops indefinitely (e.g., `while(true)`), ensure you have a corresponding `StopCoroutineWithId(id)` call in `OnDisable` or `OnDestroy` so it doesn’t run after its owner is gone.

3. **Check Cancellation Tokens in Long Operations**

   * Although `WrapWithErrorHandler` does not automatically check `cts.Token`, you can modify your own coroutine to do so:

     ```csharp
     private IEnumerator HeavyWorkWithCancellation(string id)
     {
         var token = CoroutineHelper.GetCancellationToken(id); // (You could add this method)
         for (int i = 0; i < 100; i++)
         {
             if (token.IsCancellationRequested)
                 yield break;
             // ... do work …
             yield return null;
         }
     }
     ```
   * In the current code, you would need to manually retrieve the `CancellationTokenSource` from `_cancellationSources[id]` to check `IsCancellationRequested`.

4. **Don’t Forget to Provide `onError` Callbacks**

   * If you expect your coroutine might throw exceptions (e.g., from file I/O or network calls), always supply an `onError` so you can handle them gracefully rather than letting Unity silently swallow the exception.

5. **Use `Sequence` & `Parallel` to Simplify Complex Flows**

   * Avoid deeply nested `yield return` logic; use `Sequence(...)` for step‐by‐step flows and `Parallel(...)` when multiple tasks can run simultaneously.

6. **Be Mindful of Unity’s Main Thread**

   * All `IEnumerator` calls still execute on Unity’s main thread. If you wrap a `Task` that does heavy CPU work (e.g., image processing) directly, it should be on a background thread. Use `Task.Run(...)` for that, then wrap with `ToCoroutine` to wait for completion.

7. **Single Runner vs. Multiple Runners**

   * This code uses one global `CoroutineRunner` for everything. If you need coroutines tied to specific GameObjects (e.g., so they’re automatically canceled when that object is destroyed), use Unity’s built‐in `StartCoroutine` on that MonoBehaviour instead of `CoroutineHelper`.

8. **Avoid `StopAllCoroutines()` Unless Necessary**

   * Calling `StopAllCoroutines()` (whether on the runner or on your own MonoBehaviour) will abruptly halt everything. Use with caution—prefer stopping individual coroutines by ID where possible.

---

With these explanations and examples, you should be able to integrate **CoroutineHelper** into your Unity projects to achieve robust, cancellable, parallel, and sequential coroutine/task flows—complete with error handling and lifecycle management—without reinventing the wheel each time.
