## Overview

This documentation covers two main helper systems for Unity animation management:

1. **AnimationEventHelper + AnimationHelper (Static)**

   * Provides utility methods to simplify playback, blending, speed control, parameter setting, event handling, and cleanup for `Animator` components at the static level.
2. **AnimationManager (MonoBehaviour Singleton)**

   * Offers a robust, high‐level management system that tracks animator states, handles smooth transitions, procedural IK, root motion integration, blending, event callbacks, sequences, recording, and performance optimizations.

Throughout this document, each method or feature is explained in detail, followed by a realistic example showing how you might use it in a Unity project. Code snippets assume C# in Unity (2020.3+), and that you have already set up `Animator` components and states as usual.

---

## 1. AnimationEventHelper

`AnimationEventHelper` is a small `MonoBehaviour` component whose sole purpose is to receive Unity animation events (via `Animation` or `Animator`) and invoke a C# delegate callback. By attaching `AnimationEventHelper` at runtime, you can register and unregister callbacks for string‐named animation events without manually wiring up event functions in each individual animator.

```csharp
/// <summary>
/// Helper component for managing animation events and coroutines
/// </summary>
public class AnimationEventHelper : MonoBehaviour
{
    // Delegate for animation events
    public delegate void AnimationEventCallback(string eventName);
    public event AnimationEventCallback OnAnimationEvent;

    private void OnDestroy()
    {
        StopAllCoroutines();
    }

    // Called by animation events
    public void TriggerAnimationEvent(string eventName)
    {
        OnAnimationEvent?.Invoke(eventName);
    }
}
```

### Key Parts

* **`AnimationEventCallback(string eventName)`**
  A delegate type; subscribers receive a string event name when `TriggerAnimationEvent` is called.

* **`OnAnimationEvent` (event)**
  External code can add or remove methods from this event. Whenever an animation event in Animator calls `TriggerAnimationEvent`, the registered callbacks fire.

* **`TriggerAnimationEvent(string eventName)`**
  Intended to be linked via Unity’s Animation window → Events (or via Timeline). For example, in the Animation Clip’s “Event” inspector, set the function to `TriggerAnimationEvent` and specify the event name (e.g., “Footstep”). When the animation reaches that frame, Unity invokes `AnimationEventHelper.TriggerAnimationEvent("Footstep")`, and all subscribed C# listeners run.

* **`OnDestroy()`**
  Ensures any coroutines started in this component are stopped if the GameObject is destroyed. This avoids “orphaned” coroutines.

### Real-World Example

**Scenario**: You have a humanoid character who should play a footstep sound whenever the walking animation reaches the “foot contact” frame. Rather than putting individual “PlaySound” methods inside every clip, you centralize your event callbacks with `AnimationEventHelper`.

1. **Attach `AnimationEventHelper` at runtime (via `AnimationHelper.RegisterAnimationEvent`).**
2. **In your Animation clip**, add an event at the moment you want the footstep. Set function to `TriggerAnimationEvent`, parameter = `"Footstep"`.

```csharp
public class FootstepSoundController : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    [SerializeField] private AudioClip _footstepClip;
    private AudioSource _audioSource;

    private void Awake()
    {
        _audioSource = GetComponent<AudioSource>();
        // Register for animation events
        AnimationHelper.RegisterAnimationEvent(_animator, OnAnimEventReceived);
    }

    private void OnDestroy()
    {
        // Unregister when this GameObject is destroyed
        AnimationHelper.UnregisterAnimationEvent(_animator, OnAnimEventReceived);
    }

    private void OnAnimEventReceived(string eventName)
    {
        if (eventName == "Footstep")
        {
            _audioSource.PlayOneShot(_footstepClip);
        }
    }
}
```

* When the walking animation’s foot‐impact frame is reached, `TriggerAnimationEvent("Footstep")` fires.
* That calls `OnAnimEventReceived("Footstep")`, which plays the sound.

---

## 2. AnimationHelper (Static)

`AnimationHelper` is a collection of static utility methods that simplify many common animation tasks for any `Animator`. It manages playback, blending, speed control, event registration (with the `AnimationEventHelper`), parameter safety, and cleanup. It uses two private dictionaries:

* **`_storedSpeeds: Dictionary<Animator, Dictionary<string, float>>`**
  Tracks custom speed multipliers per‐animator‐state, so you can restore original speed later.
* **`_eventHelpers: Dictionary<Animator, AnimationEventHelper>`**
  Keeps a single `AnimationEventHelper` attached to each `Animator.gameObject` for event callbacks.

```csharp
public static class AnimationHelper
{
    private static readonly Dictionary<Animator, Dictionary<string, float>> _storedSpeeds = 
        new Dictionary<Animator, Dictionary<string, float>>();

    private static readonly Dictionary<Animator, AnimationEventHelper> _eventHelpers = 
        new Dictionary<Animator, AnimationEventHelper>();

    #region Animation Control
    ...
    #endregion

    #region Runtime Modifications
    ...
    #endregion

    #region State Management
    ...
    #endregion

    #region Event Handling
    ...
    #endregion

    #region Parameter Management
    ...
    #endregion

    #region Cleanup
    ...
    #endregion
}
```

Below is a breakdown of each major section and method, including real‐world usage examples.

---

### 2.1 Animation Control

#### 2.1.1 `bool PlayAnimation(Animator animator, string stateName, int layer = 0, float normalizedTime = 0f, float speed = 1f)`

* **Purpose**: Safely play a named animation state on an `Animator`, optionally setting playback speed and start position (normalized time).

* **Parameters**

  * `animator`: Target `Animator` component.
  * `stateName`: Name of the state in the Animator Controller.
  * `layer`: Animation layer index (default = 0).
  * `normalizedTime`: Starting percentage (0–1) of the animation.
  * `speed`: Playback speed multiplier (default = 1).

* **Returns**: `true` if the call succeeded; `false` if `animator` is null, `stateName` is empty, or an exception occurred (logged to Console).

* **Implementation**:

  1. Check for null/empty.
  2. Set `animator.speed = speed`.
  3. Call `animator.Play(stateName, layer, normalizedTime)`.
  4. Catch any exceptions to avoid runtime crashes.

```csharp
public static bool PlayAnimation(Animator animator, string stateName, int layer = 0, float normalizedTime = 0f, float speed = 1f)
{
    if (animator == null || string.IsNullOrEmpty(stateName))
        return false;

    try
    {
        animator.speed = speed;
        animator.Play(stateName, layer, normalizedTime);
        return true;
    }
    catch (Exception e)
    {
        Debug.LogError($"Failed to play animation {stateName}: {e.Message}");
        return false;
    }
}
```

###### Real-World Example

**Scenario**: You want your 2D character to play a “Jump” animation from the middle of the clip at half speed.

```csharp
public class PlayerController : MonoBehaviour
{
    [SerializeField] private Animator _animator;

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            // Play "Jump" state on layer 0, halfway through (0.5f), at half speed (0.5f)
            AnimationHelper.PlayAnimation(_animator, "Jump", 0, 0.5f, 0.5f);
        }
    }
}
```

* The code sets `animator.speed = 0.5f`, then plays “Jump” from 50% through at slow speed.
* If “Jump” does not exist, or animator is null, it logs an error and returns `false`.

---

#### 2.1.2 `void CrossFadeAnimation(Animator animator, string stateName, float duration, int layer = -1)`

* **Purpose**: Smoothly blend from the current playing state to another state over a given duration.

* **Parameters**

  * `animator`: Target `Animator`.
  * `stateName`: Name of the animation state you want to transition to.
  * `duration`: Transition length in seconds.
  * `layer`: Layer index; `-1` means all layers (per Unity docs).

* **Behavior**: If `animator` and `stateName` are valid, calls `animator.CrossFade(stateName, duration, layer)`.

```csharp
public static void CrossFadeAnimation(Animator animator, string stateName, float duration, int layer = -1)
{
    if (animator != null && !string.IsNullOrEmpty(stateName))
    {
        animator.CrossFade(stateName, duration, layer);
    }
}
```

###### Real-World Example

**Scenario**: You want to smoothly transition from “Run” to “Slide” over 0.2 seconds when the player presses “Crawl” key.

```csharp
public class PlayerSlide : MonoBehaviour
{
    [SerializeField] private Animator _animator;

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.C))
        {
            // Smoothly transition into "Slide" over 0.2s on base layer
            AnimationHelper.CrossFadeAnimation(_animator, "Slide", 0.2f, 0);
        }
    }
}
```

* If the player is running (“Run” state) or idle, pressing “C” begins a 0.2s blend into “Slide.”

---

#### 2.1.3 `IEnumerator BlendLayer(Animator animator, int layer, float weight, float duration)`

* **Purpose**: Linearly interpolate (lerp) the weight of a specific animation layer from its current weight to a target weight over time.

* **Parameters**

  * `animator`: Target `Animator`.
  * `layer`: Index of the layer to blend.
  * `weight`: Final blend weight (0–1).
  * `duration`: Time in seconds over which to interpolate.

* **Returns**: A coroutine.

* **Implementation**:

  1. If `animator` is null, exit `yield break`.
  2. Read `startWeight = animator.GetLayerWeight(layer)`.
  3. Over `duration`, repeatedly `Mathf.Lerp(startWeight, weight, t)` to set `animator.SetLayerWeight(layer, newWeight)`.
  4. After `duration`, enforce final weight = `weight`.

```csharp
public static IEnumerator BlendLayer(Animator animator, int layer, float weight, float duration)
{
    if (animator == null) yield break;

    float startWeight = animator.GetLayerWeight(layer);
    float elapsed = 0f;

    while (elapsed < duration)
    {
        elapsed += Time.deltaTime;
        float t = elapsed / duration;
        animator.SetLayerWeight(layer, Mathf.Lerp(startWeight, weight, t));
        yield return null;
    }

    animator.SetLayerWeight(layer, weight);
}
```

###### Real-World Example

**Scenario**: You want to blend in an upper‐body aiming layer (IK layer) when the player equips a weapon, and blend it out when unequipped.

```csharp
public class AimController : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    private const int AimLayerIndex = 1;
    private Coroutine _blendCoroutine;

    public void EquipWeapon()
    {
        // Blend AimLayer from whatever it was to 1f over 0.3s
        if (_blendCoroutine != null) StopCoroutine(_blendCoroutine);
        _blendCoroutine = StartCoroutine(
            AnimationHelper.BlendLayer(_animator, AimLayerIndex, 1f, 0.3f)
        );
    }

    public void UnequipWeapon()
    {
        if (_blendCoroutine != null) StopCoroutine(_blendCoroutine);
        _blendCoroutine = StartCoroutine(
            AnimationHelper.BlendLayer(_animator, AimLayerIndex, 0f, 0.3f)
        );
    }
}
```

* When the player equips, the upper‐body IK layer weight smoothly rises from current to 1 over 0.3s.
* When unequipped, it fades back to 0.

---

### 2.2 Runtime Modifications

#### 2.2.1 `void SetAnimationSpeed(Animator animator, string stateName, float speedMultiplier)`

* **Purpose**: Change an animation state’s playback speed, storing the requested multiplier so you can restore original speed later. If the state is currently playing, the `animator.speed` is updated immediately.

* **Parameters**

  * `animator`: Target `Animator`.
  * `stateName`: Name of the state to which you want to apply the custom speed.
  * `speedMultiplier`: New playback speed (e.g., 2.0 for double speed).

