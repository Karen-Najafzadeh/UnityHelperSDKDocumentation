Below is a complete reference and usage guide for **TimeHelper**—a static utility class that centralizes common time‐related operations in Unity (timers, cooldowns, time scaling, scheduling, formatting, and measurement). It includes:

1. **Overview & Purpose**
2. **Method‐by‐Method Breakdown**
3. **Helper Classes & Data Structures**
4. **Usage Examples**
5. **Public Method Summary Table**

---

## 1. Overview & Purpose

`TimeHelper` is designed to simplify and unify a variety of time‐based tasks you often encounter in gameplay and UI programming. Its key features include:

* **Timer Management**: Create, pause, resume, and query countdown‐style timers by a unique ID.
* **Cooldowns**: Track cooldowns (e.g., ability usage) via simple “end timestamp” logic.
* **Time Scale Control**: Set Unity’s `Time.timeScale` smoothly (fading), pause and resume the game.
* **Scheduling**: Queue up actions (callbacks) to run after a specified delay (optionally unscaled).
* **Formatting**: Turn raw seconds or `TimeSpan` into human‐friendly strings (MM\:SS, HH\:MM\:SS, “2h, 5m” etc.).
* **Execution Measurement**: Measure how long a synchronous or asynchronous operation takes.

Most of these systems depend on an **Update** step. You should call `TimeHelper.Update()` (e.g., from a central MonoBehaviour) every frame so that timers, cooldowns, and scheduled actions progress properly.

Below, each region of functionality is explained in detail.

---

## 2. Method-by-Method Breakdown

### 2.1 Timer Management

These methods let you create named countdown timers, automatically invoke a callback when they finish, optionally loop them, and query or pause/resume them.

#### 2.1.1 `StartTimer`

```csharp
public static void StartTimer(
    string id,
    float duration,
    Action onComplete = null,
    bool loop = false
)
```

* **Purpose**: Create and start a new timer identified by `id`. If a timer with the same `id` already exists, it is stopped and replaced.
* **Parameters**:

  * `id` (string): Unique key for this timer (e.g., `"enemy_spawn"`, `"round_timer"`, etc.).
  * `duration` (float): How many seconds the timer counts down from.
  * `onComplete` (Action): Optional callback invoked once when the timer reaches zero.
  * `loop` (bool): If `true`, the timer automatically resets to `duration` and continues looping after firing its callback.
* **Details**:

  1. If `_timers` already contains the `id`, `StopTimer(id)` is called first.
  2. A new `Timer` object is created with:

     * `Duration = duration`
     * `TimeRemaining = duration`
     * `OnComplete = onComplete`
     * `IsLooping = loop`
     * `IsRunning = true`
  3. Stored into `_timers[id]`.

```csharp
// Example:
TimeHelper.StartTimer("powerUp_duration", 5f, () => Debug.Log("Power‐up expired!"), loop: false);
```

---

#### 2.1.2 `StopTimer`

```csharp
public static void StopTimer(string id)
```

* **Purpose**: Immediately stop and remove the timer identified by `id` (if it exists).
* **Parameters**:

  * `id` (string): Key of the timer to stop.

```csharp
// Example:
TimeHelper.StopTimer("powerUp_duration");
```

---

#### 2.1.3 `PauseTimer`

```csharp
public static void PauseTimer(string id)
```

* **Purpose**: Pause a running timer so that it no longer counts down.
* **Parameters**:

  * `id` (string): Key of the timer to pause.
* **Details**:

  * Sets `IsRunning = false` on the `Timer` object. The timer’s `TimeRemaining` remains frozen until you call `ResumeTimer`.

```csharp
// Example:
TimeHelper.PauseTimer("round_timer");
```

---

#### 2.1.4 `ResumeTimer`

```csharp
public static void ResumeTimer(string id)
```

* **Purpose**: Resume a previously paused timer (so it continues counting down from wherever it left off).
* **Parameters**:

  * `id` (string): Key of the timer to resume.
* **Details**:

  * Sets `IsRunning = true` on the `Timer`. If the timer’s `TimeRemaining` is already ≤ 0, it will fire immediately on the next update.

```csharp
// Example:
TimeHelper.ResumeTimer("round_timer");
```

---

#### 2.1.5 `GetRemainingTime`

```csharp
public static float GetRemainingTime(string id)
```

* **Purpose**: Query how many seconds are left on a timer (or 0 if it doesn’t exist).
* **Parameters**:

  * `id` (string): Key of the timer.
* **Returns**: Remaining time in seconds, or 0 if the timer is not found.

```csharp
float left = TimeHelper.GetRemainingTime("round_timer");
Debug.Log($"Time left: {left:F2} seconds");
```

---

#### 2.1.6 How Timers Advance: `UpdateTimers`

In order for timers to count down, you must call `TimeHelper.Update()` each frame (for example, from a central MonoBehaviour’s `Update()` method). Internally, `UpdateTimers()` runs each active `Timer`:

