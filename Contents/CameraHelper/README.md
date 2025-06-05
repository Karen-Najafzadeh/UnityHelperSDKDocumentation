Below is a detailed walkthrough of **CameraHelper**, a static utility class built to simplify common camera tasks in Unity, including camera shakes, transitions between virtual cameras, screen-to-world conversions, and viewport checks. Each region of the code is explained in depth, followed by realistic usage examples you can plug into your project.

---

## Overview

**CameraHelper** contains:

1. **Initialization & Registration**

   * Caches references to the main `Camera` and its attached `CinemachineBrain`.
   * Provides a method to register multiple `CinemachineVirtualCamera` instances under string IDs.

2. **Camera Shake (“Trauma”-Based)**

   * A “trauma” system that you can add to over time; the camera’s local position is randomly offset according to trauma² combined with user-configurable amplitude and decay settings.

3. **Camera Transitions (Cinemachine Integration)**

   * A simple priority-based switch between registered virtual cameras—set all others to low priority and bump up the target to a high priority, causing Cinemachine to blend.

4. **Screen↔World Conversions**

   * Converts a 2D screen point (pixels) to a 3D world position at a given Z-depth.
   * Converts a 3D world point back into screen-space coordinates.

5. **Viewport Management**

   * Checks whether a 3D world point lies within the camera’s viewport (i.e., on-screen).

Below is a section-by-section explanation.

---

## 1. Fields & Caches

```csharp
private static Camera _mainCamera;
private static CinemachineBrain _brain;
private static readonly Dictionary<string, CinemachineVirtualCamera> _virtualCameras 
    = new Dictionary<string, CinemachineVirtualCamera>();

// Shake parameters
private static float _trauma;
private static float _traumaDecay = 1.5f;
private static float _shakeFrequency = 25f;    // (unused in code as written, but could drive Perlin noise)
private static float _shakeAmplitude = 1f;
```

* **`_mainCamera`**

  * A cached reference to `Camera.main`. If there is no camera tagged “MainCamera,” subsequent calls print an error.

* **`_brain`**

  * If the main camera has a `CinemachineBrain` (auto-added when you place a `CinemachineVirtualCamera` in the scene), this caches it so you can later tweak blend styles or other Cinemachine settings if needed.

* **`_virtualCameras`**

  * A dictionary mapping string IDs → `CinemachineVirtualCamera` instances.
  * You register each v-cam under a unique key. Later, calling `TransitionToCamera("BattleCam")` simply sets that v-cam’s priority higher than the rest, and Cinemachine handles the blending.

* **Shake parameters**

  * **`_trauma`** (0..1) indicates how “shaky” the camera should be. The actual offset is computed as `shake = _trauma²`.
  * **`_traumaDecay`** controls how quickly `_trauma` decreases per second (so the shake fades out).
  * **`_shakeFrequency`** and **`_shakeAmplitude`** can be used to modulate the shake effect (e.g., if you implement noise-based shake). In this code, only `shakeAmplitude` is used to scale the random offset.

---

## 2. Initialization

### 2.1 Method: `public static void Initialize()`

```csharp
public static void Initialize()
{
    _mainCamera = Camera.main;
    _brain = _mainCamera?.GetComponent<CinemachineBrain>();
    if (!_mainCamera)
        Debug.LogError("No main camera found in scene");
}
```

**What It Does:**

1. **Caches the Main Camera**

   * Retrieves `Camera.main` (the first camera tagged “MainCamera” in the scene) and stores it in `_mainCamera`.
   * If there is no main camera, logs an error.

2. **Caches the CinemachineBrain (If Present)**

   * If `_mainCamera` is not null, tries to `GetComponent<CinemachineBrain>()` on it. This component is automatically added by Unity once you place any `CinemachineVirtualCamera` in the scene. Caching it lets you later query or override blending parameters at runtime if needed.

**When to Call It:**

* Ideally, call **once** at game startup—e.g., in an “AudioManager”-style initializer MonoBehaviour’s `Awake()` method, or on the first script that needs camera utilities.

