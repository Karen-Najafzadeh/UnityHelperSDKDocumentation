Below is a complete guide to the **SceneHelper** utility, including detailed explanations of how each part works, real‐world usage examples, and a final table summarizing all public methods and their purposes.

---

## Overview

`SceneHelper` is a static Unity helper designed to centralize common scene‐management tasks. Its capabilities include:

1. **Asynchronous Scene Loading/Unloading**
2. **Fade‐in/Fade‐out Transitions**
3. **Additive Scene Management**
4. **Loading‐Screen Display with Progress Bar**
5. **Scene State Persistence & Restoration**
6. **Tracking Active Scenes**

Key dependencies:

* **`UnityEngine.SceneManagement.SceneManager`** for loading/unloading.
* **`DG.Tweening`** (DOTween) for potential tween setups (though this implementation uses manual `Lerp` loops).
* **`TextMeshProUGUI`** for on‐screen progress text.
* **`UnityEngine.UI.Image`** for fade overlays and progress bars.

Below, each region and method is explained in detail.

---

## 1. Scene Loading

### 1.1 `LoadSceneAsync`

```csharp
public static async Task LoadSceneAsync(
    string sceneName,
    LoadSceneMode mode = LoadSceneMode.Single,
    bool showLoadingScreen = true,
    bool persistState = false
)
```

**Purpose**
Load a given scene (by name) either in Single mode (replacing all existing scenes) or Additive mode (stacking alongside current). Supports an optional full‐screen loading UI or a simple fade transition. Optionally persists/restores “scene state” before and after the load.

**Parameters**

* `sceneName` (string): Name of the scene to load (must be added to Build Settings).
* `mode` (LoadSceneMode): `Single` or `Additive`. Default is `Single`.
* `showLoadingScreen` (bool): If `true`, display a loading‐screen UI (with progress bar). If `false`, perform a simple fade‐to‐black/fade‐from‐black.
* `persistState` (bool): If `true`, calls `SaveSceneState()` before unloading and `RestoreSceneState()` after loading to preserve object states across reloads.

**Behavior**

1. **Persist State (optional)**

   ```csharp
   if (persistState) SaveSceneState();
   ```

   * Iterates all root GameObjects in the currently active scene, invokes any attached `ISceneStateSerializable.SaveState()`, and stores results in `_sceneStates[sceneName]`.

2. **Show Loading Screen or Fade to Black**

   ```csharp
   if (showLoadingScreen) await ShowLoadingScreen();
   else await FadeToBlack(_defaultFadeTime);
   ```

   * If `showLoadingScreen == true`: Spawns a full‐screen Canvas with a dark background, a progress‐bar UI, and fade‐in alpha from 0→1 over `_defaultFadeTime` (0.5s).
   * If `false`: Calls `FadeToBlack(...)` to display a plain black overlay, alpha 0→1.

3. **Begin Async Loading**

   ```csharp
   var operation = SceneManager.LoadSceneAsync(sceneName, mode);
   operation.allowSceneActivation = false;
   ```

   * Starts loading the scene but prevents immediate activation until progress ≥ 0.9f (Unity’s async progress goes 0→0.9, then awaits `allowSceneActivation`).

4. **Track Progress**

   ```csharp
   while (operation.progress < 0.9f)
   {
       await Task.Yield();
       if (showLoadingScreen)
           UpdateLoadingProgress(operation.progress);
   }
   ```

   * While progress < 0.9, yield each frame. If using loading‐screen, call `UpdateLoadingProgress(float)` to update UI.

5. **Activate Scene**

   ```csharp
   operation.allowSceneActivation = true;
   await Task.Yield(); // Wait one more frame for activation
   ```

   * When progress reaches 0.9, setting `allowSceneActivation = true` completes load by transitioning the game to the new scene(s).

6. **Restore State (optional)**

   ```csharp
   if (persistState) RestoreSceneState();
   ```

   * If any object in the newly loaded scene implemented `ISceneStateSerializable`, its `LoadState(object)` is invoked with previously saved data.