```csharp
// Pseudocode from UpdateTimers():
foreach (var kvp in _timers)
{
    var timer = kvp.Value;
    if (!timer.IsRunning) continue;
    timer.TimeRemaining -= Time.deltaTime;
    if (timer.TimeRemaining <= 0f)
    {
        timer.OnComplete?.Invoke();
        if (timer.IsLooping)
            timer.TimeRemaining = timer.Duration;
        else
            completedTimers.Add(kvp.Key);
    }
}
foreach (var id in completedTimers)
    _timers.Remove(id);
```

* Each frame, if `IsRunning` is true, subtract `Time.deltaTime` from `TimeRemaining`.
* When `TimeRemaining <= 0`, invoke `OnComplete()`. If `IsLooping`, reset; otherwise remove the timer.

---

### 2.2 Cooldown System

A lightweight system to mark an ability or action as “on cooldown.” Internally, each cooldown stores a future timestamp (Unix‐style) when it expires.

#### 2.2.1 `StartCooldown`

```csharp
public static void StartCooldown(string id, float duration)
```

* **Purpose**: Begin a cooldown identified by `id` that will expire in `duration` seconds from now.
* **Parameters**:

  * `id` (string): Unique key for this cooldown (e.g., `"fireball_cooldown"`).
  * `duration` (float): Number of seconds until “cooldown complete.”
* **Behavior**:

  * Internally sets `_cooldowns[id] = Time.time + duration`.

```csharp
TimeHelper.StartCooldown("dash", 2.5f); // “dash” ability on cooldown for 2.5s
```

---

#### 2.2.2 `IsCooldownComplete`

```csharp
public static bool IsCooldownComplete(string id)
```

* **Purpose**: Check if the cooldown with key `id` has finished—or never existed.
* **Parameters**:

  * `id` (string): Key to query.
* **Returns**:

  * `true` if either no cooldown exists for `id` or the stored expiration time ≤ `Time.time`.
  * If it is complete, the method also removes the entry from `_cooldowns`.

```csharp
if (TimeHelper.IsCooldownComplete("dash"))
{
    PerformDash();
}
```

---

#### 2.2.3 `GetCooldownRemaining`

```csharp
public static float GetCooldownRemaining(string id)
```

* **Purpose**: Return how many seconds are left until `id`'s cooldown is complete, or 0 if no such cooldown or already finished.
* **Parameters**:

  * `id` (string): Key to query.
* **Returns**: `max(0, endTime − Time.time)` if exists; otherwise 0.

```csharp
float left = TimeHelper.GetCooldownRemaining("dash");
Debug.Log($"Dash ready in {left:F1}s");
```

---

#### 2.2.4 How Cooldowns Advance: `UpdateCooldowns`

Within `TimeHelper.Update()`:

```csharp
foreach (var cooldown in _cooldowns)
    if (Time.time >= cooldown.Value)
        completedCooldowns.Add(cooldown.Key);
foreach (var key in completedCooldowns)
    _cooldowns.Remove(key);
```

* Each frame, any key whose expiration has passed is removed, making `IsCooldownComplete` continue returning `true`.

---

### 2.3 Time Scale Control

These methods allow you to adjust Unity’s `Time.timeScale`—either snapping instantly or smoothly interpolating over a duration. They also allow you to “pause” game time (scale = 0) and then resume to the previous scale.

#### 2.3.1 `SetTimeScale`

```csharp
public static async Task SetTimeScale(float targetScale, float duration = 0f)
```

* **Purpose**: Set `Time.timeScale` directly if `duration ≤ 0`, or smoothly transition from the current `Time.timeScale` to `targetScale` over `duration` real‐time seconds.
* **Parameters**:

  * `targetScale` (float): Desired timeScale (0 for pause, 1 for normal speed, 2 for double speed, etc.).
  * `duration` (float): How many seconds (unscaled time) it should take to interpolate. If `0` (default), instantly apply.
* **Behavior**:

  1. If `duration ≤ 0`:

     ```csharp
     Time.timeScale = targetScale;
     return;
     ```
  2. Otherwise, store `startScale = Time.timeScale`, then over multiple frames subtract `elapsed += Time.unscaledDeltaTime` and set

     ```csharp
     Time.timeScale = Mathf.Lerp(startScale, targetScale, elapsed/duration);
     ```

     until `elapsed ≥ duration`, then finalize `Time.timeScale = targetScale`.

```csharp
// Example: slow motion over 1 second, down to half speed
await TimeHelper.SetTimeScale(0.5f, 1f);
```

---

#### 2.3.2 `PauseGame`

```csharp
public static Action PauseGame(bool smooth = true)
```

* **Purpose**: Pause game updates by setting `Time.timeScale` to 0 (instant or over 0.2s fade). Returns an `Action` delegate which, when invoked, will resume to the previous scale.
* **Parameters**:

  * `smooth` (bool): If `true`, call `SetTimeScale(0, 0.2f)` to fade down over 0.2s. Otherwise, `Time.timeScale = 0` instantly.
* **Returns**: A callback `Action` which calls `ResumeGame(smooth)` when you want to unpause.

```csharp
// Example:
Action resume = TimeHelper.PauseGame(smooth: true);
// ... later, to unpause:
resume();
```

Internally, `PauseGame` stores `_previousTimeScale = Time.timeScale` before pausing so that `ResumeGame` can restore it.

---

#### 2.3.3 `ResumeGame`