**Example Usage:**

```csharp
public class CameraBootstrap : MonoBehaviour
{
    private void Awake()
    {
        CameraHelper.Initialize();
    }
}
```

* Attach `CameraBootstrap` to any GameObject in your initial scene.
* After that, you can safely call other `CameraHelper` methods without worrying about `_mainCamera` being null.

---

### 2.2 Method: `public static void RegisterVirtualCamera(string id, CinemachineVirtualCamera vcam)`

```csharp
public static void RegisterVirtualCamera(string id, CinemachineVirtualCamera vcam)
{
    _virtualCameras[id] = vcam;
}
```

**What It Does:**

* Adds (or replaces) an entry in the `_virtualCameras` dictionary, keyed by your chosen string `id`, pointing to the `vcam` instance passed in.
* Later, calling `TransitionToCamera(id)` uses that dictionary to find the v-cam.

**Important Notes:**

* **No null check** is performed here—if `vcam` is null, you’d end up storing a null reference. You might want to guard against that in your own code (e.g., `if (vcam != null) RegisterVirtualCamera(...)`).

**Example Usage:**

Imagine you have two v-cams in your scene:

1. A “FreeLookCam” for exploration.
2. A “BattleCam” for combat mode.

In a script called `VCamManager`:

```csharp
using Cinemachine;
using UnityEngine;

public class VCamManager : MonoBehaviour
{
    [SerializeField] private CinemachineVirtualCamera _freeLookCam;
    [SerializeField] private CinemachineVirtualCamera _battleCam;

    private void Awake()
    {
        CameraHelper.Initialize();

        // Register each v-cam under a unique string key
        CameraHelper.RegisterVirtualCamera("FreeLook", _freeLookCam);
        CameraHelper.RegisterVirtualCamera("Battle", _battleCam);
        
        // Optionally, set all to default priority here
        _freeLookCam.Priority = 5;
        _battleCam.Priority = 0; 
    }
}
```

* Later in gameplay, you can call `CameraHelper.TransitionToCamera("Battle")` to smoothly switch to the battle cam.

---

## 3. Camera Shake

### 3.1 Method: `public static void AddTrauma(float amount)`

```csharp
/// <summary>
/// Add trauma to trigger camera shake
/// </summary>
public static void AddTrauma(float amount)
{
    _trauma = Mathf.Clamp01(_trauma + amount);
}
```

**What It Does:**

* Increases the private `_trauma` value by `amount`, then clamps the result to the range `[0, 1]`.
* Calling `AddTrauma(0.3f)` when `_trauma` was `0.5f` yields `0.8f`. Passing `AddTrauma(1f)` when `_trauma` is `0.5f` clamps to `1f`.

**How to Use It:**

* Whenever something “shaky” happens—explosion, heavy landing, earthquake—you call `CameraHelper.AddTrauma(X)`, where `X` is how intense the shake should be (e.g., `0.2f` for small, `0.8f` for large).
* The actual random offset is computed in `UpdateShake()` based on `_trauma²`.

**Example:**

```csharp
public class Explosion : MonoBehaviour
{
    [SerializeField] private float _traumaAmount = 0.6f;

    private void Start()
    {
        // When explosion spawns, add trauma
        CameraHelper.AddTrauma(_traumaAmount);
    }
}
```

* This ensures that if multiple explosions occur in quick succession, trauma accumulates up to 1.0 (maximum shake).

---

### 3.2 Method: `public static void UpdateShake()`

```csharp
/// <summary>
/// Update camera shake. Call this from Update()
/// </summary>
public static void UpdateShake()
{
    if (_trauma <= 0) 
        return;

    float shake = Mathf.Pow(_trauma, 2);
    _mainCamera.transform.localPosition = UnityEngine.Random.insideUnitSphere * shake * _shakeAmplitude;
    _trauma = Mathf.Max(0, _trauma - _traumaDecay * Time.deltaTime);
}
```

**What It Does:**

1. **Check if There Is Any Trauma**

   * If `_trauma` is zero or below, do nothing.