7. **Hide Loading Screen or Fade from Black**

   ```csharp
   if (showLoadingScreen) await HideLoadingScreen();
   else await FadeFromBlack(_defaultFadeTime);
   ```

   * Hides UI via a fade‐out or destroys the fade canvas.

8. **Track Active Scenes**

   ```csharp
   if (mode == LoadSceneMode.Single) _activeScenes.Clear();
   _activeScenes.Add(sceneName);
   ```

   * If loading in Single mode, clear existing tracking. Then mark the newly loaded scene as active.

**Real‐World Example: Single Mode with Loading Screen**

```csharp
public async void OnUserPressedPlay()
{
    // Save any persistent data before switching
    await SceneHelper.LoadSceneAsync("MainLevel", LoadSceneMode.Single, showLoadingScreen: true, persistState: true);
    Debug.Log("MainLevel fully loaded and state restored.");
}
```

* Displays a fade‐in loading screen (0→100% progress).
* Loads “MainLevel” entirely, replacing previous scenes.
* Restores any stateful objects (if they implement `ISceneStateSerializable`).
* Finally hides the loading screen via fade‐out.

---

### 1.2 `UnloadSceneAsync`

```csharp
public static async Task UnloadSceneAsync(string sceneName, bool showTransition = true)
```

**Purpose**
Asynchronously unload a scene by name (if it’s currently active). Optionally wraps the unload with a fade transition.

**Parameters**

* `sceneName` (string): Name of the scene to unload.
* `showTransition` (bool): If `true`, fade to black before unload and fade back in afterward. Default is `true`.

**Behavior**

1. **Check Active Tracking**

   ```csharp
   if (!_activeScenes.Contains(sceneName)) return;
   ```

   * If scene is not currently tracked as active, do nothing.

2. **Fade to Black (optional)**

   ```csharp
   if (showTransition) await FadeToBlack(_defaultFadeTime);
   ```

3. **Begin Unload**

   ```csharp
   var operation = SceneManager.UnloadSceneAsync(sceneName);
   while (!operation.isDone)
       await Task.Yield();
   _activeScenes.Remove(sceneName);
   ```

4. **Fade from Black (optional)**

   ```csharp
   if (showTransition) await FadeFromBlack(_defaultFadeTime);
   ```

**Real‐World Example**

```csharp
public async void OnUserPressedBackToMenu()
{
    // Unload "MainLevel" with fade
    await SceneHelper.UnloadSceneAsync("MainLevel", showTransition: true);
    Debug.Log("MainLevel unloaded, returned to previous scene (e.g., MainMenu).");
}
```

* If “MainLevel” was tracked in `_activeScenes`, fades out, unloads it, then fades back in.

---

## 2. Scene State Management

When you need to persist certain runtime‐generated data (e.g., object positions, scores, unlocked doors) across scene unload/load, use the `ISceneStateSerializable` interface. `SceneHelper` will automatically invoke these methods to gather and restore states.

### 2.1 `SaveSceneState`

```csharp
public static void SaveSceneState()
```

**Purpose**
Iterate through all root GameObjects in the currently active scene, find any components implementing `ISceneStateSerializable`, and gather their states by calling `SaveState()`. The results are stored in a private dictionary `_sceneStates[currentSceneName]`.

**Behavior**

1. **Identify Current Scene**

   ```csharp
   var currentScene = SceneManager.GetActiveScene();
   var state = new SceneState();
   ```

2. **Iterate Root Objects**

   ```csharp
   var rootObjects = currentScene.GetRootGameObjects();
   foreach (var obj in rootObjects)
   {
       var serializableObjects = obj.GetComponentsInChildren<ISceneStateSerializable>(true);
       foreach (var serializable in serializableObjects)
       {
           string stateKey = serializable.GetStateKey();
           if (!string.IsNullOrEmpty(stateKey))
               state.ObjectStates[stateKey] = serializable.SaveState();
       }
   }
   ```

   * For each `ISceneStateSerializable`, invoke `GetStateKey()` to obtain a unique string. Then call `SaveState()` to retrieve an `object` representing its internal data (could be a simple struct, a class, etc.).
   * Store into `state.ObjectStates[stateKey]`.