```csharp
public static void ResumeGame(bool smooth = true)
```

* **Purpose**: Reverse the effect of `PauseGame` by restoring `Time.timeScale` to whatever it was before pausing (stored in `_previousTimeScale`). If `smooth=true`, transitions back over 0.2s; otherwise sets it instantly.
* **Parameters**:

  * `smooth` (bool): If `true`, fade from 0 → `_previousTimeScale` over 0.2s, else instantly set.

```csharp
// Example:
TimeHelper.ResumeGame(smooth: false);
```

---

### 2.4 Scheduling System

Queue up simple `Action` callbacks to run after a specified delay (in seconds), optionally based on unscaled time (ignoring `Time.timeScale`). You can also cancel them before they fire.

#### 2.4.1 `ScheduleAction`

```csharp
public static string ScheduleAction(
    Action action,
    float delay,
    bool useUnscaledTime = false
)
```

* **Purpose**: Create a one‐shot scheduled action that will execute `action` after `delay` seconds. Returns a GUID string (`id`) so you can cancel it if needed.
* **Parameters**:

  * `action` (Action): The callback to invoke once the delay has passed.
  * `delay` (float): How many seconds to wait before executing.
  * `useUnscaledTime` (bool): If `true`, measure delay using `Time.unscaledTime` (ignores `timeScale`); otherwise use `Time.time`.
* **Returns**: A `string` ID (GUID) that can be used later to cancel.

```csharp
// Example: schedule a power‐up to expire after 10s of unscaled time:
string id = TimeHelper.ScheduleAction(() => PowerUpExpired(), 10f, useUnscaledTime: true);
```

Internally, a new `ScheduledAction` is created:

```csharp
new ScheduledAction {
    Id = id,
    Action = action,
    ExecutionTime = (useUnscaledTime ? Time.unscaledTime : Time.time) + delay,
    UseUnscaledTime = useUnscaledTime
}
```

and appended to `_scheduledActions`.

---

#### 2.4.2 `CancelScheduledAction`

```csharp
public static void CancelScheduledAction(string id)
```

* **Purpose**: Prevent a previously scheduled action (by its GUID `id`) from firing.
* **Parameters**:

  * `id` (string): The identifier returned by `ScheduleAction`.

```csharp
// Example:
TimeHelper.CancelScheduledAction(id);
```

Internally, removes all `ScheduledAction` entries where `action.Id == id`.

---

#### 2.4.3 How Scheduled Actions Fire: `UpdateScheduledActions`

Part of `TimeHelper.Update()`.

```csharp
// Pseudocode:
var currentTime = Time.time;
var unscaledTime = Time.unscaledTime;
foreach (var scheduled in _scheduledActions)
{
    float now = scheduled.UseUnscaledTime ? unscaledTime : currentTime;
    if (now >= scheduled.ExecutionTime)
    {
        scheduled.Action?.Invoke();
        completed.Add(scheduled);
    }
}
foreach (var s in completed)
    _scheduledActions.Remove(s);
```

* Each frame, check each `ScheduledAction` whether “now” has reached or passed `ExecutionTime`.
* If so, invoke the callback and mark it to remove from the list.

---

### 2.5 Utility Methods

#### 2.5.1 `FormatTime`

```csharp
public static string FormatTime(float timeInSeconds, bool includeHours = false)
```

* **Purpose**: Convert a floating‐point number of seconds into a formatted string of the form:

  * If `includeHours=true` or if `timeInSeconds ≥ 3600`: `HH:MM:SS`
  * Otherwise: `MM:SS`
* **Parameters**:

  * `timeInSeconds` (float): Total seconds.
  * `includeHours` (bool): Force `HH:MM:SS` format even if hours = 0.
* **Returns**:

  * e.g. `FormatTime(75f) → "01:15"`,
  * `FormatTime(3675f, includeHours: true) → "01:01:15"`

```csharp
string s1 = TimeHelper.FormatTime(90f);         // "01:30"
string s2 = TimeHelper.FormatTime(4520f, true);  // "01:15:20"
```

---

#### 2.5.2 `FormatTimeSpan`

```csharp
public static string FormatTimeSpan(TimeSpan timeSpan, bool shortFormat = false)
```

* **Purpose**: Given a .NET `TimeSpan`, turn it into a human‐friendly string.
* **Parameters**:

  * `timeSpan` (TimeSpan): The span to format.
  * `shortFormat` (bool):

    * If `true`: use only the largest non‐zero unit with its suffix (e.g. “2d” or “5h” or “30s”).
    * If `false`: list all nonzero units in descending order (e.g. “1 day, 3 hours, 5 minutes, 12 seconds”).
* **Returns**:

  * `shortFormat = true`:

    * If `timeSpan.TotalDays ≥ 1`, returns `"Xd"` (days).
    * Else if `TotalHours ≥ 1`, `"Xh"`.
    * Else if `TotalMinutes ≥ 1`, `"Xm"`.
    * Else `"Xs"`.
  * `shortFormat = false`:

    * Builds a list of each nonzero component, e.g.:

      * `new TimeSpan(1,2,3,4)` → `"1 day, 2 hours, 3 minutes, 4 seconds"`
      * `new TimeSpan(0,0,5,0)` → `"5 minutes"`