2. **Compute Shake Amount**

   * `shake = _trauma²` (so small trauma yields very small offsets, while high trauma yields stronger shake).
   * Multiply by `_shakeAmplitude` to scale the effect.

3. **Apply Random Offset**

   * Sets the camera’s **localPosition** to a random point inside a unit sphere (uniformly distributed within radius 1), scaled by `shake * amplitude`.
   * Because it uses `transform.localPosition`, it temporarily scrambles the camera’s position relative to its parent. If you rely on the camera being at a specific world location, be cautious—shake offsets are relative.

4. **Decay Trauma Over Time**

   * Subtracts `_traumaDecay * Time.deltaTime` from `_trauma`—so if `_traumaDecay` is `1.5`, and frame took `0.016s` (\~60 FPS), it subtracts `0.024` each frame.
   * Ensures that `_trauma` eventually falls to zero, ending the shake.

**How to Use It:**

* You must call `CameraHelper.UpdateShake()` in **every** frame—typically inside a MonoBehaviour’s `Update()` method:

```csharp
public class CameraShakeController : MonoBehaviour
{
    private void Update()
    {
        CameraHelper.UpdateShake();
    }
}
```

* Without this per-frame call, trauma never decays and no random offsets occur.

**Example in Context:**

```csharp
public class PlayerHurt : MonoBehaviour
{
    [SerializeField] private float _hurtTrauma = 0.4f;

    private void Update()
    {
        // Always update shake each frame
        CameraHelper.UpdateShake();

        if (Input.GetKeyDown(KeyCode.H))
        {
            // Simulate player taking damage
            CameraHelper.AddTrauma(_hurtTrauma);
        }
    }
}
```

* Pressing “H” adds 0.4 trauma—camera then jitters for roughly `_hurtTrauma / _traumaDecay` seconds (≈0.266s) but decaying quadratically for a more “explosive” feel.

---

## 4. Camera Transitions

### Method: `public static void TransitionToCamera(string cameraName, float blendTime = 1f)`

```csharp
/// <summary>
/// Smoothly transition between virtual cameras
/// </summary>
public static void TransitionToCamera(string cameraName, float blendTime = 1f)
{
    if (_virtualCameras.TryGetValue(cameraName, out var vcam))
    {
        foreach (var cam in _virtualCameras.Values)
            cam.Priority = 0;
        vcam.Priority = 10;
    }
}
```

**What It Does:**

1. **Lookup the Target v-Cam**

   * Attempts to fetch a `CinemachineVirtualCamera` from `_virtualCameras` dictionary by key `cameraName`. If not found, does nothing.

2. **Lower All Other Priorities**

   * Iterates through every `CinemachineVirtualCamera` in `_virtualCameras.Values` and sets its `Priority = 0`—effectively telling Cinemachine that they should not be active.

3. **Raise the Target Camera’s Priority**

   * Sets `vcam.Priority = 10` for the fetched virtual camera. Since its priority is highest, Cinemachine will blend to that one.

4. **Blend Time**

   * Although the method signature includes `blendTime`, the code as written does not use it. In practice, you would set the blend duration in your `CinemachineBrain` or in the v-cam’s “Blend” settings so that whenever priorities switch, Cinemachine uses that time for the crossfade. The method could be extended to modify `CinemachineBrain.m_DefaultBlend` at runtime.

**How to Use It:**

* Register all your v-cams at startup under distinct keys.
* When you want to switch, call `CameraHelper.TransitionToCamera("YourCamID")`. Cinemachine will handle the smooth interpolation between the old active v-cam and the newly prioritized one.

**Example:**

Continuing from the previous registration example:

```csharp
public class GameplayCAM : MonoBehaviour
{
    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.Alpha1))
        {
            // Switch to “FreeLook” cam instantly
            CameraHelper.TransitionToCamera("FreeLook", blendTime: 0.5f);
        }
        else if (Input.GetKeyDown(KeyCode.Alpha2))
        {
            // Switch to “Battle” cam
            CameraHelper.TransitionToCamera("Battle", blendTime: 0.5f);
        }
    }
}
```