3. **Store in Dictionary**

   ```csharp
   _sceneStates[currentScene.name] = state;
   ```

**`SceneState` Class**

```csharp
private class SceneState
{
    public Dictionary<string, object> ObjectStates = new Dictionary<string, object>();
}
```

* Maps each `stateKey` to the saved data object.

**Real‐World Example**

```csharp
// Example component implementing ISceneStateSerializable:
public class DoorController : MonoBehaviour, SceneHelper.ISceneStateSerializable
{
    private bool _isOpen = false;

    public string GetStateKey() => $"Door_{gameObject.name}";

    public object SaveState()
    {
        return _isOpen; // return a bool representing open/closed
    }

    public void LoadState(object state)
    {
        if (state is bool open) 
        {
            _isOpen = open;
            // Update visual state (play animation, etc.)
        }
    }

    // ... logic that toggles _isOpen at runtime ...
}
```

* When `SaveSceneState()` runs, each `DoorController` returns its `_isOpen` bool under the key `"Door_DoorName"`.
* Later, after a reload, `RestoreSceneState()` finds the same component, looks up `"Door_DoorName"` in the saved dictionary, and invokes `LoadState(open)` to restore open/closed status.

---

### 2.2 `RestoreSceneState`

```csharp
public static void RestoreSceneState()
```

**Purpose**
If a previous call to `SaveSceneState()` stored data for this scene, find any `ISceneStateSerializable` components in the newly loaded scene and give them their saved state via `LoadState(object)`.

**Behavior**

1. **Identify Current Scene**

   ```csharp
   var currentScene = SceneManager.GetActiveScene();
   if (!_sceneStates.TryGetValue(currentScene.name, out SceneState state)) return;
   ```

2. **Iterate and Restore**

   ```csharp
   var rootObjects = currentScene.GetRootGameObjects();
   foreach (var obj in rootObjects)
   {
       var serializableObjects = obj.GetComponentsInChildren<ISceneStateSerializable>(true);
       foreach (var serializable in serializableObjects)
       {
           string stateKey = serializable.GetStateKey();
           if (!string.IsNullOrEmpty(stateKey) && state.ObjectStates.TryGetValue(stateKey, out object savedState))
           {
               serializable.LoadState(savedState);
           }
       }
   }
   ```

**Real‐World Example (Continuation)**

* After reloading “MainLevel,” the `DoorController` from the previous scene is re‐created. `RestoreSceneState()` finds `DoorController`, reads its `"Door_DoorName"` entry (a `bool`), and calls `LoadState(open)` to restore it.

---

## 3. Scene Transitions

For simple fade‐in/fade‐out effects (e.g., between scenes or UI screens), SceneHelper provides two private methods plus a helper to create a full‐screen overlay.

### 3.1 `FadeToBlack`

```csharp
private static async Task FadeToBlack(float duration)
```

**Purpose**
Display a black overlay that smoothly increases its alpha from 0→1 over `duration` seconds.

**Behavior**

1. **Get or Instantiate Canvas**

   ```csharp
   var fadeScreen = CreateOrGetFadeScreen();
   var canvasGroup = fadeScreen.GetComponent<CanvasGroup>();
   ```

   * `CreateOrGetFadeScreen()` either finds an existing GameObject named `"SceneTransitionCanvas"` or creates a new Canvas + Image child configured as a fullscreen black rectangle. A `CanvasGroup` is attached for alpha control.

2. **Lerp Alpha**

   ```csharp
   float elapsed = 0f;
   while (elapsed < duration)
   {
       elapsed += Time.deltaTime;
       canvasGroup.alpha = Mathf.Lerp(0f, 1f, elapsed / duration);
       await Task.Yield();
   }
   ```

**Note**

* After this completes, the black overlay remains at alpha = 1 (fully opaque) until `FadeFromBlack` is called.