```csharp
TimeSpan t1 = TimeSpan.FromSeconds(45);
Debug.Log(TimeHelper.FormatTimeSpan(t1, true));    // "45s"
Debug.Log(TimeHelper.FormatTimeSpan(t1, false));   // "45 seconds"

TimeSpan t2 = new TimeSpan(0, 1, 30, 0);
Debug.Log(TimeHelper.FormatTimeSpan(t2, true));    // "1h"
Debug.Log(TimeHelper.FormatTimeSpan(t2, false));   // "1 hour, 30 minutes"
```

---

### 2.6 Time Measurement

#### 2.6.1 `MeasureExecutionTime` (Synchronous)

```csharp
public static TimeSpan MeasureExecutionTime(Action action)
```

* **Purpose**: Determine how long a synchronous `action()` takes to run, using `Stopwatch`.
* **Parameters**:

  * `action` (Action): The code block to time.
* **Returns**: A `TimeSpan` representing elapsed time.

```csharp
TimeSpan elapsed = TimeHelper.MeasureExecutionTime(() => {
    // Some synchronous work, e.g., a heavy loop
    for (int i = 0; i < 1000000; i++) { /* ... */ }
});
Debug.Log($"It took {elapsed.TotalMilliseconds:F2} ms");
```

---

#### 2.6.2 `MeasureExecutionTimeAsync` (Asynchronous)

```csharp
public static async Task<TimeSpan> MeasureExecutionTimeAsync(Func<Task> action)
```

* **Purpose**: Like `MeasureExecutionTime`, but times an `async Task`.
* **Parameters**:

  * `action` (Func<Task>): An asynchronous delegate to run and await.
* **Returns**: A `TimeSpan` representing the elapsed time between start and `await action()` completion.

```csharp
TimeSpan elapsed = await TimeHelper.MeasureExecutionTimeAsync(async () => {
    // Example: wait for an async web request or Task.Delay
    await Task.Delay(500);
});
Debug.Log($"Async action took {elapsed.TotalMilliseconds:F2} ms");
```

---

### 2.7 Update Management

Because timers, cooldowns, and scheduled actions must be driven each frame, `TimeHelper` provides:

```csharp
public static void Update()
```

* **Purpose**: Should be called once per frame—typically from a MonoBehaviour’s `Update()`—to advance all timers, remove expired cooldowns, and fire scheduled actions in a timely manner.
* **Behavior**:

  1. Calls `UpdateTimers()`: advances `TimeRemaining` for each running `Timer`, invokes `OnComplete`, removes non‐looping timers as they finish.
  2. Calls `UpdateCooldowns()`: removes any cooldown entries whose expiration time ≤ `Time.time`.
  3. Calls `UpdateScheduledActions()`: checks each `ScheduledAction` against either `Time.time` or `Time.unscaledTime` (depending on its `UseUnscaledTime` flag) and fires plus removes those whose `ExecutionTime` has passed.

You must ensure some MonoBehaviour in your scene calls this every frame, for example:

```csharp
public class TimeHelperDriver : MonoBehaviour
{
    private void Update()
    {
        TimeHelper.Update();
    }
}
```

Place this on a GameObject that persists (e.g. a “GameManager”) so all your timers and scheduled tasks remain active.

---

## 3. Helper Classes & Data Structures

Internally, `TimeHelper` uses two small helper classes to organize timer and scheduled‐action data.

### 3.1 `Timer` (private)

```csharp
private class Timer
{
    public float Duration { get; set; }
    public float TimeRemaining { get; set; }
    public Action OnComplete { get; set; }
    public bool IsLooping { get; set; }
    public bool IsRunning { get; set; }
}
```

* **Fields**:

  * `Duration`: Original countdown length in seconds.
  * `TimeRemaining`: Seconds left until next completion callback.
  * `OnComplete`: Callback to invoke when this timer finishes.
  * `IsLooping`: If `true`, once it hits zero, it resets to `Duration` again (and invokes callback again).
  * `IsRunning`: If `false`, the countdown is paused—`TimeRemaining` does not change.

### 3.2 `ScheduledAction` (private)

```csharp
private class ScheduledAction
{
    public string Id { get; set; }
    public Action Action { get; set; }
    public float ExecutionTime { get; set; }
    public bool UseUnscaledTime { get; set; }
}
```

* **Fields**:

  * `Id`: Unique identifier (GUID) returned by `ScheduleAction()` so you can cancel by ID.
  * `Action`: The callback to run once the delay elapses.
  * `ExecutionTime`: The absolute timestamp at which to fire—either `Time.time + delay` or `Time.unscaledTime + delay`.
  * `UseUnscaledTime`: If `true`, compare `ExecutionTime` against `Time.unscaledTime`; otherwise against `Time.time`.

---

## 4. Usage Examples

Below are common scenarios demonstrating how to leverage various features of `TimeHelper`.

### 4.1 Countdown Timer Example

