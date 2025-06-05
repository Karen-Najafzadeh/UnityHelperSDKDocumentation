Below is a comprehensive reference guide for **InputHelper**—a static utility class that unifies Unity’s new Input System, legacy `Input` API, touch‐gesture detection, and custom key bindings. It also lays the groundwork for input recording and playback. Wherever you see “example,” imagine placing that code inside a MonoBehaviour method (e.g. `Update()` or in response to some event).

---

## Table of Contents

1. [Overview & Purpose](#overview--purpose)
2. [Cross‐Platform Input Integration](#cross‐platform‐input‐integration)

   * 2.1. Registering and Unregistering Input Actions (New Input System)
3. [Touch Gesture Detection](#touch-gesture-detection)

   * 3.1. `DetectSwipe()`
   * 3.2. `DetectTap(out Vector2 position)`
   * 3.3. `DetectDoubleTap(out Vector2 position)`
4. [Custom Key Bindings](#custom-key-bindings)

   * 4.1. Registering a Key Binding
   * 4.2. Checking if a Key Binding Is Active
   * 4.3. The `KeyBinding` Helper Type
5. [Helper Types](#helper-types)

   * 5.1. `SwipeDirection` enum
   * 5.2. `KeyBinding` class
6. [Real-World Usage Examples](#real-world-usage-examples)

   * 6.1. Bridging the New Input System with Legacy Input
   * 6.2. Detecting Swipe Directions on Mobile
   * 6.3. Detecting Single and Double Taps
   * 6.4. Registering and Checking Custom Key Bindings
   * 6.5. Combining Touch & Keyboard Input in a Single Helper
7. [Best Practices & Tips](#best-practices--tips)
8. [Summary of Public API](#summary-of-public-api)

---

## 1. Overview & Purpose

```csharp
public static class InputHelper
{
    // … internal fields and methods …
}
```

**InputHelper** is designed to:

* Work with both Unity’s **New Input System** (`UnityEngine.InputSystem`) and the legacy `Input` API (`UnityEngine.Input`).
* Detect common **touch gestures**—swipes, single taps, and double taps—without you having to rewrite boilerplate.
* Store and evaluate **custom key bindings** (e.g., “ActionJump” = Space + Shift).
* (Optionally) Record and replay input sequences (framework in place, although actual replay logic is left to you).
* Offer out‐of‐the‐box **cross‐platform** input handling: if touch is unavailable, swipe/tap detection simply returns “none.”

Use **InputHelper** when you want to centralize all your input logic—on both mobile (touch) and desktop (keyboard/controller)—within a single static helper.

---

## 2. Cross-Platform Input Integration

InputHelper wraps two major categories:

1. **Unity’s New Input System** (`InputAction`, `InputAction.CallbackContext`) for devices like gamepads, keyboards, mice, touchscreens.
2. Legacy `Input` calls (`Input.GetKey`, `Input.touchCount`, `Input.GetTouch`, etc.)—especially for **touch gestures**.

### 2.1 Registering and Unregistering Input Actions (New Input System)

> **Note**: To use these methods, make sure your project has the **Input System** package enabled (Edit ▶ Project Settings ▶ Player ▶ Active Input Handling ▶ “Both” or “Input System Package (New)”). Then build `InputAction` assets in the editor or via code.

```csharp
/// <summary>
/// Register a new input action with a callback. If an action with the same name
/// was already registered, it is first disabled, then replaced.
/// </summary>
public static void RegisterAction(
    string name,
    InputAction action,
    Action<InputAction.CallbackContext> callback)
{
    if (_actions.ContainsKey(name))
        _actions[name].Disable();

    action.Enable();
    action.performed += callback;
    _actions[name] = action;
}

/// <summary>
/// Unregister an input action previously registered via RegisterAction.
/// This disables the action and removes it from the internal dictionary.
/// </summary>
public static void UnregisterAction(string name)
{
    if (_actions.TryGetValue(name, out var action))
    {
        action.Disable();
        _actions.Remove(name);
    }
}
```

* **Purpose**

  * `RegisterAction(name, action, callback)`: Enables the passed‐in `InputAction`, wires its `.performed` event to your callback, and stores it under the string key `name`. If you register another action under the same `name`, the previous one is disabled automatically.
  * `UnregisterAction(name)`: Disables and removes that action—useful when unloading a menu or scene so that callbacks stop firing.

* **Typical Workflow**

  1. Create an `InputAction` in code or in a `.inputactions` asset.
  2. Call `InputHelper.RegisterAction("Jump", jumpAction, ctx => OnJump(ctx));`
  3. Later, if that UI or gameplay context is torn down, call `InputHelper.UnregisterAction("Jump");`

* **Example**

  ```csharp
  using UnityEngine;
  using UnityEngine.InputSystem;

  public class PlayerController : MonoBehaviour
  {
      public InputActionAsset actionAsset; // assigned via Inspector
      private InputAction _moveAction;
      private InputAction _jumpAction;

      private void Awake()
      {
          // Assume actionAsset has a map called "Player" with actions "Move" and "Jump"
          var playerMap = actionAsset.FindActionMap("Player");
          _moveAction = playerMap.FindAction("Move");
          _jumpAction = playerMap.FindAction("Jump");

          // Register movement and jump callbacks:
          InputHelper.RegisterAction("Move", _moveAction, OnMovePerformed);
          InputHelper.RegisterAction("Jump", _jumpAction, OnJumpPerformed);
      }

      private void OnDestroy()
      {
          InputHelper.UnregisterAction("Move");
          InputHelper.UnregisterAction("Jump");
      }

      private void OnMovePerformed(InputAction.CallbackContext ctx)
      {
          Vector2 input = ctx.ReadValue<Vector2>();
          // e.g. update character velocity
          Debug.Log($"Moving: {input}");
      }

      private void OnJumpPerformed(InputAction.CallbackContext ctx)
      {
          if (ctx.performed)
          {
              // Trigger a jump
              Debug.Log("Jump pressed!");
          }
      }
  }
  ```

---

## 3. Touch Gesture Detection

InputHelper can detect simple touch gestures on mobile (or any device that supports `Input.touchSupported`). Touch detection falls back gracefully (it returns “None” or `false`) on platforms without a touchscreen.

Internally, three thresholds are used (you can override them in code if needed):

* **SwipeThreshold** (in pixels): minimum distance between `TouchPhase.Began` and `TouchPhase.Ended` to qualify as a swipe.
* **TapThreshold** (in seconds): maximum duration of a single touch (i.e. from `TouchPhase.Began` to `TouchPhase.Ended`) to be considered a tap.
* **DoubleTapThreshold** (in seconds): maximum time between two consecutive taps to be considered a double‐tap.

```csharp
private static readonly float SwipeThreshold = 50f;
private static readonly float TapThreshold = 0.2f;
private static readonly float DoubleTapThreshold = 0.3f;

private static Vector2 _touchStartPos;
private static float _touchStartTime;
private static float _lastTapTime;
```

### 3.1 `public static SwipeDirection DetectSwipe()`

```csharp
public static SwipeDirection DetectSwipe()
{
    if (!Input.touchSupported) 
        return SwipeDirection.None;

    if (Input.touchCount > 0)
    {
        Touch touch = Input.GetTouch(0);

        switch (touch.phase)
        {
            case UnityEngine.TouchPhase.Began:
                _touchStartPos = touch.position;
                _touchStartTime = Time.time;
                break;

            case UnityEngine.TouchPhase.Ended:
                float duration = Time.time - _touchStartTime;
                Vector2 delta = touch.position - _touchStartPos;

                if (delta.magnitude > SwipeThreshold)
                {
                    float angle = Mathf.Atan2(delta.y, delta.x) * Mathf.Rad2Deg;
                    if (angle > -45f && angle <= 45f)
                        return SwipeDirection.Right;
                    if (angle > 45f && angle <= 135f)
                        return SwipeDirection.Up;
                    if (angle > 135f || angle <= -135f)
                        return SwipeDirection.Left;
                    if (angle > -135f && angle <= -45f)
                        return SwipeDirection.Down;
                }
                break;
        }
    }

    return SwipeDirection.None;
}
```

* **What It Does**

  1. Returns `SwipeDirection.None` immediately if the device does not support touch.
  2. If there is at least one touch (`touchCount > 0`), it reads the first touch (`GetTouch(0)`).
  3. On `TouchPhase.Began`, it records the start position (`_touchStartPos`) and start time.
  4. On `TouchPhase.Ended`, it computes the difference vector `delta = endPos – startPos`. If `delta.magnitude` exceeds `SwipeThreshold`, it calculates `atan2(delta.y, delta.x)` to determine which of the four cardinal directions the swipe is closest to:

     * **Right**: angle between –45° and 45°
     * **Up**: angle between 45° and 135°
     * **Left**: angle > 135° or ≤ –135°
     * **Down**: angle between –135° and –45°
  5. Returns the corresponding `SwipeDirection`. If the swipe was too short or no touch ended, returns `SwipeDirection.None`.

* **Usage Example**

  ```csharp
  using UnityEngine;

  public class SwipeDetector : MonoBehaviour
  {
      void Update()
      {
          var dir = InputHelper.DetectSwipe();
          switch (dir)
          {
              case InputHelper.SwipeDirection.Up:
                  Debug.Log("Swiped Up");
                  break;
              case InputHelper.SwipeDirection.Down:
                  Debug.Log("Swiped Down");
                  break;
              case InputHelper.SwipeDirection.Left:
                  Debug.Log("Swiped Left");
                  break;
              case InputHelper.SwipeDirection.Right:
                  Debug.Log("Swiped Right");
                  break;
              default:
                  break;
          }
      }
  }
  ```

  * Attach this script to a GameObject and build to a mobile device. When you swipe your finger quickly more than 50 pixels in any direction, the corresponding message logs to the Console.

### 3.2 `public static bool DetectTap(out Vector2 position)`

```csharp
public static bool DetectTap(out Vector2 position)
{
    position = Vector2.zero;

    if (!Input.touchSupported)
        return false;

    if (Input.touchCount == 1)
    {
        Touch touch = Input.GetTouch(0);

        if (touch.phase == UnityEngine.TouchPhase.Began)
        {
            _touchStartTime = Time.time;
            _touchStartPos = touch.position;
        }

        if (touch.phase == UnityEngine.TouchPhase.Ended)
        {
            float duration = Time.time - _touchStartTime;
            if (duration < TapThreshold)
            {
                position = touch.position;
                return true;
            }
        }
    }

    return false;
}
```

* **What It Does**

  1. Returns `false` immediately if touch is not supported.
  2. If exactly one touch is present, on `TouchPhase.Began` it stores the start time (`_touchStartTime`) and position (`_touchStartPos`).
  3. When `TouchPhase.Ended` occurs, it calculates `duration = Time.time – _touchStartTime`.
  4. If `duration < TapThreshold` (0.2 seconds by default), it treats this as a tap, sets `position` to the final `touch.position`, and returns `true`. Otherwise, returns `false`.

* **Usage Example**

  ```csharp
  using UnityEngine;

  public class TapDetector : MonoBehaviour
  {
      void Update()
      {
          if (InputHelper.DetectTap(out Vector2 tapPosition))
          {
              Debug.Log($"Screen tapped at: {tapPosition}");
              // Convert to world coordinates if you need:
              Vector3 worldPoint = Camera.main.ScreenToWorldPoint(new Vector3(tapPosition.x, tapPosition.y, 10f));
              Debug.Log($"Tapped world point: {worldPoint}");
          }
      }
  }
  ```

  * This logs whenever the user touches and releases within 0.2s. If you hold your finger longer than 0.2s, it won’t count as a tap.

### 3.3 `public static bool DetectDoubleTap(out Vector2 position)`

```csharp
public static bool DetectDoubleTap(out Vector2 position)
{
    position = Vector2.zero;

    if (DetectTap(out position))
    {
        float timeSinceLastTap = Time.time - _lastTapTime;
        if (timeSinceLastTap < DoubleTapThreshold)
        {
            _lastTapTime = 0f;   // Reset so that a third tap won't be mistaken
            return true;
        }
        _lastTapTime = Time.time;
    }

    return false;
}
```

* **What It Does**

  1. Calls `DetectTap(out position)` internally. If that returns `false`, this returns `false`.
  2. If there *was* a tap, calculates `timeSinceLastTap = Time.time – _lastTapTime`.
  3. If `timeSinceLastTap < DoubleTapThreshold` (0.3 seconds by default), resets `_lastTapTime` to zero and returns `true`—indicating a valid double‐tap.
  4. Otherwise, sets `_lastTapTime = Time.time` and returns `false` (we treat this single tap as the “first tap” of a potential double‐tap).

* **Usage Example**

  ```csharp
  using UnityEngine;

  public class DoubleTapDetector : MonoBehaviour
  {
      void Update()
      {
          if (InputHelper.DetectDoubleTap(out Vector2 pos))
          {
              Debug.Log($"Double‐tap detected at {pos}");
              // E.g., zoom in on that position, or toggle fullscreen UI
          }
      }
  }
  ```

  * If the user taps twice in rapid succession (each touch lasting < 0.2s, separated by less than 0.3s), you get one double‐tap event.

---

## 4. Custom Key Bindings

Besides touch, InputHelper offers a simple way to register your own **named key bindings**—e.g. “FireWeapon” might map to Ctrl+F or a gamepad button. You can register multiple `KeyCode` modifiers plus a primary key.

```csharp
private static readonly Dictionary<string, KeyBinding> _keyBindings = new Dictionary<string, KeyBinding>();
```

### 4.1 Registering a Key Binding

```csharp
/// <summary>
/// Register a new key binding under the given actionName. If one already exists,
/// it is simply overwritten.
/// </summary>
public static void RegisterKeyBinding(string actionName, KeyBinding binding)
{
    _keyBindings[actionName] = binding;
}
```

* **Purpose**:
  Store a custom mapping from a string key (e.g. `"PauseGame"`) to a `KeyBinding` object describing which `KeyCode` plus which modifiers must be held.

* **Example**:

  ```csharp
  // Bind “ToggleMap” to Ctrl + M
  var toggleMapBinding = new InputHelper.KeyBinding
  {
      PrimaryKey = KeyCode.M,
      ModifierKeys = new KeyCode[] { KeyCode.LeftControl, KeyCode.RightControl }
  };
  InputHelper.RegisterKeyBinding("ToggleMap", toggleMapBinding);
  ```

  * Now “ToggleMap” is associated with “press M while holding either left‐ or right‐Ctrl.”

### 4.2 Checking if a Key Binding Is Active

```csharp
/// <summary>
/// Returns true if the key‐binding registered under actionName is currently pressed
/// (primary key plus all modifiers). Returns false if not registered or not pressed.
/// </summary>
public static bool IsBindingActive(string actionName)
{
    if (_keyBindings.TryGetValue(actionName, out var binding))
        return binding.IsActive();

    return false;
}
```

* **Purpose**:
  At any time (e.g. inside `Update()`), call `InputHelper.IsBindingActive("ToggleMap")`. It returns `true` only if the `PrimaryKey` is held down *and* all the `ModifierKeys` (if any) are also held down.

* **Example**:

  ```csharp
  using UnityEngine;

  public class MapController : MonoBehaviour
  {
      void Start()
      {
          // Register Ctrl+M and Alt+M as synonyms:
          InputHelper.RegisterKeyBinding("ToggleMap", new InputHelper.KeyBinding {
              PrimaryKey = KeyCode.M,
              ModifierKeys = new KeyCode[]{ KeyCode.LeftControl, KeyCode.RightControl }
          });
      }

      void Update()
      {
          if (InputHelper.IsBindingActive("ToggleMap"))
          {
              ToggleMapUI();
          }
      }

      private void ToggleMapUI()
      {
          Debug.Log("Map toggled!");
          // Show/hide mini‐map
      }
  }
  ```

### 4.3 The `KeyBinding` Helper Type

```csharp
public class KeyBinding
{
    public KeyCode PrimaryKey { get; set; }
    public KeyCode[] ModifierKeys { get; set; }

    /// <summary>
    /// Returns true iff PrimaryKey is down and (if any) all ModifierKeys are also down.
    /// </summary>
    public bool IsActive()
    {
        if (!Input.GetKey(PrimaryKey))
            return false;

        if (ModifierKeys != null)
        {
            foreach (var modifier in ModifierKeys)
            {
                if (!Input.GetKey(modifier)) 
                    return false;
            }
        }

        return true;
    }
}
```

* **Usage**:

  * Set `PrimaryKey` to a `KeyCode` (e.g. `KeyCode.Space`).
  * Optionally set `ModifierKeys` to an array (e.g. `{ KeyCode.LeftAlt }`).
  * Call `IsActive()` each frame (or whenever) to see if the full combination is currently pressed.

---

## 5. Helper Types

### 5.1 `SwipeDirection` enum

```csharp
public enum SwipeDirection
{
    None,
    Up,
    Down,
    Left,
    Right
}
```

* Used as the return value of `DetectSwipe()`.
* `None` indicates no valid swipe was detected this frame.

### 5.2 `KeyBinding` class

*(See section 4.3 above.)*

---

## 6. Real-World Usage Examples

### 6.1 Bridging the New Input System with Legacy Input

You might have an action (e.g. “Reload Weapon”) that you want to trigger whenever the player either presses the “R” key (legacy) or the controller button “South” (new Input System). Here’s how you can do that:

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class ReloadManager : MonoBehaviour
{
    public InputActionAsset inputActions;
    private InputAction _reloadAction;

    void Awake()
    {
        // 1. Set up new Input System “Reload” action
        _reloadAction = inputActions.FindActionMap("Player").FindAction("Reload");
        InputHelper.RegisterAction("ReloadAction", _reloadAction, OnReloadPerformed);

        // 2. Set up a legacy key binding: “R” + no modifier
        var reloadBinding = new InputHelper.KeyBinding
        {
            PrimaryKey = KeyCode.R,
            ModifierKeys = null
        };
        InputHelper.RegisterKeyBinding("ReloadLegacy", reloadBinding);
    }

    void OnDestroy()
    {
        InputHelper.UnregisterAction("ReloadAction");
        InputHelper.RegisterKeyBinding("ReloadLegacy", null);
    }

    void Update()
    {
        // 3. Also check the legacy binding each frame:
        if (InputHelper.IsBindingActive("ReloadLegacy"))
        {
            PerformReload();
        }
    }

    private void OnReloadPerformed(InputAction.CallbackContext ctx)
    {
        if (ctx.performed)
            PerformReload();
    }

    private void PerformReload()
    {
        Debug.Log("Weapon reloaded!");
        // Actually reload weapon logic here…
    }
}
```

* Now **both** pressing “R” (legacy) and pressing the new Input System’s “Reload” button trigger `PerformReload()`.

---

### 6.2 Detecting Swipe Directions on Mobile

```csharp
using UnityEngine;

public class MobileCameraController : MonoBehaviour
{
    void Update()
    {
        var swipe = InputHelper.DetectSwipe();
        switch (swipe)
        {
            case InputHelper.SwipeDirection.Left:
                PanCameraLeft();
                break;
            case InputHelper.SwipeDirection.Right:
                PanCameraRight();
                break;
            case InputHelper.SwipeDirection.Up:
                ZoomIn();
                break;
            case InputHelper.SwipeDirection.Down:
                ZoomOut();
                break;
            default:
                break;
        }
    }

    private void PanCameraLeft() { Debug.Log("Panning camera left"); }
    private void PanCameraRight() { Debug.Log("Panning camera right"); }
    private void ZoomIn() { Debug.Log("Zooming in"); }
    private void ZoomOut() { Debug.Log("Zooming out"); }
}
```

* Each time you lift your finger, if you moved it more than 50 pixels in any direction, the camera pans or zooms accordingly.

---

### 6.3 Detecting Single and Double Taps

```csharp
using UnityEngine;

public class TapInteractor : MonoBehaviour
{
    void Update()
    {
        // Single‐tap: maybe place a marker
        if (InputHelper.DetectTap(out Vector2 tapPos))
        {
            Vector3 world = Camera.main.ScreenToWorldPoint(new Vector3(tapPos.x, tapPos.y, 10f));
            SpawnMarker(world);
        }

        // Double‐tap: maybe reset view
        if (InputHelper.DetectDoubleTap(out Vector2 dblTapPos))
        {
            CenterCameraOnTap(dblTapPos);
        }
    }

    private void SpawnMarker(Vector3 world) => Debug.Log($"Spawn marker at {world}");
    private void CenterCameraOnTap(Vector2 screen) => Debug.Log($"Center camera on screen point {screen}");
}
```

* The code differentiates a quick single tap (duration < 0.2s) from a double tap (two quick taps within 0.3s).
* On a single tap, `SpawnMarker(...)` runs; on a double tap, `CenterCameraOnTap(...)`.

---

### 6.4 Registering and Checking Custom Key Bindings

```csharp
using UnityEngine;

public class AbilityController : MonoBehaviour
{
    void Start()
    {
        // Register “UseAbility1” as LeftShift + 1
        var ability1 = new InputHelper.KeyBinding
        {
            PrimaryKey = KeyCode.Alpha1,
            ModifierKeys = new [] { KeyCode.LeftShift }
        };
        InputHelper.RegisterKeyBinding("UseAbility1", ability1);

        // Register “UseAbility2” as Alt + 2
        var ability2 = new InputHelper.KeyBinding
        {
            PrimaryKey = KeyCode.Alpha2,
            ModifierKeys = new [] { KeyCode.LeftAlt }
        };
        InputHelper.RegisterKeyBinding("UseAbility2", ability2);
    }

    void Update()
    {
        if (InputHelper.IsBindingActive("UseAbility1"))
        {
            ActivateAbility(1);
        }
        else if (InputHelper.IsBindingActive("UseAbility2"))
        {
            ActivateAbility(2);
        }
    }

    void ActivateAbility(int slot)
    {
        Debug.Log($"Activating ability {slot}");
        // Reset binding so it doesn’t fire every frame (if desired)
    }
}
```

* That registers two distinct multi-key combos. In `Update()`, you can check which one is currently held and react accordingly.

---

### 6.5 Combining Touch & Keyboard Input in a Single Helper

```csharp
using UnityEngine;

public class UnifiedInputDemo : MonoBehaviour
{
    void Update()
    {
        // 1. New Input System “Fire” action (if registered via InputHelper.RegisterAction)
        if (InputHelper.IsBindingActive("FireLegacy"))
        {
            FireWeapon();
        }

        // 2. Or if user swipes right on mobile:
        if (InputHelper.DetectSwipe() == InputHelper.SwipeDirection.Right)
        {
            TeleportRight();
        }

        // 3. Single tap to shoot on touch:
        if (InputHelper.DetectTap(out Vector2 tapPos))
        {
            ShootAtScreenPosition(tapPos);
        }
    }

    private void FireWeapon()    => Debug.Log("Firing (legacy).");
    private void TeleportRight() => Debug.Log("Teleporting right (swipe).");
    private void ShootAtScreenPosition(Vector2 pos) 
        => Debug.Log($"Shooting at {pos} (tap).");
}
```

* This demonstrates how to co‐exist new Input System, legacy key checks, and touch gestures in the same `Update()` loop.

---

## 7. Best Practices & Tips

1. **Debounce or “Consume” Gestures**

   * If you detect a swipe or tap and immediately act on it, you may wish to avoid detecting the same gesture on the next frame. You can do this by adding a short cooldown (e.g. `float lastGestureTime; if (Time.time − lastGestureTime > 0.1f) { … handle gesture; lastGestureTime = Time.time; }`).

2. **Disable Touch Logic on PC**

   * `DetectSwipe()` and the others already check `Input.touchSupported`. On desktop, they simply return `None`/`false`. You don’t need to surround them with additional platform checks.

3. **Combine with Screen-to-World Conversion If Needed**

   * A tap detection just gives you screen coordinates. To shoot or spawn at that point in world space, do:

     ```csharp
     Vector3 worldPos = Camera.main.ScreenToWorldPoint(new Vector3(screenPos.x, screenPos.y, cameraDistance));
     ```

4. **Avoid Overlapping Taps & Swipes**

   * If a user swipes, `DetectTap()` might briefly see a very short touch and misinterpret it. You can check `delta.magnitude` yourself or add a small cooldown.

5. **KeyBinding.isActive Fires Every Frame**

   * Since `IsBindingActive(...)` returns `true` for as long as the combination is held, you may want to add your own “edge detection” if you only want to fire once per key press.

6. **Unregister Input Actions When You’re Done**

   * Always call `UnregisterAction(name)` when the context changes (e.g. you navigate away from a menu) so that callbacks no longer fire.

7. **Keep Key/Action Names Consistent**

   * Use **constants** or an `enum` of action names (e.g. `public static class ActionNames { public const string Jump = "Jump"; /* … */ }`) to avoid typos when calling `RegisterAction("Jump", …)`.

8. **Customize Thresholds in Code If Needed**

   * While the default thresholds (`50f` pixels, `0.2` seconds, `0.3` seconds) work in most cases, you can adjust them by modifying `InputHelper.SwipeThreshold`, etc. early in your startup sequence.

---

## 8. Summary of Public API

Below is a quick reference for all public members in **InputHelper**:

### Touch Gesture Detection

| Method Signature                                    | Description                                                                                  |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `static SwipeDirection DetectSwipe()`               | Returns the cardinal `SwipeDirection` (Up, Down, Left, Right) if a swipe ended this frame.   |
| `static bool DetectTap(out Vector2 position)`       | Returns true if a quick tap (< 0.2s) was detected; outputs the screen position.              |
| `static bool DetectDoubleTap(out Vector2 position)` | Returns true if two taps occurred in rapid succession (< 0.3s); outputs the screen position. |

### New Input System Integration

| Method Signature                                                                                            | Description                                                                                                                  |
| ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `static void RegisterAction(string name, InputAction action, Action<InputAction.CallbackContext> callback)` | Enables and stores an `InputAction`, wiring its `.performed` to `callback`. If `name` already existed, disables the old one. |
| `static void UnregisterAction(string name)`                                                                 | Disables and removes the previously registered action with the given `name`.                                                 |

### Custom Key Bindings

| Method Signature                                                        | Description                                                  |
| ----------------------------------------------------------------------- | ------------------------------------------------------------ |
| `static void RegisterKeyBinding(string actionName, KeyBinding binding)` | Associates a `KeyBinding` object to the string `actionName`. |
| `static bool IsBindingActive(string actionName)`                        | Returns true if that `KeyBinding` is currently held down.    |

### Helper Types

| Type                                                                                | Purpose                                                                                          |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `enum SwipeDirection { None, Up, Down, Left, Right }`                               | Used by `DetectSwipe()` to indicate which direction the swipe went.                              |
| `class KeyBinding { KeyCode PrimaryKey; KeyCode[] ModifierKeys; bool IsActive(); }` | Holds a primary key and optional modifiers; `IsActive()` returns true if all are currently down. |

---

With this documentation, you should now be able to integrate **InputHelper** into your Unity projects for:

* Seamlessly combining the New Input System with legacy input checks
* Quickly adding touch‐based swipes, taps, and double‐tap logic
* Defining and querying custom key bindings (e.g. “Ctrl+Q” or “Shift+F”)
* Building cross-platform input code that works on both mobile and desktop

Happy coding!