---

### 3.2 `FadeFromBlack`

```csharp
private static async Task FadeFromBlack(float duration)
```

**Purpose**
Reverse of `FadeToBlack`: smoothly decrease alpha from 1→0, then destroy the black overlay when done.

**Behavior**

1. **Get Canvas & CanvasGroup**

   ```csharp
   var fadeScreen = CreateOrGetFadeScreen();
   var canvasGroup = fadeScreen.GetComponent<CanvasGroup>();
   ```

2. **Lerp Alpha**

   ```csharp
   float elapsed = 0f;
   while (elapsed < duration)
   {
       elapsed += Time.deltaTime;
       canvasGroup.alpha = Mathf.Lerp(1f, 0f, elapsed / duration);
       await Task.Yield();
   }
   ```

3. **Destroy Overlay**

   ```csharp
   GameObject.Destroy(fadeScreen);
   ```

---

### 3.3 `CreateOrGetFadeScreen`

```csharp
private static GameObject CreateOrGetFadeScreen()
```

**Purpose**
Ensure that there is a single “fade‐screen” Canvas in the scene. If one exists, return it; otherwise, create it.

**Behavior**

1. **Look for Existing**

   ```csharp
   var existing = GameObject.Find("SceneTransitionCanvas");
   if (existing != null) return existing;
   ```

2. **Instantiate**

   ```csharp
   var canvas = new GameObject("SceneTransitionCanvas", typeof(Canvas), typeof(CanvasGroup));
   canvas.GetComponent<Canvas>().renderMode = RenderMode.ScreenSpaceOverlay;
   canvas.GetComponent<Canvas>().sortingOrder = 9999;
   ```

3. **Add Full-Screen Image**

   ```csharp
   var image = new GameObject("FadeImage", typeof(Image));
   image.transform.SetParent(canvas.transform, false);
   var imgComp = image.GetComponent<Image>();
   imgComp.color = _defaultFadeColor; // black by default
   imgComp.GetComponent<RectTransform>().sizeDelta = new Vector2(Screen.width * 1.5f, Screen.height * 1.5f);
   ```

4. **Don’t Destroy on Load**

   ```csharp
   GameObject.DontDestroyOnLoad(canvas);
   return canvas;
   ```

**Why 1.5× Screen Dimensions?**

* Ensures the black overlay fully covers the screen even if aspect ratio changes or camera scaling shifts slightly.

---

## 4. Loading Screen

When `showLoadingScreen == true` in `LoadSceneAsync()`, SceneHelper displays a full‐screen “loading” UI that shows progress as a percentage. The UI is composed of:

* A black background (faded in via `CanvasGroup`).
* A progress bar (Image fill).
* A TextMeshProUGUI element showing “0% → 100%.”

### 4.1 Fields

```csharp
private static GameObject _loadingScreen;
private static Image _progressBar;
private static TextMeshProUGUI _progressText;
```

* `_loadingScreen`: The root Canvas GameObject for loading UI.
* `_progressBar`: The `Image` component whose `fillAmount` is updated to reflect progress.
* `_progressText`: The `TextMeshProUGUI` that displays the numeric percentage.

---

### 4.2 `ShowLoadingScreen`

```csharp
private static async Task ShowLoadingScreen()
```

**Purpose**
Instantiate the loading UI and fade it in (alpha: 0→1) over `_defaultFadeTime`.

**Behavior**

1. **Create UI**

   ```csharp
   _loadingScreen = CreateLoadingScreen();
   var canvasGroup = _loadingScreen.GetComponent<CanvasGroup>();
   ```

2. **Fade In Alpha**

   ```csharp
   float elapsed = 0f;
   while (elapsed < _defaultFadeTime)
   {
       elapsed += Time.deltaTime;
       canvasGroup.alpha = Mathf.Lerp(0f, 1f, elapsed / _defaultFadeTime);
       await Task.Yield();
   }
   ```

* After fading in, the UI remains at full opacity (black background + visible progress bar).