```csharp
public class ExampleTimerUsage : MonoBehaviour
{
    private void Start()
    {
        // Begin a timer called “Respawn” for 10 seconds; when complete, call OnRespawnReady()
        TimeHelper.StartTimer("Respawn", 10f, OnRespawnReady, loop: false);
    }

    private void OnEnable()
    {
        // Suppose we want to pause/resume the timer manually
        // On pressing “P” we pause; on pressing “R” we resume
    }

    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.P))
            TimeHelper.PauseTimer("Respawn");

        if (Input.GetKeyDown(KeyCode.R))
            TimeHelper.ResumeTimer("Respawn");

        // Display the remaining time on screen
        float left = TimeHelper.GetRemainingTime("Respawn");
        Debug.Log($"Respawn in: {left:F1}s");
    }

    private void OnRespawnReady()
    {
        Debug.Log("Player can respawn now!");
    }
}
```

* **Remember**: Another MonoBehaviour must call `TimeHelper.Update()` each frame (e.g., in a `GameManager`), or the timer will not tick down.

---

### 4.2 Cooldown Example (Ability Use)

```csharp
public class AbilityController : MonoBehaviour
{
    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            if (TimeHelper.IsCooldownComplete("Fireball"))
            {
                CastFireball();
                TimeHelper.StartCooldown("Fireball", 3.0f); // 3s cooldown
            }
            else
            {
                float wait = TimeHelper.GetCooldownRemaining("Fireball");
                Debug.Log($"Fireball ready in {wait:F1}s");
            }
        }
    }

    void CastFireball()
    {
        Debug.Log("Casting Fireball!");
        // ... spawn projectile etc.
    }
}
```

* Every time the player hits Space, we check `IsCooldownComplete("Fireball")`. If so, we cast and start a new 3s cooldown. If not, we log the remaining cooldown.

---

### 4.3 Scheduling Example (Delayed Callback)

```csharp
public class DelayedMessage : MonoBehaviour
{
    private string scheduledId;

    private void Start()
    {
        // Schedule a message to appear 5s later, ignoring timeScale (useUnscaledTime = true).
        scheduledId = TimeHelper.ScheduleAction(() => 
        {
            Debug.Log("This message appears 5 seconds from Start (unscaled).");
        }, 5f, useUnscaledTime: true);
    }

    private void Update()
    {
        // If player presses “C”, cancel that scheduled message before it runs.
        if (Input.GetKeyDown(KeyCode.C))
        {
            TimeHelper.CancelScheduledAction(scheduledId);
            Debug.Log("Scheduled message canceled.");
        }
    }
}
```

* The action will fire 5 seconds after `Start` (regardless of `Time.timeScale`). Pressing “C” before that time cancels it.

---

### 4.4 Smooth Time Scale (Slow-Motion)

```csharp
public class SlowMotionExample : MonoBehaviour
{
    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.S))
        {
            // Enter slow motion at half speed over 1 second
            _ = TimeHelper.SetTimeScale(0.5f, duration: 1f);
        }
        if (Input.GetKeyDown(KeyCode.N))
        {
            // Return to normal speed instantly
            _ = TimeHelper.SetTimeScale(1f, duration: 0f);
        }
    }
}
```

* Press “S” to slowly fade from whatever the current `timeScale` is down to 0.5 over 1 real‐second.
* Press “N” to instantly snap back to `timeScale = 1`.

---

### 4.5 Pause/Resume Entire Game

```csharp
public class PauseMenu : MonoBehaviour
{
    private Action _resumeAction;

    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.Escape))
        {
            if (_resumeAction == null)
            {
                // Pause the game with a smooth fade in of timeScale
                _resumeAction = TimeHelper.PauseGame(smooth: true);
                ShowPauseUI();
            }
            else
            {
                // Already paused; resume
                _resumeAction();
                _resumeAction = null;
                HidePauseUI();
            }
        }
    }

    void ShowPauseUI() { /* enable menu UI */ }
    void HidePauseUI() { /* hide menu UI */ }
}
```

* `PauseGame()` returns a delegate that, when invoked, resumes back to the previous time scale. Stores it in `_resumeAction`.
* Pressing Escape toggles pause/unpause accordingly.

---

### 4.6 Time Formatting Examples

```csharp
float gameSeconds = 125f;
string mmss = TimeHelper.FormatTime(gameSeconds);          
// mmss == "02:05"

float longSeconds = 3725f;
string hhmmss = TimeHelper.FormatTime(longSeconds, includeHours: true);
// hhmmss == "01:02:05"

TimeSpan span = new TimeSpan(1, 3, 15, 30); // 1d, 3h, 15m, 30s
string human = TimeHelper.FormatTimeSpan(span, shortFormat: false);
// human == "1 day, 3 hours, 15 minutes, 30 seconds"
string humanShort = TimeHelper.FormatTimeSpan(span, shortFormat: true);
// humanShort == "1d"
```

---

### 4.7 Measuring Execution Time

```csharp
void HeavyComputation()
{
    // ... some CPU‐intensive loop ...
    for (int i = 0; i < 10_000_000; i++) { /* ... */ }
}

async Task HeavyNetworkRequestAsync()
{
    await Task.Delay(700); // simulate network call
}

void ExampleMeasure()
{
    TimeSpan syncTime = TimeHelper.MeasureExecutionTime(HeavyComputation);
    Debug.Log($"Sync code took {syncTime.TotalMilliseconds:F2} ms");

    Time.timeScale = 0.5f; // slow motion, but MeasureExecutionTimeAsync uses Stopwatch, unaffected by timeScale
    TimeHelper.MeasureExecutionTimeAsync(async () => {
        await HeavyNetworkRequestAsync();
    }).ContinueWith(t => {
        Debug.Log($"Async operation took {t.Result.TotalMilliseconds:F2} ms");
    });
}
```