* Make sure each v-cam’s inspector settings define a 0.5s custom blend (in `CinemachineVirtualCamera → Blend`), or set the `CinemachineBrain.m_DefaultBlend` at runtime if you want to customize per-call.

---

## 5. Screen Conversions

Sometimes you need to convert between screen-space (pixels) and world-space coordinates, for example when placing UI elements that track world objects.

### 5.1 Method: `public static Vector3 ScreenToWorld(Vector2 screenPoint, float z = 10f)`

```csharp
/// <summary>
/// Convert screen point to world position
/// </summary>
public static Vector3 ScreenToWorld(Vector2 screenPoint, float z = 10f)
{
    if (!_mainCamera) 
        Initialize();

    Vector3 point = new Vector3(screenPoint.x, screenPoint.y, z);
    return _mainCamera.ScreenToWorldPoint(point);
}
```

**What It Does:**

1. **Lazy Initialization**

   * If `_mainCamera` is null (i.e., `Initialize()` was not called yet), calls `Initialize()` to cache `Camera.main`.

2. **Construct 3D Point**

   * Creates a `Vector3` out of `screenPoint.x`, `screenPoint.y`, and the provided `z` (distance from the camera along its forward axis).
   * For example, if `z=10`, then the returned world point will lie on the plane that is 10 units in front of the camera.

3. **Call Unity’s API**

   * Uses `Camera.ScreenToWorldPoint(Vector3)` to convert that screen coordinate to a 3D world position.

**When to Use It:**

* When you have a mouse click at `(Input.mousePosition.x, Input.mousePosition.y)` and you want to instantiate a projectile at a certain depth.
* When aligning 3D objects to UI elements or vice versa.

**Examples:**

1. **Spawn a 3D Object Where the Player Clicks (At Z = 0 world plane):**

   ```csharp
   public class ClickSpawner : MonoBehaviour
   {
       [SerializeField] private GameObject _prefab;

       private void Update()
       {
           if (Input.GetMouseButtonDown(0))
           {
               Vector2 screenPos = Input.mousePosition;
               // If your camera is orthographic or if you want Z=0, set z = -_mainCamera.transform.position.z
               Vector3 worldPos = CameraHelper.ScreenToWorld(screenPos, 10f);
               Instantiate(_prefab, worldPos, Quaternion.identity);
           }
       }
   }
   ```

   * If your camera is at `Z = -10` looking toward +Z, and you pass `z = 10f`, then the worldPos ends up on `Z = 0` plane. Adjust `z` to match your camera’s near-clip logic.

2. **UI Tooltip Following a 3D Character:**

   ```csharp
   public class TooltipFollower : MonoBehaviour
   {
       [SerializeField] private RectTransform _tooltipUI;
       [SerializeField] private Transform _targetWorldObject;

       private void LateUpdate()
       {
           Vector3 worldPos = _targetWorldObject.position + Vector3.up * 2f; 
           // Project that 3D point into screen space
           Vector2 screenPos = CameraHelper.WorldToScreen(worldPos);
           // Convert screenPos to Canvas space (assumes Screen Space - Overlay)
           _tooltipUI.position = screenPos;
       }
   }
   ```

   * This uses `WorldToScreen` from the next section to attach a UI element (tooltip) to a world object.

---

### 5.2 Method: `public static Vector2 WorldToScreen(Vector3 worldPoint)`

```csharp
/// <summary>
/// Convert world position to screen point
/// </summary>
public static Vector2 WorldToScreen(Vector3 worldPoint)
{
    if (!_mainCamera) 
        Initialize();

    return _mainCamera.WorldToScreenPoint(worldPoint);
}
```

**What It Does:**

1. **Lazy Initialization**

   * If `_mainCamera` is null, calls `Initialize()`.

2. **Call Unity’s API**

   * Uses `Camera.WorldToScreenPoint(Vector3)` to project the 3D `worldPoint` into 2D screen space (pixels). Returns a `Vector2` with `(x, y)` screen coordinates.