**`CreateLoadingScreen()`**
Lays out a Canvas containing:

* **Background**: A fullscreen black Image (same as fade screen).
* **Progress Bar Background**: A grey `Image` sized 500×20, centered.
* **Progress Bar Fill**: A white `Image`, child of ProgressBarBg, same size (500×20). Its `fillAmount` will be set in `UpdateLoadingProgress()`.
* **Progress Text**: A `TextMeshProUGUI` centered on screen, font size 24, colored white, showing percentage.

It also attaches a `CanvasGroup` to the root to allow alpha control.

---

### 4.3 `UpdateLoadingProgress`

```csharp
private static void UpdateLoadingProgress(float progress)
```

**Purpose**
Update the progress‐bar fill and text during scene loading.

* `progress` is expected in `0 … 0.9` range (Unity’s async progress). You could scale it to `0→1` by dividing by 0.9, but this implementation uses the raw value, so “90%” corresponds to `progress = 0.9`.

**Behavior**

```csharp
if (_progressBar != null)
    _progressBar.fillAmount = progress;

if (_progressText != null)
    _progressText.text = $"{(progress * 100f):F0}%";
```

* `fillAmount` (0→1) is directly set to `progress`.
* Text is formatted as an integer percentage.

---

### 4.4 `HideLoadingScreen`

```csharp
private static async Task HideLoadingScreen()
```

**Purpose**
Once the new scene is fully loaded, fade the loading‐screen UI out (alpha: 1→0) and then destroy it.

**Behavior**

1. **Guard**

   ```csharp
   if (_loadingScreen == null) return;
   ```
2. **Get CanvasGroup**

   ```csharp
   var canvasGroup = _loadingScreen.GetComponent<CanvasGroup>();
   ```
3. **Fade Out**

   ```csharp
   float elapsed = 0f;
   while (elapsed < _defaultFadeTime)
   {
       elapsed += Time.deltaTime;
       canvasGroup.alpha = Mathf.Lerp(1f, 0f, elapsed / _defaultFadeTime);
       await Task.Yield();
   }
   ```
4. **Destroy UI**

   ```csharp
   GameObject.Destroy(_loadingScreen);
   _loadingScreen = null;
   ```

---

## 5. Helper Classes & Interface

### 5.1 `SceneState` (private)

```csharp
private class SceneState
{
    public Dictionary<string, object> ObjectStates = new Dictionary<string, object>();
}
```

* Used internally to store the mapping of each `ISceneStateSerializable.GetStateKey()` → `SaveState()` output.
* Indexed by scene name in `_sceneStates`.

---

### 5.2 `ISceneStateSerializable` Interface

```csharp
public interface ISceneStateSerializable
{
    string GetStateKey();
    object SaveState();
    void LoadState(object state);
}
```

**Purpose**
Any component that implements this interface can:

* Provide a **unique key** (`GetStateKey()`) so that multiple objects don’t clobber each other.
* Return a serializable data object (`SaveState()`) representing its runtime state (position, health, unlocked, etc.).
* Re‐apply previously saved state (`LoadState(object)`) after a scene reload.

**Real‐World Example**

```csharp
public class PlayerStats : MonoBehaviour, SceneHelper.ISceneStateSerializable
{
    public int health;
    public int score;

    // Unique key for this component; could incorporate player ID, etc.
    public string GetStateKey() => $"PlayerStats_{gameObject.name}";

    // Return a simple struct or array
    public object SaveState()
    {
        return new PlayerSaveData
        {
            health = this.health,
            score = this.score
        };
    }

    public void LoadState(object state)
    {
        if (state is PlayerSaveData data)
        {
            this.health = data.health;
            this.score = data.score;
        }
    }

    [Serializable]
    private struct PlayerSaveData
    {
        public int health;
        public int score;
    }
}
```

* On `SaveSceneState()`, `GetStateKey()`→ e.g. `"PlayerStats_Player1"`, and `SaveState()` returns a small struct.
* On `RestoreSceneState()`, that struct is fed back to `LoadState(...)`.