* The synchronous measurement uses a high‐resolution `Stopwatch`, so it is independent of `Time.timeScale`.
* The asynchronous version likewise measures only wall‐clock time, ignoring Unity’s `timeScale`.

---

## 5. Public Method Summary Table

| Method                                                                                   | Description                                                                                                                                                                                                               |
| ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`StartTimer(string id, float duration, Action onComplete = null, bool loop = false)`** | Create & start a named countdown timer. Calls `onComplete` when the timer reaches zero. If `loop=true`, resets & continues.                                                                                               |
| **`StopTimer(string id)`**                                                               | Stop and remove the timer identified by `id` (if it exists).                                                                                                                                                              |
| **`PauseTimer(string id)`**                                                              | Pause a running timer (it stops counting down until resumed).                                                                                                                                                             |
| **`ResumeTimer(string id)`**                                                             | Resume a paused timer (it continues counting down from wherever it left off).                                                                                                                                             |
| **`GetRemainingTime(string id)`**                                                        | Return how many seconds remain on the timer `id`, or 0 if the timer does not exist.                                                                                                                                       |
| **`StartCooldown(string id, float duration)`**                                           | Begin a “cooldown” that expires `duration` seconds from now. Internally stores `Time.time + duration`.                                                                                                                    |
| **`IsCooldownComplete(string id)`**                                                      | Return `true` if no cooldown exists for `id` or if `Time.time` has passed the stored expiration. Removes the entry if expired.                                                                                            |
| **`GetCooldownRemaining(string id)`**                                                    | Return how many seconds until the cooldown `id` expires (clamped ≥ 0), or 0 if no cooldown.                                                                                                                               |
| **`SetTimeScale(float targetScale, float duration = 0f)`**                               | Set Unity’s `Time.timeScale` to `targetScale`. If `duration>0`, interpolate smoothly from the current scale to `targetScale` over `duration` unscaled seconds.                                                            |
| **`PauseGame(bool smooth = true)`**                                                      | Pause gameplay by fading `timeScale` to 0 (over 0.2s if `smooth=true`, otherwise instantly). Stores previous scale to restore later. Returns an `Action` that, when invoked, calls `ResumeGame(smooth)`.                  |
| **`ResumeGame(bool smooth = true)`**                                                     | Resume gameplay from a paused state by restoring `timeScale` to the stored previous scale. Fades over 0.2s if `smooth=true`, otherwise instantly.                                                                         |
| **`ScheduleAction(Action action, float delay, bool useUnscaledTime = false)`**           | Schedule an `action` to run once after `delay` seconds. If `useUnscaledTime=true`, uses `Time.unscaledTime` (ignoring `timeScale`), else uses `Time.time`. Returns a GUID `string` ID for cancellation.                   |
| **`CancelScheduledAction(string id)`**                                                   | Cancel a previously scheduled action (won’t fire) identified by `id` (returned from `ScheduleAction`).                                                                                                                    |
| **`FormatTime(float timeInSeconds, bool includeHours = false)`**                         | Convert `timeInSeconds` to a string `"MM:SS"` (or `"HH:MM:SS"` if `includeHours=true` or hours>0).                                                                                                                        |
| **`FormatTimeSpan(TimeSpan timeSpan, bool shortFormat = false)`**                        | Convert `timeSpan` into a human‐readable string. If `shortFormat=true`, returns only the largest unit (e.g., `"2h"`); otherwise, lists all nonzero units, e.g. `"1 day, 3 hours, 15 minutes, 30 seconds"`.                |
| **`MeasureExecutionTime(Action action)`**                                                | Run a synchronous `action()` and return a `TimeSpan` indicating how long it took (using `Stopwatch`).                                                                                                                     |
| **`MeasureExecutionTimeAsync(Func<Task> action)`**                                       | Run an asynchronous `action()` (awaited) and return a `TimeSpan` measuring the wall‐clock time from just before invocation until after `await action()`.                                                                  |
| **`Update()`**                                                                           | Must be called once per frame to update timers, cooldowns, and scheduled actions. Internally invokes `UpdateTimers()`, `UpdateCooldowns()`, and `UpdateScheduledActions()`. Place in a MonoBehaviour’s `Update()` method. |

---

## 6. Implementation Notes & Best Practices

* **Central Update Driver**

  * Because `TimeHelper` relies on `Update()` to tick timers and fire scheduled callbacks, you must have a MonoBehaviour somewhere in your scene that calls `TimeHelper.Update()` every frame. For example:

    ```csharp
    public class TimeHelperDriver : MonoBehaviour
    {
        private void Update()
        {
            TimeHelper.Update();
        }
    }
    ```

    Attach that script to any “persistent” GameObject (e.g., a `GameManager` set to `DontDestroyOnLoad`).