**When to Use It:**

* As shown above, for UI elements that must track 3D objects.
* If you need to highlight a world-space collider by drawing a 2D marker at its screen location.

**Real-World Example:**

```csharp
public class HealthBar : MonoBehaviour
{
    [SerializeField] private RectTransform _healthBarUI;
    [SerializeField] private Transform _enemy;

    private void Update()
    {
        Vector3 headPosition = _enemy.position + Vector3.up * 1.8f;
        Vector2 screenPos = CameraHelper.WorldToScreen(headPosition);
        _healthBarUI.position = screenPos;
    }
}
```

* The health bar UI will always hover above the enemy’s head, even as the enemy moves in 3D.

---

## 6. Viewport Management

### Method: `public static bool IsInViewport(Vector3 worldPoint)`

```csharp
/// <summary>
/// Check if a world position is visible in the camera's viewport
/// </summary>
public static bool IsInViewport(Vector3 worldPoint)
{
    if (!_mainCamera) 
        Initialize();

    Vector3 viewportPoint = _mainCamera.WorldToViewportPoint(worldPoint);
    return viewportPoint.x >= 0 && viewportPoint.x <= 1 && 
           viewportPoint.y >= 0 && viewportPoint.y <= 1 && 
           viewportPoint.z > 0;
}
```

**What It Does:**

1. **Lazy Initialization**

   * If `_mainCamera` is null, calls `Initialize()`.

2. **Compute Viewport Coordinates**

   * `Camera.WorldToViewportPoint(Vector3)` returns a `Vector3` where

     * `(x, y)` lie in `[0..1]` if the point is on-screen horizontally and vertically.
     * `z > 0` if the point is in front of the camera (i.e., not behind).

3. **Return True/False**

   * Returns `true` if `0 ≤ x ≤ 1`, `0 ≤ y ≤ 1`, and `z > 0`.
   * If any of those conditions fail, the point lies off-screen or behind the camera.

**When to Use It:**

* To decide if you should spawn or despawn effects (e.g., only show an indicator if the enemy is on screen).
* To test line-of-sight—for example, only show UI icons for visible targets.

**Examples:**

1. **Spawn Particle Effect Only If an Object Is On-Screen:**

   ```csharp
   public class ImpactEffect : MonoBehaviour
   {
       [SerializeField] private GameObject _particlePrefab;

       public void SpawnAt(Vector3 worldPos)
       {
           if (CameraHelper.IsInViewport(worldPos))
           {
               Instantiate(_particlePrefab, worldPos, Quaternion.identity);
           }
       }
   }
   ```

   * This guarantees you don’t waste particles behind the camera or far off-screen.

2. **UI Indicator for Off-Screen Enemies:**

   ```csharp
   public class OffscreenIndicator : MonoBehaviour
   {
       [SerializeField] private RectTransform _indicatorUI; // e.g., arrow icon
       [SerializeField] private Transform _enemy;

       private void Update()
       {
           Vector3 enemyPos = _enemy.position;
           bool onScreen = CameraHelper.IsInViewport(enemyPos);

           if (!onScreen)
           {
               _indicatorUI.gameObject.SetActive(true);
               // Compute direction toward enemy
               Vector3 screenCenter = new Vector3(Screen.width / 2f, Screen.height / 2f, 0f);
               Vector2 enemyScreen = CameraHelper.WorldToScreen(enemyPos);
               Vector2 direction = (enemyScreen - (Vector2)screenCenter).normalized;
               // Place indicator at screen edge in that direction
               float edgePadding = 50f;
               Vector2 indicatorPos = screenCenter + direction * (screenCenter.magnitude - edgePadding);
               _indicatorUI.position = indicatorPos;
               float angle = Mathf.Atan2(direction.y, direction.x) * Mathf.Rad2Deg;
               _indicatorUI.rotation = Quaternion.Euler(0, 0, angle - 90f);
           }
           else
           {
               _indicatorUI.gameObject.SetActive(false);
           }
       }
   }
   ```

   * If the enemy moves off-screen, the arrow points at it from the closest edge; otherwise the arrow is hidden.