---

## 6. Method Summary Table

| Method                                                  | Description                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`LoadSceneAsync(string, LoadSceneMode, bool, bool)`** | Asynchronously load a scene by name. If `showLoadingScreen=true`, display a loading UI and update progress. If `persistState=true`, save/restore scene state before/after load. Also handles fade transitions if `showLoadingScreen=false`. Tracks the newly active scene in `_activeScenes`. |
| **`UnloadSceneAsync(string, bool)`**                    | Asynchronously unload a scene by name if it is currently tracked. If `showTransition=true`, fade to black before unload and fade back.                                                                                                                                                        |
| **`SaveSceneState()`**                                  | For the currently active scene, iterate all `ISceneStateSerializable` components and store each’s `SaveState()` under its `GetStateKey()` in `_sceneStates[sceneName]`.                                                                                                                       |
| **`RestoreSceneState()`**                               | For the currently active scene, find any `ISceneStateSerializable` components and call `LoadState(object)` with previously saved data from `_sceneStates[sceneName]`.                                                                                                                         |
| **`FadeToBlack(float duration)`**                       | Private helper: instantiate (or reuse) a full‐screen black Canvas‐Overlay and smoothly interpolate its alpha from 0→1 over `duration` seconds.                                                                                                                                                |
| **`FadeFromBlack(float duration)`**                     | Private helper: fade the existing black overlay’s alpha from 1→0 over `duration` seconds, then destroy it.                                                                                                                                                                                    |
| **`CreateOrGetFadeScreen()`**                           | Private helper: return the existing fade‐screen Canvas (`"SceneTransitionCanvas"`) if found; otherwise, create a new Canvas+Image covering the screen.                                                                                                                                        |
| **`ShowLoadingScreen()`**                               | Private helper: instantiate a full‐screen loading UI (Canvas + black background + progress bar + percentage text) and fade it in over `_defaultFadeTime`.                                                                                                                                     |
| **`HideLoadingScreen()`**                               | Private helper: fade out the loading UI’s CanvasGroup alpha from 1→0 over `_defaultFadeTime`, then destroy the UI object.                                                                                                                                                                     |
| **`UpdateLoadingProgress(float progress)`**             | Private helper: set the loading UI’s progress bar fill and percentage text based on `progress` (0→1).                                                                                                                                                                                         |
| **`CreateLoadingScreen()`**                             | Private helper: build and return a new loading UI GameObject consisting of a Canvas, background Image, progress‐bar Image, and TextMeshProUGUI.                                                                                                                                               |
| **`GetSaveSceneState()`** and **`_sceneStates`**        | (Internal data) A `Dictionary<string, SceneState>` mapping scene names to saved state dictionaries for each `ISceneStateSerializable` component inside that scene.                                                                                                                            |
| **`_activeScenes`**                                     | (Internal data) A `HashSet<string>` listing which scene names have been loaded via `LoadSceneAsync`.                                                                                                                                                                                          |
| **`SceneState` (private class)**                        | Holds a `Dictionary<string, object> ObjectStates` for serialized data from all `ISceneStateSerializable` components.                                                                                                                                                                          |
| **`ISceneStateSerializable` (interface)**               | Components that want to persist custom state: must provide `GetStateKey()`, `SaveState()`, and `LoadState(object)`.                                                                                                                                                                           |

---

## 7. Putting It All into Practice

Below is an example scenario where you use `SceneHelper` to:

1. Load a “Menu” scene on application start (no persistence needed, simple fade).
2. When the player clicks “Play,” load “Level1” with a loading screen and preserve any level‐specific data.
3. When the player finishes “Level1,” unload it and return to “Menu.”