* **Unique IDs**

  * Always choose unique, descriptive `id` strings for timers, cooldowns, and scheduled actions to avoid collisions. Example: `"enemySpawn_Orc1"`, `"cooldown_Jump"`, `"schedule_ShowHint"`, etc.

* **TimeScale Interactions**

  * Timers and cooldowns use `Time.deltaTime` and `Time.time`, so if `Time.timeScale` is set to 0 (paused), timers and cooldowns will also “pause.” If you want a timer or scheduled action to ignore `timeScale`, schedule it with `useUnscaledTime=true` and modify `TimeHelper.UpdateScheduledActions()` accordingly (it already checks unscaled time for those).
  * When you call `PauseGame(smooth: true)`, `TimeHelper.SetTimeScale(0, 0.2f)` will gradually slow down over 0.2s. Note that while the scale is lower (e.g., 0.5→0 over 0.2s), all timers and cooldowns tick proportionally slower—because they use `Time.deltaTime` and `Time.time`. In some cases, you may want to keep certain timers running in unscaled time; for those, consider using `ScheduleAction(..., useUnscaledTime:true)` instead.

* **Exception Safety**

  * Scheduled actions are invoked in a `try/catch` block within `UpdateScheduledActions()`, so one failing callback won’t stop other actions. Errors are logged to the console.

* **Looping Timers**

  * If you pass `loop = true` to `StartTimer`, the timer’s `OnComplete` callback is invoked every `duration` seconds. The timer’s `TimeRemaining` is reset to `Duration`. You can manually `StopTimer(id)` at any time to cancel further loops.

* **Canceling Scheduled Actions**

  * If you call `ScheduleAction` and later decide you don’t want it to run (e.g. the player died before a “game over” UI should show), call `CancelScheduledAction(id)`. The action will never fire.

* **Measuring Async vs. Sync**

  * `MeasureExecutionTime` wraps a synchronous delegate, while `MeasureExecutionTimeAsync` awaits an asynchronous delegate. Both measure actual wall‐clock elapsed time (they do not depend on `Time.timeScale`).

---

## 7. Putting It All Together

Below is a simple **GameManager** example that demonstrates how you might wire up `TimeHelper.Update()` and use multiple features.

```csharp
using UnityEngine;
using System;

public class GameManager : MonoBehaviour
{
    private string respawnTimerId;
    private string powerUpCooldownId;
    private Action resumeAction;

    private void Start()
    {
        // Example: Schedule a “welcome” message after 2 seconds unscaled
        TimeHelper.ScheduleAction(() => Debug.Log("Welcome to the game!"), 2f, useUnscaledTime: true);

        // Example: Start a looping timer that logs every 5 seconds
        TimeHelper.StartTimer("heartbeat", 5f, OnHeartbeat, loop: true);

        // Example: Start a 10s respawn timer
        respawnTimerId = "respawnTimer";
        TimeHelper.StartTimer(respawnTimerId, 10f, OnPlayerRespawn, loop: false);

        // Example: Start a 3s cooldown for “dash”
        powerUpCooldownId = "dashCooldown";
        TimeHelper.StartCooldown(powerUpCooldownId, 3f);
    }

    private void OnHeartbeat()
    {
        Debug.Log("Heartbeat—5s passed.");
    }

    private void OnPlayerRespawn()
    {
        Debug.Log("Player can respawn now!");
    }

    private void Update()
    {
        // Drive TimeHelper’s internals each frame
        TimeHelper.Update();

        // Example: Pause / Resume the game with the “P” key
        if (Input.GetKeyDown(KeyCode.P))
        {
            if (resumeAction == null)
            {
                resumeAction = TimeHelper.PauseGame(smooth: true);
                Debug.Log("Game paused.");
            }
            else
            {
                resumeAction();
                resumeAction = null;
                Debug.Log("Game resumed.");
            }
        }

        // Example: Attempt to dash with spacebar when cooldown is complete
        if (Input.GetKeyDown(KeyCode.Space))
        {
            if (TimeHelper.IsCooldownComplete(powerUpCooldownId))
            {
                Debug.Log("Dashing!");
                TimeHelper.StartCooldown(powerUpCooldownId, 3f);
            }
            else
            {
                float wait = TimeHelper.GetCooldownRemaining(powerUpCooldownId);
                Debug.Log($"Dash on cooldown—wait {wait:F1}s");
            }
        }

        // Example: Pause the respawn timer if “O” is held down
        if (Input.GetKey(KeyCode.O))
            TimeHelper.PauseTimer(respawnTimerId);
        else
            TimeHelper.ResumeTimer(respawnTimerId);
    }

    private void OnDestroy()
    {
        // Always clean up when this object is destroyed
        TimeHelper.StopTimer("heartbeat");
        TimeHelper.StopTimer(respawnTimerId);
    }
}
```

**Explanation of Key Points**

* In `Start()`, we schedule an unscaled message, start a looping heartbeat timer, start a one‐shot respawn timer, and start a 3s dash cooldown.
* In `Update()`, we call `TimeHelper.Update()` so that all timers and scheduled actions can progress.
* Pressing “P” toggles pause/unpause via `PauseGame`/`resumeAction`.
* Pressing Space attempts a dash; if the cooldown is already complete, we dash and restart it; otherwise we display how many seconds remain.
* Holding “O” will pause the respawn timer (`PauseTimer`), so “OnPlayerRespawn” callback is deferred until “O” is released.