---

## 7. Putting It All Together: Example Project Snippet

Below is a small sample MonoBehaviour demonstrating how to integrate all of **CameraHelper**’s functionality in one place:

```csharp
using UnityEngine;
using Cinemachine;

public class CameraDemo : MonoBehaviour
{
    [Header("Virtual Cameras")]
    [SerializeField] private CinemachineVirtualCamera _freeCam;
    [SerializeField] private CinemachineVirtualCamera _battleCam;

    [Header("Shake Settings")]
    [SerializeField] private float _smallTrauma = 0.2f;
    [SerializeField] private float _bigTrauma = 0.7f;

    private void Start()
    {
        // 1. Initialize helper and register v-cams
        CameraHelper.Initialize();
        CameraHelper.RegisterVirtualCamera("Free", _freeCam);
        CameraHelper.RegisterVirtualCamera("Battle", _battleCam);

        // Ensure initial camera is “Free”
        CameraHelper.TransitionToCamera("Free");
    }

    private void Update()
    {
        // 2. Update the shake each frame
        CameraHelper.UpdateShake();

        // 3. Toggle between Free and Battle cams
        if (Input.GetKeyDown(KeyCode.F))
        {
            CameraHelper.TransitionToCamera("Free");
        }
        else if (Input.GetKeyDown(KeyCode.B))
        {
            CameraHelper.TransitionToCamera("Battle");
        }

        // 4. Trigger small or big shakes
        if (Input.GetKeyDown(KeyCode.Alpha1))
        {
            CameraHelper.AddTrauma(_smallTrauma);
        }
        else if (Input.GetKeyDown(KeyCode.Alpha2))
        {
            CameraHelper.AddTrauma(_bigTrauma);
        }

        // 5. Spawn an object at mouse position if you click
        if (Input.GetMouseButtonDown(0))
        {
            Vector2 screenPos = Input.mousePosition;
            Vector3 worldPos = CameraHelper.ScreenToWorld(screenPos, z: 10f);
            Debug.Log($"Clicked world position: {worldPos}");
            // e.g., Instantiate(yourPrefab, worldPos, Quaternion.identity);
        }

        // 6. Check if a specific world object is on-screen
        var enemy = GameObject.FindWithTag("Enemy");
        if (enemy != null)
        {
            bool isVisible = CameraHelper.IsInViewport(enemy.transform.position);
            Debug.Log($"Enemy on-screen? {isVisible}");
        }
    }
}
```

* **Step 1**: In `Start()`, we call `CameraHelper.Initialize()` and `RegisterVirtualCamera()` for each v-cam. We also immediately switch to “Free” cam by calling `TransitionToCamera("Free")`.
* **Step 2**: In `Update()`, we call `UpdateShake()` so that any previously-added trauma will animate as camera offsets.
* **Step 3**: Press “F” or “B” to switch to “Free” or “Battle” cam, respectively. Cinemachine will automatically blend over the configured blend time.
* **Step 4**: Press “1” to add a small shake, press “2” to add a big shake.
* **Step 5**: Left-click anywhere on screen to log the world position at `z = 10` in front of the camera.
* **Step 6**: Continuously check if an `Enemy` GameObject is in the current camera’s viewport.

---

## 8. Best Practices & Tips

1. **Always Call `Initialize()` Early**

   * If you forget to initialize, `_mainCamera` stays null and most methods will call `Initialize()` lazily—logging errors if no camera is found. For best performance, do it once at startup.

2. **Camera Shake on a Separate MonoBehaviour**

   * Put `CameraHelper.UpdateShake()` inside a dedicated “CameraShakeController” MonoBehaviour’s `LateUpdate()` to ensure shake happens after all other transforms update. For example:

     ```csharp
     public class CameraShakeController : MonoBehaviour
     {
         private void LateUpdate()
         {
             CameraHelper.UpdateShake();
         }
     }
     ```

     * This ensures that Cine-machine’s blending and following logic runs first, then you apply the random offset.