* **Behavior**:

  1. If `animator` is null, return.
  2. Ensure `_storedSpeeds[animator]` dictionary exists.
  3. Store `speedMultiplier` under `stateName`.
  4. If `IsPlaying(animator, stateName)` is true, set `animator.speed = speedMultiplier`.

```csharp
public static void SetAnimationSpeed(Animator animator, string stateName, float speedMultiplier)
{
    if (animator == null) return;

    if (!_storedSpeeds.ContainsKey(animator))
    {
        _storedSpeeds[animator] = new Dictionary<string, float>();
    }

    _storedSpeeds[animator][stateName] = speedMultiplier;
    
    if (IsPlaying(animator, stateName))
    {
        animator.speed = speedMultiplier;
    }
}
```

###### Real-World Example

**Scenario**: You have a “CastingSpell” animation. When the player gains a “Haste” buff, you want that casting to play 50% faster.

```csharp
public class BuffController : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    private const string SpellState = "CastingSpell";

    public void OnHasteBuffApplied()
    {
        // Double‐speed play if currently casting
        AnimationHelper.SetAnimationSpeed(_animator, SpellState, 1.5f);
    }

    public void OnHasteBuffRemoved()
    {
        // Restore to normal speed
        AnimationHelper.RestoreAnimationSpeed(_animator, SpellState);
    }
}
```

* If the player is mid‐cast when the buff is applied, the state’s playback speeds up.
* When removed, state speed resets to 1.

---

#### 2.2.2 `void RestoreAnimationSpeed(Animator animator, string stateName)`

* **Purpose**: If you previously called `SetAnimationSpeed`, this method resets `animator.speed` back to 1 and clears the stored multiplier for `stateName`.

* **Parameters**

  * `animator`: Target `Animator`.
  * `stateName`: Name of the state whose custom speed you want to remove.

* **Behavior**:

  1. If `animator` is null or `_storedSpeeds` does not contain this animator, return.
  2. If `stateName` is stored, set `animator.speed = 1f` and remove this entry from `_storedSpeeds[animator]`.

```csharp
public static void RestoreAnimationSpeed(Animator animator, string stateName)
{
    if (animator == null || !_storedSpeeds.ContainsKey(animator)) return;

    if (_storedSpeeds[animator].ContainsKey(stateName))
    {
        animator.speed = 1f;
        _storedSpeeds[animator].Remove(stateName);
    }
}
```

###### Real-World Example

Continuing the **Haste** scenario above:

```csharp
public void OnHasteBuffRemoved()
{
    // Immediately restore to default speed if casting
    AnimationHelper.RestoreAnimationSpeed(_animator, SpellState);
}
```

* Ensures that if the casting animation was sped up, we revert it when buff ends.

---

### 2.3 State Management

#### 2.3.1 `bool IsPlaying(Animator animator, string stateName, int layer = 0)`

* **Purpose**: Check if a given `stateName` is currently playing on the specified `layer`.

* **Parameters**

  * `animator`: Target `Animator`.
  * `stateName`: Exact name of the state.
  * `layer`: Layer index (default = 0).

* **Returns**: `true` if the current state’s name matches `stateName`; `false` otherwise.

* **Implementation**:

  1. If `animator` is null, return `false`.
  2. Get `AnimatorStateInfo` from `animator.GetCurrentAnimatorStateInfo(layer)`.
  3. Return `stateInfo.IsName(stateName)`.

```csharp
public static bool IsPlaying(Animator animator, string stateName, int layer = 0)
{
    if (animator == null) return false;
    var stateInfo = animator.GetCurrentAnimatorStateInfo(layer);
    return stateInfo.IsName(stateName);
}
```

###### Real-World Example

**Scenario**: You don’t want to restart the “Jump” animation if it’s already playing.

```csharp
public void AttemptJump()
{
    if (!AnimationHelper.IsPlaying(_animator, "Jump"))
    {
        AnimationHelper.PlayAnimation(_animator, "Jump");
    }
}
```

* If the “Jump” state is already active on base layer, the call is skipped.

---

#### 2.3.2 `float GetNormalizedTime(Animator animator, int layer = 0)`

* **Purpose**: Obtain the normalized time (0–1) of the currently playing state on a given layer.
* **Parameters**

  * `animator`: Target `Animator`.
  * `layer`: Layer index (default = 0).
* **Returns**: Normalized progress of the current state's clip. Returns `0f` if `animator` is null.

```csharp
public static float GetNormalizedTime(Animator animator, int layer = 0)
{
    if (animator == null) return 0f;
    var stateInfo = animator.GetCurrentAnimatorStateInfo(layer);
    return stateInfo.normalizedTime;
}
```

###### Real-World Example

**Scenario**: You want to spawn a particle effect exactly when the “SwordSwing” animation reaches 80% of its duration.

```csharp
public class SwordController : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    [SerializeField] private ParticleSystem _slashEffect;

    void Update()
    {
        // Check if SwordSwing is playing
        if (AnimationHelper.IsPlaying(_animator, "SwordSwing"))
        {
            float progress = AnimationHelper.GetNormalizedTime(_animator);
            if (progress >= 0.8f && !_slashEffect.isPlaying)
            {
                // Spawn effect exactly at 80% of the swing
                _slashEffect.Play();
            }
        }
    }
}
```

---

#### 2.3.3 `IEnumerator WaitForAnimationComplete(Animator animator, string stateName, int layer = 0)`

* **Purpose**: A coroutine that suspends execution until the specified `stateName` finishes playing.

* **Parameters**

  * `animator`: Target `Animator`.
  * `stateName`: The state to wait for.
  * `layer`: The layer index (default = 0).

* **Usage**: Often used to chain behaviors after an animation without polling.

* **Implementation**:

  1. If `animator` is null, exit.
  2. First, wait until the animator actually enters `stateName` (`while (!IsPlaying(...)) yield return null;`).
  3. Once playing, obtain `stateInfo.length` (in seconds) and `yield return new WaitForSeconds(length)` to wait the remainder of the clip.

```csharp
public static IEnumerator WaitForAnimationComplete(Animator animator, string stateName, int layer = 0)
{
    if (animator == null) yield break;

    // Wait for the state to begin
    while (!IsPlaying(animator, stateName, layer))
        yield return null;

    var stateInfo = animator.GetCurrentAnimatorStateInfo(layer);
    // Wait for the clip’s full length
    yield return new WaitForSeconds(stateInfo.length);
}
```

###### Real-World Example

**Scenario**: You want the character to automatically switch back to “Idle” only after the “Attack” animation has fully played.

```csharp
public class EnemyAI : MonoBehaviour
{
    [SerializeField] private Animator _animator;

    public void PerformAttack()
    {
        // Start the attack and then return to idle
        StartCoroutine(AttackRoutine());
    }

    private IEnumerator AttackRoutine()
    {
        if (AnimationHelper.PlayAnimation(_animator, "Attack"))
        {
            // Wait for the “Attack” clip to complete
            yield return AnimationHelper.WaitForAnimationComplete(_animator, "Attack");
            // Now that attack is done, play Idle
            AnimationHelper.PlayAnimation(_animator, "Idle");
        }
    }
}
```

* Kick off “Attack”, then transparently yield until it ends, before resuming “Idle.”

---

### 2.4 Event Handling

#### 2.4.1 `void RegisterAnimationEvent(Animator animator, AnimationEventHelper.AnimationEventCallback callback)`

* **Purpose**: Attach an `AnimationEventHelper` on the `animator.gameObject` if not already present, and then subscribe the provided callback to its `OnAnimationEvent` event.

* **Parameters**

  * `animator`: Target `Animator`.
  * `callback`: Method with signature `(string eventName)` that should be invoked when any animation event is triggered.

* **Behavior**:

  1. If `animator` is null, return.
  2. If `_eventHelpers` does not contain this animator, add a new `AnimationEventHelper` via `animator.gameObject.AddComponent<AnimationEventHelper>()` and store in `_eventHelpers`.
  3. Subscribe `callback` to `OnAnimationEvent`.

```csharp
public static void RegisterAnimationEvent(Animator animator, AnimationEventHelper.AnimationEventCallback callback)
{
    if (animator == null) return;

    if (!_eventHelpers.ContainsKey(animator))
    {
        var helper = animator.gameObject.AddComponent<AnimationEventHelper>();
        _eventHelpers[animator] = helper;
    }

    _eventHelpers[animator].OnAnimationEvent += callback;
}
```

###### Real-World Example

Building on the “Footstep” example in **Section 1**, but now explicitly using `AnimationHelper`:

```csharp
public class FootstepSoundController : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    [SerializeField] private AudioClip _footstepClip;
    private AudioSource _audioSource;

    private void Awake()
    {
        _audioSource = GetComponent<AudioSource>();
        AnimationHelper.RegisterAnimationEvent(_animator, OnAnimEventReceived);
    }

    private void OnDestroy()
    {
        AnimationHelper.UnregisterAnimationEvent(_animator, OnAnimEventReceived);
    }

    private void OnAnimEventReceived(string eventName)
    {
        if (eventName == "Footstep")
        {
            _audioSource.PlayOneShot(_footstepClip);
        }
    }
}
```

#### 2.4.2 `void UnregisterAnimationEvent(Animator animator, AnimationEventHelper.AnimationEventCallback callback)`

* **Purpose**: Remove a previously registered callback from the `AnimationEventHelper`.

* **Parameters**

  * `animator`: Target `Animator`.
  * `callback`: Delegate reference to remove.

* **Behavior**:

  1. If `animator` is null or `_eventHelpers` does not contain it, return.
  2. Unsubscribe `callback` from `OnAnimationEvent`.

```csharp
public static void UnregisterAnimationEvent(Animator animator, AnimationEventHelper.AnimationEventCallback callback)
{
    if (animator == null || !_eventHelpers.ContainsKey(animator)) return;
    _eventHelpers[animator].OnAnimationEvent -= callback;
}
```

---

### 2.5 Parameter Management

#### 2.5.1 `void SetParameter(Animator animator, string paramName, object value)`

* **Purpose**: Safely set a parameter on the `Animator`, validating both that the parameter exists and that `value` matches the expected type (`Bool`, `Int`, `Float`, or `Trigger`).

* **Parameters**

  * `animator`: Target `Animator`.
  * `paramName`: Name of the parameter in the Animator Controller.
  * `value`: A `bool` (for `Bool` or `Trigger`), `int` (for `Int`), or `float` (for `Float`).

* **Behavior**:

  1. If `animator` is null or `paramName` is empty, return immediately.
  2. Iterate `animator.parameters`; find one whose `name` matches `paramName`.
  3. Based on `param.type`, cast `value` to expected type.

     * If `Bool` and `value is bool`, call `animator.SetBool(paramName, boolValue)`.
     * If `Int` and `value is int`, call `animator.SetInteger(paramName, intValue)`.
     * If `Float` and `value is float`, call `animator.SetFloat(paramName, floatValue)`.
     * If `Trigger` and `value is bool triggerValue`:

       * If `triggerValue == true`, call `animator.SetTrigger(paramName)`.
       * Else (`false`), call `animator.ResetTrigger(paramName)`.
  4. If no parameter matches, nothing happens.