```csharp
public class GameBootstrap : MonoBehaviour
{
    private async void Start()
    {
        // On startup, load MainMenu with fade (no loading screen, no state persistence)
        await SceneHelper.LoadSceneAsync("MainMenu", LoadSceneMode.Single, showLoadingScreen: false, persistState: false);
    }
}

public class MainMenuController : MonoBehaviour
{
    public async void OnPlayButtonClicked()
    {
        // Load Level1 with a loading screen and preserve any “Menu” state (if needed)
        await SceneHelper.LoadSceneAsync("Level1", LoadSceneMode.Single, showLoadingScreen: true, persistState: true);
    }

    public async void OnQuitButtonClicked()
    {
        Application.Quit();
    }
}

public class Level1Controller : MonoBehaviour, SceneHelper.ISceneStateSerializable
{
    private int _enemiesDefeated = 0;

    public string GetStateKey() => "Level1_EnemiesDefeated";

    public object SaveState() => _enemiesDefeated;

    public void LoadState(object state)
    {
        if (state is int n)
        {
            _enemiesDefeated = n;
            // Restore enemy‐count display in UI
        }
    }

    public async void OnFinishLevel()
    {
        // Player has completed level; unload Level1 and return to MainMenu
        await SceneHelper.UnloadSceneAsync("Level1", showTransition: true);
        // Optionally reload MainMenu if not still present in Single mode
        await SceneHelper.LoadSceneAsync("MainMenu", LoadSceneMode.Single, showLoadingScreen: false, persistState: false);
    }

    private void OnEnemyDefeated()
    {
        _enemiesDefeated++;
        // Update UI, etc.
    }
}
```

**Explanation**

1. `GameBootstrap.Start()` immediately calls `LoadSceneAsync("MainMenu", …)`, which fades to black then loads “MainMenu.”
2. When the user clicks “Play,” `MainMenuController` calls `LoadSceneAsync("Level1", showLoadingScreen: true, persistState: true)`—showing a progress bar while “Level1” loads. It also saves any `ISceneStateSerializable` state from “MainMenu” in `_sceneStates["MainMenu"]`, though that might be empty if no components implemented it.
3. If the user completes Level 1, `Level1Controller.OnFinishLevel()` calls `UnloadSceneAsync("Level1", showTransition: true)` to fade out and unload; then reload “MainMenu” in Single mode without a loading screen.

---

## 8. Best Practices & Tips

1. **Register `ISceneStateSerializable` Components**

   * Any GameObject whose runtime data you want to preserve across scene loads must implement `ISceneStateSerializable`.
   * Use a truly unique `GetStateKey()` (e.g., `"Enemy_" + enemyID`) so two objects don’t collide.

2. **Additive Scenes**

   * If you use `LoadSceneMode.Additive`, subsequent calls to `LoadSceneAsync` with `LoadSceneMode.Single` will clear `_activeScenes` and behave as expected. For additive unloading, call `UnloadSceneAsync("AddedSceneName", false)`.

3. **Loading Screen Customization**

   * The sample `CreateLoadingScreen()` builds a bare‐bones UI. You can modify it to add logos, animations, or adjust layout to suit your game.
   * If using DOTween, you might replace the manual `while (elapsed < duration)` loops with `canvasGroup.DOFade(…)` calls.

4. **Avoid Overlapping Calls**

   * Calling `LoadSceneAsync` while another load is in progress might yield unpredictable UI or race conditions. You may want to disable UI buttons until the first call completes.

5. **Scene Dependencies**

   * If “Level2” depends on “Level1” (e.g., sharing data), load them additively (`LoadSceneAsync("Level1", Additive)`, then `LoadSceneAsync("Level2", Additive)`). Manage persistence by calling `SaveSceneState()` at appropriate times.

6. **Cleanup Fade or Loading Screen**

   * If your game tends to glitch, ensure that in every code path where a fade or loading‐screen is created, a corresponding `FadeFromBlack()` or `HideLoadingScreen()` will run. You may wrap those in `try/finally` if needed.

---

With this documentation and the method summary table, you now have:

* A clear understanding of scene‐load/unload lifecycles.
* How to show/hide loading screens and fade transitions.
* How to persist and restore arbitrary runtime data across scene changes via `ISceneStateSerializable`.
* A straightforward pattern to manage additive vs. single‐scene workflows.