3. **Blend Time Configuration**

   * The `TransitionToCamera` method includes a `blendTime` parameter, but the code does not actually use it. To control blend duration at runtime, you can call:

     ```csharp
     _brain.m_DefaultBlend = new CinemachineBlendDefinition(
         CinemachineBlendDefinition.Style.EaseInOut, 
         blendTime
     );
     ```

     * For example, modify `TransitionToCamera` to set `_brain.m_DefaultBlend.BlendTime = blendTime` before adjusting priorities.

4. **Trauma Decay & Amplitude Tweaks**

   * Adjust `_traumaDecay` (units per second) and `_shakeAmplitude` to suit your game’s feel. A high decay means the screen snaps back quickly; a low decay yields a slower fade.
   * You can also replace `Random.insideUnitSphere` with Perlin-noise-based shake for smoother, more natural jostling.

5. **Viewport Checks for Culling**

   * Use `IsInViewport` to pre-filter objects for AI, particle spawning, or other visual effects. If an object is off-screen, you might disable its AI updates or skip expensive computations.

6. **Multiple Cameras & Layers**

   * If you have multiple camera setups (e.g., split-screen), `CameraHelper` only references `Camera.main`. You can extend it to accept multiple cameras by storing them in a dictionary keyed by index or role.

7. **ScreenToWorld Z Value**

   * The `z` parameter in `ScreenToWorld(screenPoint, z)` is the distance from the **camera’s near plane** (not necessarily world origin). If your camera is at `z = -10` and pointing toward `z = 0`, passing `z = 10` yields a world depth of `z = 0`. Adjust accordingly if your camera has a different orientation.

---

## 9. Summary of Public API

| Method Signature                                                                     | Purpose                                                                                                                    |
| ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `public static void Initialize()`                                                    | Cache `Camera.main` and its `CinemachineBrain`, log an error if none found.                                                |
| `public static void RegisterVirtualCamera(string id, CinemachineVirtualCamera vcam)` | Add a v-cam to the internal dictionary under `id` so you can transition to it later.                                       |
| `public static void AddTrauma(float amount)`                                         | Increase the “trauma” shake value (clamped to \[0,1]). Call repeatedly for stacked shakes.                                 |
| `public static void UpdateShake()`                                                   | If `_trauma > 0`, apply a random localPosition offset of magnitude `trauma² × amplitude`, then decrement trauma over time. |
| `public static void TransitionToCamera(string cameraName, float blendTime = 1f)`     | Lower all registered v-cams’ priority to 0, raise the chosen cam’s priority to a high number so Cinemachine blends to it.  |
| `public static Vector3 ScreenToWorld(Vector2 screenPoint, float z = 10f)`            | Convert a 2D screen pixel coordinate to a 3D world point at depth `z` (distance from camera’s near plane).                 |
| `public static Vector2 WorldToScreen(Vector3 worldPoint)`                            | Convert a 3D world position into 2D screen coordinates (pixels) using `Camera.main`.                                       |
| `public static bool IsInViewport(Vector3 worldPoint)`                                | Return true if the given world point has viewport x,y in \[0,1] and z > 0 (i.e., in front of camera and on-screen).        |

---

## 10. Conclusion

**CameraHelper** streamlines several common camera tasks:

* **Shake Effects**: Build more impactful moments (explosions, heavy landings) by adding “trauma” that decays over time.
* **Seamless Virtual Camera Switching**: Register multiple Cinemachine v-cams and call `TransitionToCamera("SomeCam")`—Cinemachine handles the rest.
* **Coordinate Conversions**: Quickly map between screen coordinates (pixels) and world coordinates (3D) with a specified depth, enabling clickable world spawning or UI-tracking.
* **Viewport Checks**: Determine if a given world point is currently visible, allowing you to cull or highlight objects conditionally.

By following the usage patterns and examples provided above, you can integrate this helper into any Unity project that uses Cinemachine and requires dynamic camera control, shakes, or viewport-based logic—without rewriting the same boilerplate in multiple places.