```csharp
public static void SetParameter(Animator animator, string paramName, object value)
{
    if (animator == null || string.IsNullOrEmpty(paramName)) return;

    foreach (AnimatorControllerParameter param in animator.parameters)
    {
        if (param.name == paramName)
        {
            switch (param.type)
            {
                case AnimatorControllerParameterType.Bool when value is bool boolValue:
                    animator.SetBool(paramName, boolValue);
                    break;
                case AnimatorControllerParameterType.Int when value is int intValue:
                    animator.SetInteger(paramName, intValue);
                    break;
                case AnimatorControllerParameterType.Float when value is float floatValue:
                    animator.SetFloat(paramName, floatValue);
                    break;
                case AnimatorControllerParameterType.Trigger when value is bool triggerValue:
                    if (triggerValue)
                        animator.SetTrigger(paramName);
                    else
                        animator.ResetTrigger(paramName);
                    break;
            }
            break;
        }
    }
}
```

###### Real-World Example

**Scenario**: Your enemy AI controller uses an integer parameter “Health” and a boolean parameter “IsDead.” When the enemy takes damage, you want to update the animator accordingly:

```csharp
public class EnemyHealth : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    private int _health = 100;

    public void TakeDamage(int amount)
    {
        _health -= amount;
        if (_health <= 0)
        {
            _health = 0;
            // Trigger the death animation
            AnimationHelper.SetParameter(_animator, "IsDead", true);
        }
        else
        {
            // Update Health parameter so the blend tree shifts to a “hurt” cycle
            AnimationHelper.SetParameter(_animator, "Health", _health);
        }
    }
}
```

---

### 2.6 Cleanup

#### 2.6.1 `void Cleanup(Animator animator)`

* **Purpose**: Removes all helper data and components used by `AnimationHelper` for this `animator`.
  Specifically:

  1. Removes any stored speed entries in `_storedSpeeds`.
  2. Removes and destroys the `AnimationEventHelper` attached in `_eventHelpers`.

* **Parameters**

  * `animator`: Target `Animator`.

```csharp
public static void Cleanup(Animator animator)
{
    if (animator == null) return;

    if (_storedSpeeds.ContainsKey(animator))
        _storedSpeeds.Remove(animator);

    if (_eventHelpers.ContainsKey(animator))
    {
        if (_eventHelpers[animator] != null)
            UnityEngine.Object.Destroy(_eventHelpers[animator]);
        _eventHelpers.Remove(animator);
    }
}
```

###### Real-World Example

**Scenario**: You have a pooled enemy prefab on which you registered multiple animation event callbacks and possibly changed speeds. Before returning that enemy to a pool (or when it is destroyed), you need to remove all references to avoid leaks:

```csharp
public class EnemyPoolable : MonoBehaviour
{
    [SerializeField] private Animator _animator;

    private void OnDisable()
    {
        // Clean up all internal helper states
        AnimationHelper.Cleanup(_animator);
    }
}
```

---

## 3. AnimationManager (MonoBehaviour Singleton)

`AnimationManager` is a comprehensive manager attached to a dedicated GameObject named `[AnimationManager]` (created automatically at app startup). It continuously polls all registered animators at a fixed frequency to track state entry/exit, maintain transition coroutines, handle parameter lerps, apply procedural IK, root motion integration, blending, sequences of animations, recording frames, and other advanced utilities.

Because it uses the Singleton pattern with `[RuntimeInitializeOnLoadMethod]`, you never need to manually add it to a scene—just call `AnimationManager.Instance` and its `Awake` sets up routines.

```csharp
public class AnimationManager : MonoBehaviour
{
    #region Singleton Pattern
    private static AnimationManager _instance;
    public static AnimationManager Instance
    {
        get
        {
            if (_instance == null)
            {
                var go = new GameObject("[AnimationManager]");
                _instance = go.AddComponent<AnimationManager>();
                DontDestroyOnLoad(go);
            }
            return _instance;
        }
    }

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void EnsureInitialized()
    {
        var _ = Instance;
    }
    #endregion

    #region Data Structures
    private class AnimatorState
    {
        public Animator Animator;
        public string CurrentState;
        public string PreviousState;
        public float StateEnterTime;
        public Dictionary<string, AnimatorControllerParameter> Parameters;
        public Coroutine ActiveCoroutine;

        public Dictionary<string, AnimationEventCallback> EventCallbacks = new Dictionary<string, AnimationEventCallback>();
        public Dictionary<string, RuntimeAnimatorController> CachedControllers = new Dictionary<string, RuntimeAnimatorController>();
        public Queue<AnimationRecordFrame> RecordedFrames = new Queue<AnimationRecordFrame>();
        public bool IsRecording;
        public float RecordStartTime;
    }

    public class AnimationOverride
    {
        public AnimationClip OriginalClip;
        public AnimationClip OverrideClip;
    }

    public class AnimationEventCallback
    {
        public string EventName;
        public Action<AnimationEvent> Callback;
        public bool IsPersistent;
    }

    public class AnimationRecordFrame
    {
        public float TimeStamp;
        public Dictionary<string, float> Parameters;
        public Vector3 Position;
        public Quaternion Rotation;
        public Dictionary<AvatarIKGoal, IKData> IKData;
    }

    public class IKData
    {
        public Vector3 Position;
        public Quaternion Rotation;
        public float PositionWeight;
        public float RotationWeight;
    }

    public class AnimationProfile
    {
        public float TransitionSpeed = 1f;
        public bool UseSmoothing = true;
        public float SmoothingWeight = 0.5f;
        public bool EnableRootMotion = true;
        public float MaxVelocity = 5f;
        public AnimationCurve BlendCurve = AnimationCurve.EaseInOut(0, 0, 1, 1);
    }
    #endregion

    #region Properties and Fields
    private readonly Dictionary<Animator, AnimatorState> _animatorStates = new Dictionary<Animator, AnimatorState>();
    private readonly List<Animator> _pendingCleanup = new List<Animator>();

    [SerializeField] private float _defaultTransitionDuration = 0.25f;
    [SerializeField] private int _statePollingFrequency = 30;

    public event Action<Animator, string> OnStateEntered;
    public event Action<Animator, string> OnStateExited;
    #endregion

    #region Initialization
    private void Awake()
    {
        StartCoroutine(StatePollingRoutine());
        StartCoroutine(CleanupRoutine());
    }

    private IEnumerator StatePollingRoutine()
    {
        var wait = new WaitForSeconds(1f / _statePollingFrequency);
        while (true)
        {
            UpdateAllAnimatorStates();
            yield return wait;
        }
    }

    private IEnumerator CleanupRoutine()
    {
        var wait = new WaitForSeconds(5f);
        while (true)
        {
            CleanupInvalidAnimators();
            yield return wait;
        }
    }
    #endregion

    ...
}
```

### 3.1 Data Structures

* **`AnimatorState`**:
  Tracks state info for each registered `Animator`, including:

  * `Animator`: The reference.
  * `CurrentState` & `PreviousState`: Strings for fullPathHash names.
  * `StateEnterTime`: UNIX‐style `Time.time` when entered.
  * `Parameters`: Cached dictionary mapping parameter names to `AnimatorControllerParameter`.
  * `ActiveCoroutine`: The currently running coroutine handling transitions, parameter fades, etc.
  * `EventCallbacks`: Mapping from string event names to `AnimationEventCallback` (which wraps an `Action<AnimationEvent>` and whether it is persistent).
  * `CachedControllers`: Mapping from string keys to `RuntimeAnimatorController` (for quickly switching controllers).
  * `RecordedFrames`: Queue of frames for recording (position, rotation, parameter values, IK data).
  * `IsRecording` & `RecordStartTime`: Flags and timestamp for whether recording is active.

* **`AnimationOverride`**:
  Simple pair of `(OriginalClip, OverrideClip)` used when building an `AnimatorOverrideController`.

* **`AnimationEventCallback`**:
  Associates an `eventName` with a callback `Action<AnimationEvent>` and a boolean `IsPersistent`. Non‐persistent callbacks are removed after firing once.

* **`AnimationRecordFrame`**:
  Stores a single frame of recorded animation (timestamp, parameter values, root `Position` and `Rotation`, plus any IK data for hands/feet).

* **`IKData`**:
  Holds position, rotation, and weights for a single `AvatarIKGoal`.

* **`AnimationProfile`**:
  Contains various runtime configuration values (transition speed, smoothing, root motion toggle, max velocity, blending curve).

---

### 3.2 Core Public API

The `AnimationManager` exposes several high‐level methods for controlling animations centrally. All of these ensure the `Animator` is valid and automatically register it in `_animatorStates`.

#### 3.2.1 `void PlayState(Animator animator, object stateName, float transitionDuration = -1f, int layer = 0)`

* **Purpose**: Transition from the current state to the given state (hash or string) over `transitionDuration` seconds (or `_defaultTransitionDuration` if negative).

* **Parameters**

  * `animator`: Target `Animator`.
  * `stateName`: Either a `string` (state name) or an `int` (precomputed hash).
  * `transitionDuration`: If `< 0`, uses `_defaultTransitionDuration` (0.25s by default).
  * `layer`: Which layer to play the state on (default = 0).

* **Behavior**:

  1. Validate `animator` (must be non‐null, active).
  2. Convert `stateName` to an integer hash via `ConvertStateName`.
  3. If `AnimatorState.ActiveCoroutine` exists (e.g., a previous transition), `StopCoroutine` it.
  4. Start `TransitionRoutine(animator, nameHash, duration, layer)` and store its `Coroutine` in `ActiveCoroutine`.

```csharp
public void PlayState(
    Animator animator,
    object stateName,
    float transitionDuration = -1f,
    int layer = 0)
{
    if (!ValidateAnimator(animator)) return;
    var duration = transitionDuration < 0 ? _defaultTransitionDuration : transitionDuration;
    var nameHash = ConvertStateName(stateName);

    var stateInfo = GetAnimatorState(animator);
    if (stateInfo.ActiveCoroutine != null)
    {
        StopCoroutine(stateInfo.ActiveCoroutine);
    }

    stateInfo.ActiveCoroutine = StartCoroutine(
        TransitionRoutine(animator, nameHash, duration, layer)
    );
}
```

**`TransitionRoutine`**:

* Cross‐fades to the new state, then waits until the next frame in which the new state’s hash differs from the previous, then sets `ActiveCoroutine = null`.

```csharp
private IEnumerator TransitionRoutine(
    Animator animator,
    int targetStateHash,
    float duration,
    int layer)
{
    var state = GetAnimatorState(animator);
    var currentState = animator.GetCurrentAnimatorStateInfo(layer);

    // Trigger the cross‐fade
    animator.CrossFade(targetStateHash, duration, layer);

    // Wait until the current state after the transition is NOT the same as before
    yield return new WaitUntil(() =>
        animator.GetCurrentAnimatorStateInfo(layer).fullPathHash != currentState.fullPathHash
    );

    state.ActiveCoroutine = null;
}
```

###### Real-World Example

**Scenario**: You have a third‐person character that idles, then runs to attack, then returns to idle when attack finishes. You want to ensure no conflicting transitions and use a consistent `AnimationManager`.