This demonstrates timers, cooldowns, scheduling, and time‐scale control all working together.

---

## 8. Complete Method Reference Table

Below is a consolidated table listing all public members of **TimeHelper** (grouped by region):

| **Region**             | **Method**                                                                           | **Description**                                                                                                                                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Timer Management**   | `StartTimer(string id, float duration, Action onComplete = null, bool loop = false)` | Create & start a named timer that counts down from `duration` seconds. Optionally loop and call `onComplete` when it hits zero. Overwrites any existing timer with same `id`.                                        |
|                        | `StopTimer(string id)`                                                               | Immediately stop & remove the timer identified by `id`.                                                                                                                                                              |
|                        | `PauseTimer(string id)`                                                              | Pause a running timer (`TimeRemaining` stops decreasing).                                                                                                                                                            |
|                        | `ResumeTimer(string id)`                                                             | Resume a previously paused timer (it continues countdown).                                                                                                                                                           |
|                        | `GetRemainingTime(string id) → float`                                                | Return how many seconds are left on the timer `id`, or 0 if none exists.                                                                                                                                             |
| **Cooldown System**    | `StartCooldown(string id, float duration)`                                           | Begin a cooldown identified by `id` that will expire in `duration` seconds. Internally stores `Time.time + duration`.                                                                                                |
|                        | `IsCooldownComplete(string id) → bool`                                               | `true` if no cooldown exists or if the stored expiration time ≤ `Time.time`. If expired, removes the entry and returns `true`. Otherwise returns `false`.                                                            |
|                        | `GetCooldownRemaining(string id) → float`                                            | Return how many seconds remain until cooldown `id` expires (or 0 if none).                                                                                                                                           |
| **Time Scale Control** | `SetTimeScale(float targetScale, float duration = 0f) → Task`                        | Set `Time.timeScale` to `targetScale`. If `duration>0`, linearly interpolate from the current scale to `targetScale` over `duration` (measured in unscaled seconds).                                                 |
|                        | `PauseGame(bool smooth = true) → Action`                                             | Pause the game (scale → 0). If `smooth=true`, fade `timeScale` to 0 over 0.2s; else snap immediately. Returns an `Action` that, when invoked, calls `ResumeGame(smooth)`.                                            |
|                        | `ResumeGame(bool smooth = true)`                                                     | Resume the game from the previous timescale saved by `PauseGame`. If `smooth=true`, fade from 0 → previous scale over 0.2s; else snap immediately.                                                                   |
| **Scheduling System**  | `ScheduleAction(Action action, float delay, bool useUnscaledTime = false) → string`  | Schedule `action` to run once after `delay` seconds (unscaled if `useUnscaledTime=true`, else scaled). Returns a GUID string `id` you can use to cancel.                                                             |
|                        | `CancelScheduledAction(string id)`                                                   | Cancel a previously scheduled action (prevents it from firing) by its `id`.                                                                                                                                          |
| **Utility Methods**    | `FormatTime(float timeInSeconds, bool includeHours = false) → string`                | Convert `timeInSeconds` into `"MM:SS"` (or `"HH:MM:SS"` if `includeHours=true` or hours>0).                                                                                                                          |
|                        | `FormatTimeSpan(TimeSpan timeSpan, bool shortFormat = false) → string`               | Convert a `TimeSpan` into a human‐readable string. If `shortFormat=true`, return only the largest nonzero unit with its suffix (e.g. `"3h"`). Otherwise list all nonzero parts (e.g. `"1 day, 2 hours, 5 minutes"`). |
| **Time Measurement**   | `MeasureExecutionTime(Action action) → TimeSpan`                                     | Measure how long a synchronous `action()` takes to run and return a `TimeSpan`.                                                                                                                                      |
|                        | `MeasureExecutionTimeAsync(Func<Task> action) → Task<TimeSpan>`                      | Measure how long an asynchronous `action()` takes (awaited) and return a `TimeSpan`.                                                                                                                                 |
| **Update Management**  | `Update()`                                                                           | Must be called once per frame. Internally calls `UpdateTimers()`, `UpdateCooldowns()`, and `UpdateScheduledActions()`. Drives timers, cooldowns, and scheduled‐action execution.                                     |

---

## 9. Final Notes

* **Always call `TimeHelper.Update()` each frame**—otherwise timers, cooldowns, and scheduled actions will not advance.
* Choose **unique IDs** for each timer, cooldown, and scheduled action to prevent accidental clashes.
* If you need certain timers or scheduled actions to run even when the game is paused (`timeScale = 0`), schedule them with `useUnscaledTime = true` or implement your own logic inside `UpdateScheduledActions`.
* The **pause/unpause** methods conveniently return a callback to resume, so you can do:

  ```csharp
  var resume = TimeHelper.PauseGame();
  // … later …
  resume();
  ```
* Use **`MeasureExecutionTime`** to benchmark small code blocks or async tasks during development or profiling.

With this detailed guide and method summary, you should have everything you need to integrate **TimeHelper** into your Unity project and handle all common “time” scenarios in one place.