```csharp
public class CharacterCombat : MonoBehaviour
{
    [SerializeField] private Animator _animator;

    private void Start()
    {
        // Subscribe to state‐entered notifications (optional)
        AnimationManager.Instance.OnStateEntered += OnAnimationStateEntered;
    }

    public void AttackTarget()
    {
        // Transition from idle/run to "Attack" state on base layer with 0.2s fade
        AnimationManager.Instance.PlayState(_animator, "Attack", 0.2f, 0);
    }

    private void OnAnimationStateEntered(Animator animator, string newState)
    {
        if (animator == _animator && newState == "Attack")
        {
            // Maybe spawn hitbox or damage logic here
            Debug.Log("Attack has started!");
        }
    }
}
```

* Calling `PlayState` ensures any existing transition on that `Animator` is canceled; the new `TransitionRoutine` cross‐fades to “Attack.”

---

#### 3.2.2 `void LerpParameter(Animator animator, string parameterName, float targetValue, float duration, Action onComplete = null)`

* **Purpose**: Smoothly interpolate a float parameter from its current value to `targetValue` over `duration` seconds, then optionally invoke a callback.

* **Parameters**

  * `animator`: Target `Animator`.
  * `parameterName`: Name of the float parameter.
  * `targetValue`: Desired final value.
  * `duration`: Time in seconds.
  * `onComplete`: (Optional) Action to call after interpolation finishes.

* **Behavior**:

  1. Validate `animator`.
  2. Validate that `parameterName` exists and is of type `Float` via `ValidateParameter`.
  3. Stop any `state.ActiveCoroutine`.
  4. Start `ParameterLerpRoutine` which each frame calculates `t = elapsed/duration`, sets `animator.SetFloat(parameterName, Mathf.Lerp(startValue, targetValue, t))`. When done, sets final value and invokes `onComplete`.

```csharp
public void LerpParameter(
    Animator animator,
    string parameterName,
    float targetValue,
    float duration,
    Action onComplete = null)
{
    if (!ValidateAnimator(animator)) return;
    if (!ValidateParameter(animator, parameterName, AnimatorControllerParameterType.Float))
        return;

    var state = GetAnimatorState(animator);
    if (state.ActiveCoroutine != null)
    {
        StopCoroutine(state.ActiveCoroutine);
    }

    state.ActiveCoroutine = StartCoroutine(
        ParameterLerpRoutine(
            animator,
            parameterName,
            targetValue,
            duration,
            onComplete
        )
    );
}

private IEnumerator ParameterLerpRoutine(
    Animator animator,
    string parameterName,
    float targetValue,
    float duration,
    Action onComplete)
{
    float startValue = animator.GetFloat(parameterName);
    float elapsed = 0;

    while (elapsed < duration)
    {
        if (!animator || !animator.gameObject.activeInHierarchy) yield break;

        elapsed += Time.deltaTime;
        float t = Mathf.Clamp01(elapsed / duration);
        animator.SetFloat(parameterName, Mathf.Lerp(startValue, targetValue, t));
        yield return null;
    }

    animator.SetFloat(parameterName, targetValue);
    onComplete?.Invoke();
}
```

###### Real-World Example

**Scenario**: You have a “Speed” parameter controlling a blend tree from Walk → Run. When the player starts sprinting, you want to smoothly ramp speed from current to 1.5f over 0.5 seconds.

```csharp
public class PlayerMovement : MonoBehaviour
{
    [SerializeField] private Animator _animator;

    public void StartSprinting()
    {
        // Ramp “Speed” param from whatever it currently is to 1.5f over 0.5s
        AnimationManager.Instance.LerpParameter(
            _animator,
            "Speed",
            1.5f,
            0.5f,
            onComplete: () => Debug.Log("Sprint parameter reached target!")
        );
    }

    public void StopSprinting()
    {
        // Ramp back to 0.8f (jog speed) over 0.3s
        AnimationManager.Instance.LerpParameter(
            _animator,
            "Speed",
            0.8f,
            0.3f
        );
    }
}
```

* While sprinting, `Speed` is smoothly changed, driving the blend tree. On complete, a message logs.

---

#### 3.2.3 `void ApplyOverrides(Animator animator, List<AnimationOverride> overrides)`

* **Purpose**: Replace specific animation clips in an animator controller at runtime by creating an `AnimatorOverrideController`. This allows you to dynamically swap out animation clips (e.g., for different character outfits).

* **Parameters**

  * `animator`: Target `Animator`.
  * `overrides`: A list of `AnimationOverride`, each containing `OriginalClip` (the clip name in base controller) and `OverrideClip` (the new clip).

* **Behavior**:

  1. Validate `animator`.
  2. Create a new `AnimatorOverrideController` based on `animator.runtimeAnimatorController`.
  3. Build a `List<KeyValuePair<AnimationClip, AnimationClip>>` from each `AnimationOverride`.
  4. Call `ApplyOverrides(clipPairs)` on the new override controller.
  5. Assign `animator.runtimeAnimatorController = overrideController`.

```csharp
public void ApplyOverrides(
    Animator animator,
    List<AnimationOverride> overrides)
{
    if (!ValidateAnimator(animator)) return;

    var overrideController = new AnimatorOverrideController(
        animator.runtimeAnimatorController
    );

    var clipPairs = overrides.Select(o => new KeyValuePair<AnimationClip, AnimationClip>(
        o.OriginalClip,
        o.OverrideClip
    )).ToList();

    overrideController.ApplyOverrides(clipPairs);
    animator.runtimeAnimatorController = overrideController;
}
```

###### Real-World Example

**Scenario**: You have two sets of sword swing animations—“Sword\_Swing\_Wood” and “Sword\_Swing\_Metal.” At runtime, if the player picks up a metal sword, you override the swing clip to use the metal version.

```csharp
public class WeaponSwitcher : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    [SerializeField] private AnimationClip _woodSwing;
    [SerializeField] private AnimationClip _metalSwing;

    public void EquipMetalSword()
    {
        var overrides = new List<AnimationManager.AnimationOverride>
        {
            new AnimationManager.AnimationOverride {
                OriginalClip = _woodSwing,
                OverrideClip  = _metalSwing
            }
        };
        
        AnimationManager.Instance.ApplyOverrides(_animator, overrides);
    }
}
```

* If the base controller originally used `_woodSwing` for the “Swing” state, after calling `ApplyOverrides`, the “Swing” state will play `_metalSwing` instead.

---

### 3.3 State Management & Polling

#### 3.3.1 `UpdateAllAnimatorStates()`

* Called periodically (via `StatePollingRoutine`) to compare each tracked animator’s current state name to the last known `AnimatorState.CurrentState`.

  1. Loop through all `KeyValuePair<Animator, AnimatorState>`.
  2. If the `Animator` is null, skip.
  3. Get `currentStateInfo = animator.GetCurrentAnimatorStateInfo(0)`.
  4. Convert `fullPathHash` to a string name via the placeholder `GetStateName(...)` (note: this method currently returns `hash.ToString()`, but you can customize it to return actual clip names if you maintain a lookup table).
  5. If `currentStateName != state.CurrentState`, that means we entered a new state. Invoke `OnStateExited(animator, previousState)`, update `state.PreviousState`, `state.CurrentState = currentStateName`, `state.StateEnterTime = Time.time`, then invoke `OnStateEntered(animator, currentStateName)`.

```csharp
private void UpdateAllAnimatorStates()
{
    foreach (var pair in _animatorStates)
    {
        if (pair.Key == null) continue;

        var currentState = pair.Key.GetCurrentAnimatorStateInfo(0);
        var currentStateName = GetStateName(pair.Key, currentState.fullPathHash);

        if (pair.Value.CurrentState != currentStateName)
        {
            OnStateExited?.Invoke(pair.Key, pair.Value.CurrentState);
            pair.Value.PreviousState = pair.Value.CurrentState;
            pair.Value.CurrentState = currentStateName;
            pair.Value.StateEnterTime = Time.time;
            OnStateEntered?.Invoke(pair.Key, currentStateName);
        }
    }
}
```

* **Note**: `GetStateName(...)` currently returns `hash.ToString()`. If you want human‐readable state names, you need to build your own mapping from hash to string using `Animator.StringToHash` (e.g., store a dictionary during initialization).

#### 3.3.2 `CleanupInvalidAnimators()`

* Called periodically (via `CleanupRoutine`) to remove any `Animator` entries whose GameObject is destroyed or inactive.

1. Clear `_pendingCleanup`.
2. Loop `_animatorStates.Keys`; if `animator == null` or `animator.gameObject.activeInHierarchy == false`, add to `_pendingCleanup`.
3. Remove each such animator from `_animatorStates`.

```csharp
private void CleanupInvalidAnimators()
{
    _pendingCleanup.Clear();

    foreach (var animator in _animatorStates.Keys)
    {
        if (animator == null || !animator.gameObject.activeInHierarchy)
        {
            _pendingCleanup.Add(animator);
        }
    }

    foreach (var animator in _pendingCleanup)
    {
        _animatorStates.Remove(animator);
    }
}
```

---

### 3.4 Helper Methods

These methods handle state dictionary lookups, parameter validation, animator validation, and name hashing:

#### 3.4.1 `AnimatorState GetAnimatorState(Animator animator)`

* **Purpose**: Retrieve or create the `AnimatorState` object for a given `Animator`.
* **Behavior**:

  1. If `_animatorStates` already has an entry for this `animator`, return it.
  2. Otherwise, create a new `AnimatorState`:

     * `Animator` = the provided `animator`.
     * `Parameters` = `animator.parameters.ToDictionary(p => p.name)`.
     * Initialize `CurrentState` and `PreviousState` to empty strings, etc.
  3. Store in `_animatorStates` and return.

```csharp
private AnimatorState GetAnimatorState(Animator animator)
{
    if (!_animatorStates.TryGetValue(animator, out var state))
    {
        state = new AnimatorState
        {
            Animator = animator,
            Parameters = animator.parameters.ToDictionary(p => p.name)
        };
        _animatorStates[animator] = state;
    }
    return state;
}
```

#### 3.4.2 `bool ValidateAnimator(Animator animator)`

* **Purpose**: Ensure `animator` is non‐null and active.
* **Returns**: `true` if valid; logs an error or warning if not.

```csharp
private bool ValidateAnimator(Animator animator)
{
    if (!animator)
    {
        Debug.LogError("Invalid animator reference!");
        return false;
    }

    if (!animator.isActiveAndEnabled)
    {
        Debug.LogWarning($"Animator {animator.name} is disabled!");
        return false;
    }

    return true;
}
```

#### 3.4.3 `bool ValidateParameter(Animator animator, string parameterName, AnimatorControllerParameterType expectedType)`

* **Purpose**: Check that the `Animator` has a parameter with name `parameterName` and that its type matches `expectedType`.
* **Returns**: `true` if valid; logs an error if parameter is missing or wrong type.

```csharp
private bool ValidateParameter(
    Animator animator,
    string parameterName,
    AnimatorControllerParameterType expectedType)
{
    var state = GetAnimatorState(animator);
    if (!state.Parameters.TryGetValue(parameterName, out var parameter))
    {
        Debug.LogError($"Parameter {parameterName} not found on {animator.name}");
        return false;
    }

    if (parameter.type != expectedType)
    {
        Debug.LogError($"Parameter {parameterName} is type {parameter.type}, expected {expectedType}");
        return false;
    }

    return true;
}
```

#### 3.4.4 `string GetStateName(Animator animator, int hash)`

* **Purpose (placeholder)**: Return a human‐readable string for an animation state `hash`.
* **Current Implementation**: Returns `hash.ToString()`.
* **Customization**: You might want to build a dictionary from `AnimatorController` which maps `hash` to clip names (using `Animator.StringToHash(clipName)` or other tooling).

```csharp
private string GetStateName(Animator animator, int hash)
{
    // Implementation requires custom state name lookup (e.g., using reflection or a map).
    return hash.ToString();
}
```

#### 3.4.5 `int ConvertStateName(object stateName)`

* **Purpose**: Convert between `string` (state name) or `int` (already hashed) to an integer hash to be used in `animator.CrossFade` or `animator.Play(...)`.
* **Behavior**:

  * If `stateName` is a `string`, call `Animator.StringToHash`.
  * If `stateName` is an `int`, return that `int`.
  * Otherwise, throw an exception.

```csharp
private int ConvertStateName(object stateName)
{
    return stateName switch
    {
        string s => Animator.StringToHash(s),
        int i => i,
        _ => throw new ArgumentException("Invalid state name type")
    };
}
```

---

## 4. Advanced Features (New in Version)

The code includes several advanced routines that go beyond simple cross‐fade and parameter lerping. These features automate complex tasks, such as IK transitions, procedural animation, root motion integration, blending multiple states, and sequencing several animations. Below is a detailed breakdown.

### 4.1 Advanced Inverse Kinematics Control

#### 4.1.1 `void SetIKTarget(Animator animator, AvatarIKGoal ikGoal, Transform target, float positionWeight = 1f, float rotationWeight = 1f, float transitionDuration = 0.3f)`

* **Purpose**: Gradually blend IK weights for a specific `AvatarIKGoal` (e.g., left hand, right hand) from 0 to `positionWeight`/`rotationWeight` over `transitionDuration`. Simultaneously, update the IK target’s position and rotation each frame.
* **Parameters**

  * `animator`: The `Animator` with `AvatarIK` enabled.
  * `ikGoal`: One of `AvatarIKGoal.LeftHand`, `RightHand`, `LeftFoot`, `RightFoot`.
  * `target`: The `Transform` representing world position/rotation for the IK effector.
  * `positionWeight`: The final weight for IK position (0–1).
  * `rotationWeight`: The final weight for IK rotation (0–1).
  * `transitionDuration`: Time (in seconds) to ramp from 0 to the given weights.

```csharp
public void SetIKTarget(
    Animator animator,
    AvatarIKGoal ikGoal,
    Transform target,
    float positionWeight = 1f,
    float rotationWeight = 1f,
    float transitionDuration = 0.3f)
{
    if (!ValidateAnimator(animator)) return;

    StartCoroutine(IKTransitionRoutine(
        animator,
        ikGoal,
        target,
        positionWeight,
        rotationWeight,
        transitionDuration
    ));
}

private IEnumerator IKTransitionRoutine(
    Animator animator,
    AvatarIKGoal ikGoal,
    Transform target,
    float positionWeight,
    float rotationWeight,
    float duration)
{
    float elapsed = 0f;
    while (elapsed < duration)
    {
        float t = elapsed / duration;
        
        animator.SetIKPositionWeight(ikGoal, Mathf.Lerp(0, positionWeight, t));
        animator.SetIKRotationWeight(ikGoal, Mathf.Lerp(0, rotationWeight, t));
        
        if (target != null)
        {
            animator.SetIKPosition(ikGoal, target.position);
            animator.SetIKRotation(ikGoal, target.rotation);
        }
        
        elapsed += Time.deltaTime;
        yield return null;
    }

    // Ensure final weight is explicitly set
    animator.SetIKPositionWeight(ikGoal, positionWeight);
    animator.SetIKRotationWeight(ikGoal, rotationWeight);
    if (target != null)
    {
        animator.SetIKPosition(ikGoal, target.position);
        animator.SetIKRotation(ikGoal, target.rotation);
    }
}
```

###### Real-World Example

**Scenario**: You have a character who, when holding a gun, should aim the right hand’s IK target at the gun’s muzzle. When holstering, the IK weight should smoothly transition down.

```csharp
public class GunAimingController : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    [SerializeField] private Transform _gunMuzzleTransform;

    public void StartAiming()
    {
        // Transition IK weight over 0.2s so the hand gradually moves onto the gun
        AnimationManager.Instance.SetIKTarget(
            _animator,
            AvatarIKGoal.RightHand,
            _gunMuzzleTransform,
            positionWeight: 1f,
            rotationWeight: 1f,
            transitionDuration: 0.2f
        );
    }

    public void StopAiming()
    {
        // Transition IK weight back to zero over 0.2s
        AnimationManager.Instance.SetIKTarget(
            _animator,
            AvatarIKGoal.RightHand,
            null,          // No target if just blending IK off
            positionWeight: 0f,
            rotationWeight: 0f,
            transitionDuration: 0.2f
        );
    }

    // In OnAnimatorIK(int layerIndex), unity will call SetIKPosition/Rotation automatically 
    // based on the weights that were set above.
}
```

* When `StartAiming` is called, over 0.2 seconds the hand IK weight goes from 0 to 1, and the hand follows `_gunMuzzleTransform`.
* When `StopAiming` is called, we blend weights back to 0, effectively disabling IK.

---

### 4.2 Procedural Animation System

#### 4.2.1 `void AddProceduralMovement(Animator animator, string parameterName, float amplitude = 1f, float frequency = 1f, float phaseOffset = 0f)`

* **Purpose**: Drive a float parameter procedurally using a sine wave:
  `value(t) = sin((timer + phaseOffset) × frequency × 2π) × amplitude`. This can be used for subtle breathing, idle head bob, or rhythmic foot placement via blend trees.
* **Parameters**

  * `animator`: The `Animator` whose parameter to modify each frame.
  * `parameterName`: Name of the float parameter. Must exist (else nothing happens).
  * `amplitude`: Maximum absolute float value.
  * `frequency`: Oscillation frequency (cycles per second).
  * `phaseOffset`: Initial offset for the sine wave, in seconds.

```csharp
public void AddProceduralMovement(
    Animator animator,
    string parameterName,
    float amplitude = 1f,
    float frequency = 1f,
    float phaseOffset = 0f)
{
    if (!ValidateAnimator(animator)) return;

    StartCoroutine(ProceduralMovementRoutine(
        animator,
        parameterName,
        amplitude,
        frequency,
        phaseOffset
    ));
}

private IEnumerator ProceduralMovementRoutine(
    Animator animator,
    string parameterName,
    float amplitude,
    float frequency,
    float phaseOffset)
{
    float timer = phaseOffset;
    while (animator != null && animator.gameObject.activeInHierarchy)
    {
        float value = Mathf.Sin(timer * frequency * 2 * Mathf.PI) * amplitude;
        animator.SetFloat(parameterName, value);
        timer += Time.deltaTime;
        yield return null;
    }
}
```

###### Real-World Example

**Scenario**: You have a “Breath” parameter in your animator that controls a subtle chest‐expansion blend. You want the character to “breathe” continuously.

```csharp
public class CharacterBreathing : MonoBehaviour
{
    [SerializeField] private Animator _animator;

    void Start()
    {
        // Sine wave with amplitude 0.2 at 0.25 Hz (one breath every 4 seconds)
        AnimationManager.Instance.AddProceduralMovement(
            _animator,
            "Breath",
            amplitude: 0.2f,
            frequency: 0.25f,
            phaseOffset: 0f
        );
    }
}
```

* This will continuously update the `Breath` parameter from –0.2 → +0.2 in a 4s cycle, driving a chest‐expansion blend tree or mask.

---

### 4.3 Root Motion Integration

#### 4.3.1 `void EnableRootMotion(Animator animator, bool applyPosition = true, bool applyRotation = true, float maxVelocity = 5f)`

* **Purpose**: Enable root motion on an `Animator` and then manually apply the delta position and rotation to a `CharacterController` (if present), clamped by `maxVelocity × deltaTime`. This gives you fine control over root motion usage, ensuring no teleportation if animations have high root speeds.

* **Parameters**

  * `animator`: The `Animator` with root motion enabled in its states.
  * `applyPosition`: Whether to move a `CharacterController` by root position.
  * `applyRotation`: Whether to rotate the `CharacterController` by root rotation.
  * `maxVelocity`: Maximum allowed speed (units per second).

* **Implementation**:

  1. Validate `animator`.
  2. Set `animator.applyRootMotion = true`.
  3. Start `RootMotionControlRoutine`, which:

     * Caches `previousPosition = animator.rootPosition` and `previousRotation = animator.rootRotation`.
     * Every frame, calculate `deltaPosition = animator.rootPosition – previousPosition` and `deltaRotation = animator.rootRotation × Inverse(previousRotation)`.
     * If there is a `CharacterController` attached:

       * For position: clamp magnitude of `deltaPosition` to `maxVelocity × Time.deltaTime` and call `controller.Move(motion)`.
       * For rotation: convert `deltaRotation` to `(angle, axis)` via `ToAngleAxis` and call `controller.transform.Rotate(axis, angle × Time.deltaTime)`.
     * Update `previousPosition`/`previousRotation`.

```csharp
public void EnableRootMotion(
    Animator animator,
    bool applyPosition = true,
    bool applyRotation = true,
    float maxVelocity = 5f)
{
    if (!ValidateAnimator(animator)) return;

    animator.applyRootMotion = true;
    StartCoroutine(RootMotionControlRoutine(
        animator,
        applyPosition,
        applyRotation,
        maxVelocity
    ));
}

private IEnumerator RootMotionControlRoutine(
    Animator animator,
    bool applyPosition,
    bool applyRotation,
    float maxVelocity)
{
    Vector3 previousPosition = animator.rootPosition;
    Quaternion previousRotation = animator.rootRotation;

    while (animator != null && animator.applyRootMotion)
    {
        Vector3 deltaPosition = animator.rootPosition - previousPosition;
        Quaternion deltaRotation = animator.rootRotation * Quaternion.Inverse(previousRotation);

        if (animator.TryGetComponent<CharacterController>(out var controller))
        {
            if (applyPosition)
            {
                Vector3 motion = Vector3.ClampMagnitude(
                    deltaPosition,
                    maxVelocity * Time.deltaTime
                );
                controller.Move(motion);
            }

            if (applyRotation)
            {
                deltaRotation.ToAngleAxis(out float angle, out Vector3 axis);
                controller.transform.Rotate(axis, angle * Time.deltaTime);
            }
        }

        previousPosition = animator.rootPosition;
        previousRotation = animator.rootRotation;
        yield return null;
    }
}
```

###### Real-World Example

**Scenario**: You have a “Roll” animation with significant forward root motion. You want to apply it to a high‐speed CharacterController but prevent the character from exceeding 8 units/s.

```csharp
public class PlayerRoll : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    private CharacterController _cc;

    private void Start()
    {
        _cc = GetComponent<CharacterController>();
    }

    public void PerformRoll()
    {
        // Enable root motion; clamp to 8 units/s
        AnimationManager.Instance.EnableRootMotion(_animator, applyPosition: true, applyRotation: true, maxVelocity: 8f);

        // Play roll animation
        AnimationManager.Instance.PlayState(_animator, "Roll", transitionDuration: 0.1f, layer: 0);
    }
}
```

* Once “Roll” begins, the `RootMotionControlRoutine` will apply root motion via `_cc.Move()`, but clamp the velocity to at most 8 stuff/second.

---

### 4.4 Animation Blending System

#### 4.4.1 `void BlendAnimations(Animator animator, string stateA, string stateB, float blendValue, float blendTime = 0.5f)`

* **Purpose**: Blend between two animation states (`stateA` and `stateB`) by controlling a float blend parameter. This assumes your Animator Controller has a float parameter named “Blend” that drives conditional transitions or a special blend tree. Over `blendTime`, this routine cross‐fades from `stateA` to `stateB` proportionally to `blendValue` each frame.

* **Parameters**

  * `animator`: The `Animator`.
  * `stateA`: First state in the blend.
  * `stateB`: Second state in the blend.
  * `blendValue`: Final blend amount (0–1) indicating how much of `stateB` vs `stateA`.
  * `blendTime`: Duration of the blend.

* **Implementation**:

  1. Validate `animator`.
  2. Start `AnimationBlendRoutine` which, over `duration`, each frame:

     * Read current `Blend` parameter value.
     * Interpolate `value = Mathf.Lerp(currentBlend, targetBlend, t)`.
     * Do `animator.CrossFade(stateA, 0f, 0, 1 – value)` (normalized time = 1 – `value`).
     * Do `animator.CrossFade(stateB, 0f, 0, value)` (normalized time = `value`).
     * After `duration`, the cross‐fade is effectively held at the final weights.

```csharp
public void BlendAnimations(
    Animator animator,
    string stateA,
    string stateB,
    float blendValue,
    float blendTime = 0.5f)
{
    if (!ValidateAnimator(animator)) return;

    StartCoroutine(AnimationBlendRoutine(
        animator,
        stateA,
        stateB,
        blendValue,
        blendTime
    ));
}

private IEnumerator AnimationBlendRoutine(
    Animator animator,
    string stateA,
    string stateB,
    float targetBlend,
    float duration)
{
    float currentBlend = animator.GetFloat("Blend");
    float elapsed = 0f;

    while (elapsed < duration)
    {
        float t = elapsed / duration;
        float value = Mathf.Lerp(currentBlend, targetBlend, t);
        animator.CrossFade(stateA, 0f, 0, 1 - value);
        animator.CrossFade(stateB, 0f, 0, value);

        elapsed += Time.deltaTime;
        yield return null;
    }
    // Force final crossfade at the end
    animator.CrossFade(stateA, 0f, 0, 1 - targetBlend);
    animator.CrossFade(stateB, 0f, 0, targetBlend);
}
```

###### Real-World Example

**Scenario**: Your character has two states “Walk” and “Run” that you want to blend smoothly based on a “RunFactor” parameter. Instead of building a blend tree, you manually crossfade.

```csharp
public class RunBlendController : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    private float _runFactor = 0f; // 0 = Walk, 1 = Run

    void Update()
    {
        // Suppose input axis controls run factor
        float input = Input.GetAxis("Vertical"); // 0–1
        if (Mathf.Abs(input - _runFactor) > 0.01f)
        {
            _runFactor = input;
            // Blend from "Walk" to "Run" over 0.2s based on input
            AnimationManager.Instance.BlendAnimations(
                _animator,
                "Walk",
                "Run",
                _runFactor,
                blendTime: 0.2f
            );
        }
    }
}
```

* As the player pushes forward, `_runFactor` goes from 0 to 1. We call `BlendAnimations` each time input changes enough. The routine cross‐fades “Walk” vs “Run” proportionally.

---

### 4.5 Animation Sequence System

The sequence system allows you to queue up a list of animations, each with its own transition, delay, and optional callback, and then play them one after the other. This is especially useful for choreographed action sequences (e.g., executing a combo of attacks, performing a series of emotes, etc.)

#### 4.5.1 `public class AnimationSequence`

* **Purpose**: Represents a single step in a sequence.
* **Fields**

  * `Animator Animator`: Which `Animator` to play on (all steps in a sequence must share the same animator).
  * `string StateName`: Name (or hash as string) of the state to transition to.
  * `float TransitionDuration`: How long the cross‐fade should take.
  * `int Layer`: Which layer to play the state on.
  * `float PostDelay`: How many seconds to wait *after the state and transition* before moving to the next step.
  * `Action OnComplete`: Callback to fire after this step finishes (optional).

```csharp
public class AnimationSequence
{
    public Animator Animator;
    public string StateName;
    public float TransitionDuration;
    public int Layer;
    public float PostDelay;
    public Action OnComplete;
}
```

#### 4.5.2 `void PlaySequence(Animator animator, List<AnimationSequence> sequence, bool interruptCurrent = true)`

* **Purpose**: Start playing the provided list of `AnimationSequence` steps in order, interrupting any current sequence if `interruptCurrent` is `true`.

* **Parameters**

  * `animator`: The `Animator` that all steps refer to.
  * `sequence`: `List<AnimationSequence>` to execute in order.
  * `interruptCurrent`: If `true`, stop any currently running sequence on this `animator` before starting a new one.

* **Behavior**:

  1. Validate `animator`.
  2. If `interruptCurrent` and `_activeSequences` already has a running coroutine for this `animator`, `StopCoroutine` it.
  3. Store the new `Coroutine` returned by `StartCoroutine(ProcessSequenceRoutine(animator, sequence))` in `_activeSequences[animator]`.

```csharp
public void PlaySequence(
    Animator animator,
    List<AnimationSequence> sequence,
    bool interruptCurrent = true)
{
    if (!ValidateAnimator(animator)) return;

    if (interruptCurrent && _activeSequences.TryGetValue(animator, out var existing))
    {
        StopCoroutine(existing);
    }

    _activeSequences[animator] = StartCoroutine(
        ProcessSequenceRoutine(animator, sequence)
    );
}
```

* If you want to stop a sequence prematurely:

  * Call `StopSequence(animator, executeFinalCallback)`.

#### 4.5.3 `IEnumerator ProcessSequenceRoutine(Animator animator, List<AnimationSequence> sequence)`

* **Implementation**:

  1. Loop through each `step` in `sequence`.
     a. Validate that `step.Animator == animator`; otherwise log an error and `yield break`.
     b. Call `PlayState` for that step.
     c. Determine `stateDuration = GetStateDuration(animator, step.StateName, step.Layer)`.
     d. `totalWait = Mathf.Max(step.TransitionDuration + stateDuration, step.PostDelay)`.
     e. `yield return new WaitForSeconds(totalWait)`.
     f. Invoke `step.OnComplete` if not null.
  2. After all steps, remove `animator` from `_activeSequences`.

```csharp
private IEnumerator ProcessSequenceRoutine(
    Animator animator,
    List<AnimationSequence> sequence)
{
    foreach (var step in sequence)
    {
        if (step.Animator != animator)
        {
            Debug.LogError("Sequence contains mismatched animators!");
            yield break;
        }

        PlayState(
            step.Animator,
            step.StateName,
            step.TransitionDuration,
            step.Layer
        );

        float stateDuration = GetStateDuration(
            step.Animator,
            step.StateName,
            step.Layer
        );

        float totalWait = Mathf.Max(
            step.TransitionDuration + stateDuration,
            step.PostDelay
        );

        yield return new WaitForSeconds(totalWait);

        step.OnComplete?.Invoke();
    }

    _activeSequences.Remove(animator);
}
```

#### 4.5.4 `float GetStateDuration(Animator animator, string stateName, int layer)`

* **Purpose**: If the state identified by `stateName` is currently playing on `layer`, return the remaining duration of that state (in seconds).
* **Implementation**:

  1. Fetch `stateInfo = animator.GetCurrentAnimatorStateInfo(layer)`.
  2. If `stateInfo.IsName(stateName)`, return `stateInfo.length × (1 – (stateInfo.normalizedTime % 1))`.
  3. Otherwise, return `0f`.

```csharp
private float GetStateDuration(Animator animator, string stateName, int layer)
{
    var stateInfo = animator.GetCurrentAnimatorStateInfo(layer);
    if (stateInfo.IsName(stateName))
    {
        // normalizedTime may exceed 1 if looped; so %1 gives current loop
        return stateInfo.length * (1 - stateInfo.normalizedTime % 1);
    }
    return 0;
}
```

* **Note**: Because `length` is the full clip length (in seconds) and `normalizedTime` is the overall progress (e.g., 1.3 means full clip + 30% of next loop), using modulo ensures you only wait for the remainder of the current loop.

#### 3.5.5 `void StopSequence(Animator animator, bool executeFinalCallback = false)`

* **Purpose**: Immediately halt any active sequence for `animator`. Optionally invoke the final step’s `OnComplete` callback if `executeFinalCallback` is `true` (requires extra tracking to store which callback is “last”).
* **Implementation** (in code stub, final‐callback execution is left as a comment placeholder):

```csharp
public void StopSequence(Animator animator, bool executeFinalCallback = false)
{
    if (_activeSequences.TryGetValue(animator, out var coroutine))
    {
        StopCoroutine(coroutine);
        _activeSequences.Remove(animator);

        if (executeFinalCallback)
        {
            // Would require storing the last executed callback from the sequence
            // in order to invoke it here. Implementation left as TODO.
        }
    }
}
```

###### Real-World Example

**Scenario**: You want the boss enemy to perform a three‐step attack combo:

1. **WindUp** (0.2s), no post delay.
2. **Slash** (0.15s), 0.1s post delay (for hit detection).
3. **Recovery** (0.3s), no post delay.
   After the combo, call a callback to enable the boss’s next AI action.

```csharp
public class BossCombo : MonoBehaviour
{
    [SerializeField] private Animator _animator;

    void Start()
    {
        var comboSequence = new List<AnimationManager.AnimationSequence>
        {
            new AnimationManager.AnimationSequence
            {
                Animator = _animator,
                StateName = "WindUp",
                TransitionDuration = 0.1f,
                Layer = 0,
                PostDelay = 0f,
                OnComplete = () => Debug.Log("WindUp done")
            },
            new AnimationManager.AnimationSequence
            {
                Animator = _animator,
                StateName = "Slash",
                TransitionDuration = 0.05f,
                Layer = 0,
                PostDelay = 0.1f,
                OnComplete = () => Debug.Log("Slash landed, apply damage")
            },
            new AnimationManager.AnimationSequence
            {
                Animator = _animator,
                StateName = "Recovery",
                TransitionDuration = 0.1f,
                Layer = 0,
                PostDelay = 0f,
                OnComplete = () => Debug.Log("Recovery complete, next AI")
            }
        };

        // Begin combo
        AnimationManager.Instance.PlaySequence(_animator, comboSequence);
    }

    public void InterruptCombo()
    {
        // Immediately stop combo; do not run final callback
        AnimationManager.Instance.StopSequence(_animator, executeFinalCallback: false);
    }
}
```

* The manager transitions “WindUp” → “Slash” → “Recovery,” executing each step’s `OnComplete` after its post‐delay.
* If the boss is interrupted (e.g., stunned), you call `StopSequence`, halting the routine.

---

### 4.6 Animation Events (Advanced)

In addition to `AnimationHelper.RegisterAnimationEvent`, the `AnimationManager` has its own event callback system to handle Unity’s animation events at a higher level (providing persistence flags, multiple callbacks per event name, etc.).

#### 4.6.1 `void RegisterEventCallback(Animator animator, string eventName, Action<AnimationEvent> callback, bool persistent = false)`

* **Purpose**: Register a callback for Unity’s built‐in `AnimationEvent` system. When an event with `stringParameter = eventName` is triggered, this callback will be invoked. If `persistent == false`, the callback is automatically removed after the first invocation.

* **Parameters**

  * `animator`: Target `Animator`.
  * `eventName`: The `stringParameter` to listen for in animation clips.
  * `callback`: `Action<AnimationEvent>` to invoke when the event occurs.
  * `persistent`: If `true`, keep listening even after the first fire; if `false`, remove after first.

* **Behavior**:

  1. `ValidateAnimator`.
  2. `var state = GetAnimatorState(animator)`.
  3. Save in `state.EventCallbacks[eventName] = new AnimationEventCallback { EventName = eventName, Callback = callback, IsPersistent = persistent }`.

```csharp
public void RegisterEventCallback(
    Animator animator,
    string eventName,
    Action<AnimationEvent> callback,
    bool persistent = false)
{
    if (!ValidateAnimator(animator)) return;

    var state = GetAnimatorState(animator);
    state.EventCallbacks[eventName] = new AnimationEventCallback
    {
        EventName = eventName,
        Callback = callback,
        IsPersistent = persistent
    };
}
```

#### 4.6.2 `private void OnAnimationEvent(AnimationEvent evt)`

* **Purpose**: Internal method that should be invoked when Unity’s animation system delivers any `AnimationEvent`.
* **Behavior**:

  1. Loop through all `AnimatorState` objects in `_animatorStates.Values`.
  2. If `state.EventCallbacks` contains an entry for `evt.stringParameter`, call `callback.Callback(evt)`.
  3. If `callback.IsPersistent == false`, remove that entry from `state.EventCallbacks`.

```csharp
private void OnAnimationEvent(AnimationEvent evt)
{
    foreach (var state in _animatorStates.Values)
    {
        if (state.EventCallbacks.TryGetValue(evt.stringParameter, out var callback))
        {
            callback.Callback?.Invoke(evt);

            if (!callback.IsPersistent)
            {
                state.EventCallbacks.Remove(evt.stringParameter);
            }
        }
    }
}
```

**Note**: To get Unity to call `AnimationManager.OnAnimationEvent`, you must either:

* Add an `AnimationEventHelper` to the `animator.gameObject` that calls `AnimationManager.Instance.OnAnimationEvent(evt)` on each event, or
* Place the function in a script on the same GameObject as the `Animator`, with the function signature `public void OnAnimationEvent(AnimationEvent evt)` and have the clips’ events call it directly.

The code above implies you attach `AnimationEventHelper` to gather events into the manager, but you could also refactor to call `AnimationManager.Instance.OnAnimationEvent(evt)` from `AnimationEventHelper.TriggerAnimationEvent`.

###### Real-World Example

**Scenario**: Your boss has an “Enrage” event baked into its “RageBuild” animation at 75%. You want to register a callback that raises a damage multiplier exactly once when that event fires.

```csharp
public class BossAI : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    private bool _isEnraged = false;

    void Start()
    {
        // Register for a single event "Enrage"
        AnimationManager.Instance.RegisterEventCallback(
            _animator,
            "Enrage",
            OnEnrageEvent,
            persistent: false
        );
    }

    private void OnEnrageEvent(AnimationEvent evt)
    {
        if (!_isEnraged)
        {
            _isEnraged = true;
            // Increase damage or other stats
            Debug.Log("Boss has become enraged!");
        }
    }
}
```

* When “RageBuild” clip’s AnimationEvent with `stringParameter = "Enrage"` triggers, `OnEnrageEvent` runs and sets `_isEnraged=true`. Because `persistent=false`, the callback auto‐removes itself.

---

### 4.7 Animation Recording (Stub)

Parts of the code are commented out but outline how you might:

* **`StartRecording(Animator animator)`**
  Begin capturing `Time.time`, parameters, position, rotation, IK data each frame for this animator by setting `state.IsRecording = true` and clearing `state.RecordedFrames`.

* **`StopRecording(Animator animator, string saveKey)`**
  Stops recording, then serializes `state.RecordedFrames` to JSON (via a hypothetical `JsonHelper`) and stores it in `PlayerPrefs` under `"RecordedAnim_{saveKey}"`.

* **`PlayRecordedAnimation(Animator animator, string saveKey, bool loop)`**
  Loads recorded JSON from `PlayerPrefs`, deserializes to a list of `AnimationRecordFrame`, then iterates frames in a coroutine to replay parameter values, transform position, rotation, and IK values.

```csharp
public void StartRecording(Animator animator)
{
    if (!ValidateAnimator(animator)) return;

    var state = GetAnimatorState(animator);
    state.IsRecording = true;
    state.RecordStartTime = Time.time;
    state.RecordedFrames.Clear();
}

// (StopRecording and PlayRecordedAnimation are commented out, showing intended logic)
```

* **Use Case**:

  * You want to record the exact movement and parameter changes of a complex animation for replay or network sync.
  * At runtime, call `StartRecording(characterAnimator)`. Each frame inside `OnAnimatorIK` or `LateUpdate`, you’d check `state.IsRecording` and push a new `AnimationRecordFrame` onto `state.RecordedFrames` with current camera‐space transform, IK positions, parameter values, etc.
  * Later, call `StopRecording` to save. To replay, call `PlayRecordedAnimation(animator, saveKey, loop: false)`.

---

### 4.8 Performance Optimization

#### 4.8.1 `void CacheAnimatorController(Animator animator, string key, RuntimeAnimatorController controller)`

* **Purpose**: Store a reference to a `RuntimeAnimatorController` under a `key` in `state.CachedControllers`. Useful if you need to switch quickly between multiple controllers at runtime (e.g., different weapon sets).
* **Implementation**:

  1. Validate `animator`.
  2. `var state = GetAnimatorState(animator)`.
  3. `state.CachedControllers[key] = controller`.

```csharp
public void CacheAnimatorController(
    Animator animator,
    string key,
    RuntimeAnimatorController controller)
{
    if (!ValidateAnimator(animator)) return;

    var state = GetAnimatorState(animator);
    state.CachedControllers[key] = controller;
}
```

#### 4.8.2 `void SwitchToCachedController(Animator animator, string key)`

* **Purpose**: If a `RuntimeAnimatorController` was previously cached under `key`, assign it immediately to `animator.runtimeAnimatorController`.
* **Implementation**:

  1. Validate `animator`.
  2. `var state = GetAnimatorState(animator)`.
  3. If `state.CachedControllers.TryGetValue(key, out controller)`, do `animator.runtimeAnimatorController = controller`.

```csharp
public void SwitchToCachedController(
    Animator animator,
    string key)
{
    if (!ValidateAnimator(animator)) return;

    var state = GetAnimatorState(animator);
    if (state.CachedControllers.TryGetValue(key, out var controller))
    {
        animator.runtimeAnimatorController = controller;
    }
}
```

###### Real-World Example

**Scenario**: You want two different character classes (Warrior, Mage) to share an `Animator` component but swap controllers on the fly. At character spawn:

```csharp
public class CharacterLoader : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    [SerializeField] private RuntimeAnimatorController _warriorController;
    [SerializeField] private RuntimeAnimatorController _mageController;

    void Start()
    {
        // Pre‐cache both controllers
        AnimationManager.Instance.CacheAnimatorController(_animator, "Warrior", _warriorController);
        AnimationManager.Instance.CacheAnimatorController(_animator, "Mage", _mageController);

        // Suppose player chose "Mage"
        AnimationManager.Instance.SwitchToCachedController(_animator, "Mage");
    }

    public void SwitchToWarrior()
    {
        AnimationManager.Instance.SwitchToCachedController(_animator, "Warrior");
    }
}
```

#### 4.8.3 `void SetCullingMode(Animator animator, bool enableCulling, float cullDistance = 50f)`

* **Purpose**: Enable or disable culling of an `Animator` for performance. When `enableCulling == true`, sets `animator.cullingMode = AnimatorCullingMode.CullUpdateTransforms`, which stops updating transforms when off‐screen. Also sets `renderer.allowOcclusionWhenDynamic` accordingly if `Animator` includes a `Renderer`.
* **Parameters**

  * `animator`: Target `Animator`.
  * `enableCulling`: If `true`, only update transforms when on‐screen. If `false`, always animate.
  * `cullDistance`: Not explicitly used in code, but you could incorporate this by enabling/disabling `Renderer` based on distance if you want to extend functionality.

```csharp
public void SetCullingMode(
    Animator animator,
    bool enableCulling,
    float cullDistance = 50f)
{
    if (!ValidateAnimator(animator)) return;

    animator.cullingMode = enableCulling ?
        AnimatorCullingMode.CullUpdateTransforms :
        AnimatorCullingMode.AlwaysAnimate;

    var renderer = animator.GetComponent<Renderer>();
    if (renderer)
    {
        renderer.allowOcclusionWhenDynamic = enableCulling;
    }
}
```

###### Real-World Example

**Scenario**: In a large open world, you want NPC animators far from the camera to stop updating (cull), but keep close ones animating. You might check distance in another manager:

```csharp
public class NPCCullingController : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    [SerializeField] private Transform _playerCamera;

    void Update()
    {
        float dist = Vector3.Distance(_animator.transform.position, _playerCamera.position);
        bool shouldCull = dist > 60f;
        AnimationManager.Instance.SetCullingMode(_animator, enableCulling: shouldCull);
    }
}
```

#### 4.8.4 (Commented) `void OptimizeAnimator(...)`

* **Purpose** (commented out): Adjust root motion, update mode (`Fixed` vs `Normal`), culling, and disable unused components (e.g., making attached `Rigidbody` kinematic if root motion is disabled).
* **Potential Implementation**:

  * Validate `animator`.
  * `animator.applyRootMotion = !disableRootMotion`.
  * `animator.updateMode = useFixedUpdate ? AnimatorUpdateMode.Fixed : AnimatorUpdateMode.Normal`.
  * Call `SetCullingMode(animator, enableCulling)`.
  * If the `Animator` has a `Rigidbody` and `disableRootMotion == true`, do `rigidbody.isKinematic = true`.

Because it’s commented out, you can uncomment or incorporate these lines as needed.

---

## 5. Putting It All Together

### 5.1 Typical Initialization and Usage Workflow

1. **At runtime**, some other script calls `AnimationManager.Instance` (or `AnimationHelper` static methods) to guarantee initialization.
2. **Register any states** you want to track for enter/exit events:

   ```csharp
   AnimationManager.Instance.OnStateEntered += HandleStateEntry;
   AnimationManager.Instance.OnStateExited += HandleStateExit;
   ```
3. **Play animations** using `AnimationManager.Instance.PlayState(animator, "Run", 0.2f, 0)`.
4. **Blend parameters** with `AnimationManager.Instance.LerpParameter(animator, "Speed", 1.0f, 0.5f)`.
5. **Attach IK** targets via `AnimationManager.Instance.SetIKTarget(animator, AvatarIKGoal.LeftHand, handTargetTransform)`.
6. **Enable root motion** via `AnimationManager.Instance.EnableRootMotion(animator)` if needed.
7. **Create sequences** of actions using `AnimationManager.Instance.PlaySequence(...)`.
8. **Register animation events** using `AnimationHelper.RegisterAnimationEvent` or `AnimationManager.RegisterEventCallback`.
9. **Record or Playback** animations if needed (once you implement `StopRecording`, `PlayRecordedAnimation`).
10. **Cache controllers** if you need quick switching, then call `SwitchToCachedController`.
11. **Adjust culling** dynamically with `AnimationManager.SetCullingMode`.
12. **When no longer needed** (e.g., object destroyed or returning to pool), call:

    ```csharp
    AnimationManager.Instance.StopSequence(animator);
    AnimationHelper.Cleanup(animator);
    // Optionally also remove from _animatorStates via Destroy.
    ```

---

## 6. Summary of Important Methods & Examples

| Method                                                                                           | Purpose                                                                                                             | Example Use Case                                                                                               |
| ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **AnimationHelper.PlayAnimation**<br>`(Animator, string, int=0, float=0, float=1)`               | Safely play a state by name on an `Animator`, optionally with speed & start-time.                                   | *Play “Jump” state halfway (0.5f) at half speed.*                                                              |
| **AnimationHelper.CrossFadeAnimation**<br>`(Animator, string, float, int=-1)`                    | Cross-fade smoothly from current state to new state over `duration`.                                                | *When player presses “C”, cross-fade from “Run” to “Slide” over 0.2s.*                                         |
| **AnimationHelper.BlendLayer**<br>`IEnumerator (Animator, int, float, float)`                    | Coroutine to interpolate a layer’s weight from current to target over `duration`.                                   | *Blend upper-body aiming layer (index = 1) from 0 → 1 in 0.3s when equipping weapon.*                          |
| **AnimationHelper.SetAnimationSpeed**<br>`(Animator, string, float)`                             | Set speed multiplier for a given state; if currently playing, updates speed immediately.                            | *Apply a 1.5× speed to “CastingSpell” when a “Haste” buff is active.*                                          |
| **AnimationHelper.RestoreAnimationSpeed**<br>`(Animator, string)`                                | Reset speed to 1.0 for a given state and remove stored speed record.                                                | *After “Haste” buff ends, restore “CastingSpell” to normal speed.*                                             |
| **AnimationHelper.IsPlaying**<br>`(Animator, string, int=0)`                                     | Check if a particular state is actively playing on a layer.                                                         | *Prevent re-triggering “Jump” if it’s already playing.*                                                        |
| **AnimationHelper.GetNormalizedTime**<br>`(Animator, int=0)`                                     | Return normalized progress (0–1) of the current clip.                                                               | *Spawn particle when “SwordSwing” reaches 80%.*                                                                |
| **AnimationHelper.WaitForAnimationComplete**<br>`IEnumerator (Animator, string, int=0)`          | Coroutine that yields until the specified state has finished playing.                                               | *After “Attack” state completes, return to “Idle”.*                                                            |
| **AnimationHelper.RegisterAnimationEvent**<br>`(Animator, Action<string>)`                       | Add an `AnimationEventHelper` component to an `Animator` and subscribe to its `OnAnimationEvent`.                   | *Play footstep sound when “Footstep” event triggers in running animation.*                                     |
| **AnimationHelper.UnregisterAnimationEvent**<br>`(Animator, Action<string>)`                     | Unsubscribe from animation events and leave the `AnimationEventHelper` on the GameObject.                           | *Cleanup event listeners before object destruction.*                                                           |
| **AnimationHelper.SetParameter**<br>`(Animator, string, object)`                                 | Safely set parameters (Bool, Int, Float, Trigger) only if types match.                                              | *Set “IsDead” trigger to true when enemy health reaches 0.*                                                    |
| **AnimationHelper.Cleanup**<br>`(Animator)`                                                      | Remove all stored speeds and destroy any attached `AnimationEventHelper`.                                           | *When pooling/destroying enemy, call this to avoid leaks.*                                                     |
| **AnimationManager.PlayState**<br>`(Animator, object, float=-1, int=0)`                          | High‐level cross-fade to a new state (string or hash) with default or specified transition time.                    | *Invoke “Attack” with 0.2s cross-fade; if already transitioning, cancels previous transition.*                 |
| **AnimationManager.LerpParameter**<br>`(Animator, string, float, float, Action)`                 | Smoothly interpolate a float parameter over time, with optional completion callback.                                | *Ramp “Speed” from 1.0→0.0 over 0.3s when player begins walking.*                                              |
| **AnimationManager.ApplyOverrides**<br>`(Animator, List<AnimationOverride>)`                     | Create an `AnimatorOverrideController` at runtime to swap out specified clips.                                      | *Swap out “Sword\_Swing” with“Metal\_Swing” when equipping a metal weapon.*                                    |
| **AnimationManager.SetIKTarget**<br>`(Animator, AvatarIKGoal, Transform, float, float, float)`   | Coroutine to smoothly raise an IK effector from 0→given weights and update its target each frame.                   | *Aim left hand to hold a gun over 0.2s by updating IK position/rotation.*                                      |
| **AnimationManager.AddProceduralMovement**<br>`(Animator, string, float, float, float)`          | Coroutine to drive a float parameter with a sine wave (amplitude, frequency, phase).                                | *Make character “breathe” by oscillating “Breath” parameter with amplitude 0.2 at 0.25 Hz.*                    |
| **AnimationManager.EnableRootMotion**<br>`(Animator, bool, bool, float)`                         | Enable root motion and apply it to `CharacterController`, clamped by `maxVelocity`.                                 | *Use “Roll” animation’s root motion to move a character at up to 8 units/s.*                                   |
| **AnimationManager.BlendAnimations**<br>`(Animator, string, string, float, float)`               | Coroutine to cross‐fade between two named states based on a blend parameter value over a duration.                  | *Blend “Walk” and “Run” states based on an input‐driven parameter over 0.2s.*                                  |
| **AnimationManager.PlaySequence**<br>`(Animator, List<AnimationSequence>, bool=true)`            | Queue a series of animation steps (with cross-fade, delays, callbacks), canceling any existing sequence if desired. | *Play boss combo: “WindUp”→“Slash”→“Recovery” with respective durations; after each step invoke damage logic.* |
| **AnimationManager.StopSequence**<br>`(Animator, bool=false)`                                    | Immediately cancel the running sequence for an `Animator`. Optionally invoke final callback.                        | *Stop current combo mid‐air if boss is stunned.*                                                               |
| **AnimationManager.RegisterEventCallback**<br>`(Animator, string, Action<AnimationEvent>, bool)` | Register for a Unity animation event by name, auto‐removing callback after first call if `persistent == false`.     | *Register a “FireBullet” event to instantiate a bullet prefab at the muzzle.*                                  |
| **AnimationManager.CacheAnimatorController**<br>`(Animator, string, RuntimeAnimatorController)`  | Store a reference to a controller under a key for quick swapping at runtime.                                        | *Cache both warrior and mage controllers, then call `SwitchToCachedController` when the player changes class.* |
| **AnimationManager.SwitchToCachedController**<br>`(Animator, string)`                            | Immediately assign a previously cached controller to the `Animator`.                                                | *Switch to “Mage” animation controller upon selecting that class.*                                             |
| **AnimationManager.SetCullingMode**<br>`(Animator, bool, float)`                                 | Toggle culling so the `Animator` only updates when visible, and adjust `Renderer.allowOcclusionWhenDynamic`.        | *If NPC is >60 units from camera, enable culling to save performance.*                                         |

---

## 7. Best Practices & Tips

1. **Always Validate Your Animator**:
   Use `AnimationManager.Instance.ValidateAnimator(animator)` or check `animator != null && animator.isActiveAndEnabled` before attempting any operations. This avoids runtime null‐refs or disabled component errors.

2. **Use Hashes for Performance**:
   If you call `PlayState`, `CrossFade`, or `IsPlaying` frequently, pass an `int` hash (via `Animator.StringToHash("StateName")`) instead of a string. This eliminates string‐lookup overhead.

3. **Keep Track of Coroutines**:
   Since `AnimationManager` stores `ActiveCoroutine` per‐animator‐state, you can ensure you don’t inadvertently run multiple conflicting coroutines on the same `Animator`. If you want multiple parameter lerps at once, you must refactor to allow separate coroutines instead of canceling `ActiveCoroutine`.

4. **Parameter Safety**:
   Always set parameters through `SetParameter` or `LerpParameter` so that you don’t accidentally set the wrong type (e.g., sending a `float` to an “Int” parameter). This method verifies at runtime and logs clear errors.

5. **Blend Trees vs. Manual Cross‐Fade**:
   If you only need to blend two or three animations based on a float, consider using a Blend Tree in the Animator Controller. However, `BlendAnimations` is helpful if you want to control blending procedurally or your Animator Controller does not already have a blend tree.

6. **Event Handling**:

   * **Static vs. Manager**:
     You can either listen to events with the static `AnimationHelper.RegisterAnimationEvent`, which internally creates an `AnimationEventHelper` component. Alternatively, if you want all animation events to funnel through a single central manager, you can call `AnimationManager.RegisterEventCallback`.
   * **Persistent vs. One‐Shot**:
     If you only need to respond once, set `persistent = false` so it’s automatically removed.

7. **Cleaning Up**:
   Always call `AnimationHelper.Cleanup(animator)` when an object is destroyed or returned to a pooling system. This removes any stray callbacks or dynamic components.

8. **Recording & Playback**:
   The recording system is partly stubbed out. If you need to record complex motion for replays or network sync, you would:

   * In `StartRecording`, set `IsRecording = true`.
   * In `OnAnimatorIK` or a `LateUpdate`, if `IsRecording` is `true`, collect parameters, `transform.position`, `transform.rotation`, and IK data into a new `AnimationRecordFrame` and enqueue it in `state.RecordedFrames`.
   * In `StopRecording`, serialize `state.RecordedFrames` to a JSON string via your favorite JSON utility and store to `PlayerPrefs` or another data store.
   * In `PlayRecordedAnimation`, load the JSON, deserialize, and drive a coroutine that replays parameter values and transforms frame by frame.

9. **Performance Tips**:

   * Use `SetCullingMode` to disable animator updates when off‐screen.
   * If you have static objects or NPCs far from the camera, disable root motion or set `updateMode` to `AnimatorUpdateMode.UnscaledTime` if you don’t need them in sync with physics.
   * Cache animator controllers if switching often to avoid reloading or instantiating new `AnimatorOverrideController` objects frequently.

---

## 8. Conclusion

This comprehensive helper system streamlines nearly every aspect of runtime animation management in Unity:

* **AnimationHelper (Static)**

  * Quick methods for playing, cross‐fading, blending layers, speeding up states, parameter safety, event registration, and cleanup.

* **AnimationManager (MonoBehaviour Singleton)**

  * Centralized state polling, robust transition routines, parameter lerps, runtime clip overrides, advanced IK, procedural parameter animation, root motion integration, manual blending, sequenced animations, event callback management, recording stubs, and performance optimizations like culling and controller caching.

By using these utilities, you can avoid repetitive boilerplate code, keep your animation logic organized, and focus on high‐level gameplay or feature requirements rather than low‐level `Animator` calls.

Feel free to adapt or extend this system for your project’s unique needs—whether that’s adding a custom state‐name lookup for human‐readable state names, implementing the commented‐out recording playback, or integrating more sophisticated optimization logic.

Happy animating!
