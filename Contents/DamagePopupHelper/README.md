Below is a complete breakdown of **DamagePopupHelper**—a static utility class that spawns and animates floating damage/heal numbers in world space. It leverages a generic object-pooling system (via `ObjectPoolHelper`), TextMeshPro for crisp text, DOTween for smooth animations, and a simple “billboard” component so popups always face the camera. Each method and field is explained in detail, followed by practical, copy-and-paste examples showing how you might incorporate these in a real Unity project.

---

## Table of Contents

1. [Overview & Purpose](#overview--purpose)
2. [Internal Constants & Fields](#internal-constants--fields)
3. [Initialization](#initialization)

   * 3.1. `Initialize()`
   * 3.2. `CreateDamagePopupPrefab()`
4. [Displaying a Single Damage or Heal Number](#displaying-a-single-damage-or-heal-number)

   * 4.1. `ShowDamage(...)`
5. [Displaying Combo Hits](#displaying-combo-hits)

   * 5.1. `ShowCombo(...)`
6. [Billboard Component](#billboard-component)
7. [Putting It All Together: Real-World Usage](#putting-it-all-together-real-world-usage)

   * 7.1. Bootstrapping at Game Start
   * 7.2. Showing Damage in Combat Scripts
   * 7.3. Healing Popups
   * 7.4. Critical Hits & Color Coding
   * 7.5. Combo Attacks
8. [Customization & Tips](#customization--tips)
9. [Summary of Public API](#summary-of-public-api)

---

## 1. Overview & Purpose

```csharp
/// <summary>
/// A specialized helper for displaying damage numbers and other floating text in world space.
/// Integrates with ObjectPoolHelper for efficient popup management and supports various
/// animation styles and color coding.
/// </summary>
public static class DamagePopupHelper { … }
```

* **Goal**: Whenever a character takes damage (or receives healing, critical hits, or elemental bursts), display a floating number above their head (or at any world position) that briefly animates (pops, floats upward, and fades).
* **Key Features**:

  1. **Object Pooling** via `ObjectPoolHelper` to avoid instantiating/destroying dozens of TMP GameObjects per second.
  2. **Animation** using DOTween (punch scale, upward movement, fade out).
  3. **Color Coding** for normal damage, critical hits, healing, and elemental damage.
  4. **Combo Support** (multiple popups in quick succession).
  5. **Billboard** component ensures the text always faces the camera, no matter its rotation.

---

## 2. Internal Constants & Fields

```csharp
// Pool key for damage popup prefabs
private const string POOL_KEY = "DamagePopup";

// Animation settings
private static readonly float PopupDuration = 1f;
private static readonly float PopupFloatDistance = 1f;
private static readonly float PopupSpreadRadius = 0.5f;
private static readonly float CriticalScale = 1.5f;

// Color settings
private static readonly Color CriticalColor = new Color(1f, 0.2f, 0.2f);
private static readonly Color NormalColor = Color.white;
private static readonly Color HealColor = new Color(0.2f, 1f, 0.2f);
private static readonly Color ElementalColors = new Color(0.2f, 0.6f, 1f);
```

1. **`POOL_KEY`**

   * A unique string (`"DamagePopup"`) used to identify this specific pool in the global `ObjectPoolHelper` system.
   * All popups fetched or returned will reference this key.

2. **Animation Settings**

   * **`PopupDuration = 1f`**: Total time (in seconds) for the float + fade animation.
   * **`PopupFloatDistance = 1f`**: How far (in world units) the popup moves upward from its starting position.
   * **`PopupSpreadRadius = 0.5f`**: Radius (in world units) for a random “jitter” in XZ-plane so multiple popups do not stack exactly on top of each other.
   * **`CriticalScale = 1.5f`**: (Note: Although declared, `CriticalScale` is not actually used in the provided code—it suggests an intention to scale critical hits larger, which you could implement by applying a larger initial scale via DOTween.)

3. **Color Settings**

   * **`CriticalColor`**: A reddish tint (RGB = 1.0, 0.2, 0.2) for critical‐hit numbers.
   * **`NormalColor`**: Pure white for standard damage.
   * **`HealColor`**: A greenish tint (RGB = 0.2, 1.0, 0.2) for healing popups (shown with a “+”).
   * **`ElementalColors`**: A bluish color (RGB = 0.2, 0.6, 1.0) reserved for elemental damage—unused in the sample but provided as a placeholder for future expansion (e.g., fire vs. ice).

---

## 3. Initialization

Before spawning any popups, you **must** call `DamagePopupHelper.Initialize()` once—ideally at game startup or level load—so that the pool of popup GameObjects is set up.

### 3.1 Method: `public static async Task Initialize()`

```csharp
/// <summary>
/// Initialize the damage popup system
/// </summary>
public static async Task Initialize()
{
    var prefab = CreateDamagePopupPrefab();
    await ObjectPoolHelper.InitializePoolAsync(POOL_KEY, prefab, new ObjectPoolHelper.PoolSettings
    {
        InitialSize = 20,
        MaxSize = 100,
        ExpandBy = 10,
        AutoExpand = true
    });
}
```

**What It Does**:

1. **Create a “Template” Prefab** via `CreateDamagePopupPrefab()` (described below).

2. **Call `ObjectPoolHelper.InitializePoolAsync(...)`** with:

   * **`POOL_KEY`** (`"DamagePopup"`)
   * **`prefab`**: the `GameObject` returned by `CreateDamagePopupPrefab()`.
   * **`PoolSettings`**:

     * `InitialSize = 20`: Pre-instantiate 20 popup GameObjects to avoid hitches at runtime.
     * `MaxSize = 100`: Don’t allow more than 100 instances in the pool at once.
     * `ExpandBy = 10`: If the pool runs out, create 10 more at a time.
     * `AutoExpand = true`: Automatically grow the pool (by `ExpandBy`) whenever all pre-allocated instances are in use.

3. **`await`** the pool initialization—this is asynchronous, so your caller can await. Until this completes, calling `ShowDamage` will probably return `null` because the pool isn’t ready.

**Notes**:

* The `CreateDamagePopupPrefab()` method constructs the `GameObject` entirely in code—no reliance on a Unity-serialized prefab asset.
* You must call **exactly once** before any `ShowDamage` or `ShowCombo` calls. If you skip initialization, `ObjectPoolHelper.Get(...)` will fail to locate a pool and return `null`.

---

### 3.2 Method: `private static GameObject CreateDamagePopupPrefab()`

```csharp
private static GameObject CreateDamagePopupPrefab()
{
    var go = new GameObject("DamagePopup");
    
    // 1. Add TextMeshPro component
    var tmp = go.AddComponent<TMP_Text>();
    tmp.font = Resources.Load<TMP_FontAsset>("Fonts/Default");
    tmp.fontSize = 8;
    tmp.alignment = TextAlignmentOptions.Center;
    tmp.color = Color.white;
    
    // 2. Make it face the camera
    go.AddComponent<Billboard>();
    
    // 3. Add poolable component linking to POOL_KEY
    var poolable = go.AddComponent<PoolableObject>();
    poolable.PoolKey = POOL_KEY;

    return go;
}
```

**What It Does**:

1. **New GameObject**

   * Creates a fresh `GameObject` named `"DamagePopup"`. This is not placed in the scene yet—it’s used as the “template” for pooling.

2. **Attach `TMP_Text`**

   * Calls `AddComponent<TMP_Text>()`. In many projects, you might use `TextMeshProUGUI` or `TextMeshPro` (3D) instead—here, `TMP_Text` is a base class that is appropriate for world-space text.
   * Sets:

     * `tmp.font = Resources.Load<TMP_FontAsset>("Fonts/Default");`

       * Loads a `FontAsset` named `"Default"` from `Assets/Resources/Fonts/Default.asset`. You must ensure such an asset exists in your project’s `Resources/Fonts` folder.
     * `tmp.fontSize = 8`: A small world-space font size—tweak as needed.
     * `tmp.alignment = TextAlignmentOptions.Center`: Centered text so it reads correctly from any angle.
     * `tmp.color = Color.white`: Default color; will be overridden based on damage type.

3. **Attach `Billboard` Component**

   * Adds the helper `Billboard` (defined below) so that at runtime, each spawned popup automatically rotates to face the camera, regardless of world rotation.

4. **Attach `PoolableObject`**

   * `PoolableObject` is assumed to be a simple script (from a generic pooling package) with a public `string PoolKey` field and a `ReturnToPool()` method. By setting `PoolKey = POOL_KEY`, we allow `DamagePopupHelper` (and `ObjectPoolHelper`) to know where to recycle this instance when we’re done animating.

5. **Return**

   * The method returns this newly constructed `GameObject`. `ObjectPoolHelper.InitializePoolAsync` will clone it (and its entire component graph) to fill the initial pool.

**Important**:

* Because this is created via code, any changes to the popup UI (e.g., text font, font size, outline, shadow) must be edited here (or by assigning a different `TMP_FontAsset`). If you prefer to design the popup in the Unity Editor (with multiple components, child objects, custom animations), you can instead load a prebuilt prefab from `Resources` and pass that to `InitializePoolAsync`.

---

## 4. Displaying a Single Damage or Heal Number

### 4.1 Method: `public static async void ShowDamage(Vector3 worldPos, float amount, bool isCritical = false, bool isHeal = false)`

```csharp
/// <summary>
/// Show a damage number at the specified world position
/// </summary>
public static async void ShowDamage(
    Vector3 worldPos,
    float amount,
    bool isCritical = false,
    bool isHeal = false)
{
    // 1. Fetch a pooled instance at the given world position + no rotation
    var popup = ObjectPoolHelper.Get(POOL_KEY, worldPos, Quaternion.identity);
    if (popup == null)
        return;

    // 2. Grab the TMP_Text component and set its string & color
    var text = popup.GetComponent<TMP_Text>();
    text.text = isHeal ? $"+{amount:F0}" : $"-{amount:F0}";
    text.color = isHeal ? HealColor : (isCritical ? CriticalColor : NormalColor);

    // 3. Slight random XZ offset to avoid overlapping popups
    Vector3 randomOffset = Random.insideUnitCircle * PopupSpreadRadius;
    popup.transform.position += randomOffset;

    // 4. Build a DOTween sequence for scale, float, and fade
    var sequence = DOTween.Sequence();
    
    // 4a. Punch scale (small “pop” effect)
    sequence.Append(popup.transform.DOPunchScale(
        Vector3.one * 0.5f,  // punch magnitude
        0.2f                 // duration of punch
    ));

    // 4b. Float upward (Y increases by PopupFloatDistance over PopupDuration seconds)
    sequence.Join(popup.transform.DOMoveY(
        popup.transform.position.y + PopupFloatDistance,
        PopupDuration
    ).SetEase(Ease.OutCubic));

    // 4c. Fade text opacity to zero in the second half of the animation
    sequence.Join(text.DOFade(
        0f,
        PopupDuration * 0.5f
    ).SetDelay(PopupDuration * 0.5f));

    // 5. Wait synchronously for PopupDuration (in milliseconds) before returning to pool
    await Task.Delay((int)(PopupDuration * 1000));
    popup.GetComponent<PoolableObject>().ReturnToPool();
}
```

**Step-by-Step Explanation**:

1. **Retrieve a Pooled Instance**

   * `ObjectPoolHelper.Get(POOL_KEY, worldPos, Quaternion.identity)` attempts to fetch an available popup GameObject from the `"DamagePopup"` pool, placing it at position `worldPos` with no rotation (`Quaternion.identity`).
   * If the pool is exhausted (and cannot expand further), `Get()` may return null—so we bail out if `popup == null`.

2. **Text Setup**

   * `var text = popup.GetComponent<TMP_Text>()`: We assume each popup has a `TMP_Text` component (added by `CreateDamagePopupPrefab`).

   * `text.text = isHeal ? $"+{amount:F0}" : $"-{amount:F0}";`

     * If `isHeal == true`, prefix with “+” (e.g., “+150”). Otherwise prefix with “-” for damage (e.g., “-75”).
     * `:F0` formats the float as an integer (“no decimal places”).

   * `text.color = isHeal ? HealColor : (isCritical ? CriticalColor : NormalColor);`

     * If healing, use `HealColor` (green).
     * Else if critical, use `CriticalColor` (red).
     * Else use `NormalColor` (white).

3. **Random Spread**

   * `Random.insideUnitCircle` returns a 2D vector of magnitude ≤ 1. Multiply by `PopupSpreadRadius` (0.5) to get a random offset up to 0.5 units in XZ.
   * `popup.transform.position += randomOffset;`

     * Shifts the spawn position slightly so multiple popups at the same `worldPos` aren’t stacked exactly on top of each other, improving readability.

4. **DOTween Animation Sequence**

   * `var sequence = DOTween.Sequence();` creates a new tween sequence for chaining multiple tweens together.

   * **Punch Scale**

     ```csharp
     sequence.Append(popup.transform.DOPunchScale(
         Vector3.one * 0.5f,  // “punch” to 1.5× scale momentarily
         0.2f                 // over 0.2 seconds
     ));
     ```

     * `DOPunchScale` makes the object briefly jump to a larger scale (1 + 0.5 = 1.5×), then return to its original scale, simulating a quick “pop” effect.

   * **Float Up** (`Join` means it runs concurrently with whatever is currently at that same position in the timeline)

     ```csharp
     sequence.Join(
         popup.transform.DOMoveY(
             popup.transform.position.y + PopupFloatDistance,
             PopupDuration
         ).SetEase(Ease.OutCubic)
     );
     ```

     * Moves the popup’s Y-position from its current `y` to `y + 1.0f` over `PopupDuration` (1 second), easing out smoothly.

   * **Fade Out Text**

     ```csharp
     sequence.Join(
         text.DOFade(
             0f,
             PopupDuration * 0.5f  // fade to transparent over 0.5s
         ).SetDelay(PopupDuration * 0.5f)
     );
     ```

     * The fade waits for half the duration (0.5 s) via `SetDelay(0.5)`, then over the next 0.5 s interpolates alpha from 1 → 0.
     * Meanwhile, the object is still floating upward.

5. **Return to Pool After Animation**

   * `await Task.Delay((int)(PopupDuration * 1000))` pauses this async method for 1000 ms (1 second).
   * Once the delay is over, call `popup.GetComponent<PoolableObject>().ReturnToPool()`.

     * This typically disables the GameObject and enqueues it back into the pool for later reuse.

**Why `async void`?**

* We want the popup to animate in the background without blocking the calling code. An `async void` method “fires and forgets”—perfect for UI popups. Be careful: exceptions in an `async void` are uncaught; however, in this case, we are not awaiting anything that throws inside the method body.

---

## 5. Displaying Combo Hits

### 5.1 Method: `public static void ShowCombo(Vector3 worldPos, float[] damages, bool isCritical = false)`

```csharp
/// <summary>
/// Show damage numbers in a combo (multiple hits)
/// </summary>
public static void ShowCombo(Vector3 worldPos, float[] damages, bool isCritical = false)
{
    for (int i = 0; i < damages.Length; i++)
    {
        // Delay each number slightly so they appear in rapid succession
        float delay = i * 0.1f;
        DOVirtual.DelayedCall(
            delay,
            () => ShowDamage(worldPos, damages[i], isCritical)
        );
    }
}
```

**What It Does**:

1. **Iterate Through `damages[]`**

   * Each element in the `float[] damages` array represents a separate hit (e.g., a 3‐hit combo might have damages = `[50, 75, 100]`).

2. **Schedule Each Popup with a Small Delay**

   * For each index `i`, compute `delay = i * 0.1f`—the first damage is instant (0 s), second after 0.1 s, third after 0.2 s, etc.
   * Call `DOVirtual.DelayedCall(delay, callback)` which uses DOTween’s scheduler to invoke the callback after `delay` seconds.

3. **Callback Invokes `ShowDamage(...)`**

   * `() => ShowDamage(worldPos, damages[i], isCritical)` shows each damage popup in quick succession at the same `worldPos`.
   * Because `ShowDamage` itself introduces random spread, the popups won’t be exactly on top of each other.

**Note**:

* This method does not return a `Task` or `IEnumerator`—it simply schedules the popups and returns immediately.
* If you pass `isCritical = true`, all of the combo’s numbers will use `CriticalColor`.

---

## 6. Billboard Component

Whenever a damage popup spawns, you want it to face the player’s camera so it’s easy to read. The `Billboard` script (attached to each popup prefab) handles that.

```csharp
/// <summary>
/// Makes an object always face the camera
/// </summary>
public class Billboard : MonoBehaviour
{
    private Camera _mainCamera;
    
    void Start()
    {
        _mainCamera = Camera.main;
    }
    
    void LateUpdate()
    {
        if (_mainCamera != null)
        {
            transform.forward = _mainCamera.transform.forward;
        }
    }
}
```

**How It Works**:

1. **Cache `Camera.main` in `Start()`**

   * On first frame, store a reference to the main camera. If you have multiple cameras or swap cameras at runtime, you may need to update this reference accordingly.

2. **In `LateUpdate()`**

   * Each frame (after all other transforms have updated), sets `transform.forward = _mainCamera.transform.forward`.
   * This aligns the popup’s local “forward” vector to exactly match the camera’s “forward” vector—effectively rotating the popup so its face is always parallel to the camera plane.
   * As a result, no matter where the camera moves or rotates, the text remains readable from the front.

**Example**:

* If the camera moves behind an enemy and sees the popup from an angle, the `Billboard` ensures the text still rotates to face the camera instead of appearing as a flat, unreadable line from the wrong side.

---

## 7. Putting It All Together: Real-World Usage

Below are a series of examples illustrating how you might wire up **DamagePopupHelper** in a typical Unity project.

### 7.1 Bootstrapping at Game Start

Create a MonoBehaviour in your initial scene (e.g., an “UIManager” or “GameInitializer”) to initialize the popup pool before anything else runs.

```csharp
using UnityEngine;

public class GameInitializer : MonoBehaviour
{
    public async void Awake()
    {
        // 1. Initialize the DamagePopupHelper (builds the pool asynchronously)
        await DamagePopupHelper.Initialize();
        
        // 2. (Optionally) log that it’s ready
        Debug.Log("Damage popup system initialized.");
    }
}
```

* **Attach** this script to a persistent GameObject (e.g., “Managers”) in your very first scene.
* **`await DamagePopupHelper.Initialize()`** ensures the pool of 20 initial instances is created.
* **After initialization**, you can immediately call `DamagePopupHelper.ShowDamage(...)` anywhere in your code without worrying about a null pool.

---

### 7.2 Showing Damage in Combat Scripts

Whenever an enemy or player takes damage, call `ShowDamage` with the world position and damage amount.

```csharp
public class EnemyHealth : MonoBehaviour
{
    [SerializeField] private float _maxHealth = 100f;
    private float _currentHealth;

    private void Start()
    {
        _currentHealth = _maxHealth;
    }

    public void TakeDamage(float amount, bool isCritical = false)
    {
        _currentHealth -= amount;
        
        // 1. Display the floating damage number above this enemy
        Vector3 popupPosition = transform.position + Vector3.up * 2f; // 2 units above the enemy’s feet
        DamagePopupHelper.ShowDamage(popupPosition, amount, isCritical, isHeal: false);
        
        // 2. Check for death
        if (_currentHealth <= 0f)
        {
            Die();
        }
    }

    private void Die()
    {
        // Example: On death, show a large “0” or a special popup
        Vector3 popupPosition = transform.position + Vector3.up * 2f;
        DamagePopupHelper.ShowDamage(popupPosition, 0f, isCritical: false, isHeal: false);
        Destroy(gameObject);
    }
}
```

* **`TakeDamage(...)`**: Whenever this enemy takes damage, subtract health and spawn a popup 2 units above its position.
* `ShowDamage` handles color, animation, and returning to pool automatically.

---

### 7.3 Healing Popups

Similarly, if the player (or an NPC) receives healing, call `ShowDamage` with `isHeal = true`. The helper shows “+X” in green.

```csharp
public class PlayerHealth : MonoBehaviour
{
    [SerializeField] private float _maxHealth = 150f;
    private float _currentHealth;

    private void Start()
    {
        _currentHealth = _maxHealth;
    }

    public void Heal(float amount)
    {
        _currentHealth = Mathf.Min(_currentHealth + amount, _maxHealth);
        
        // Show a green “+amount” popup
        Vector3 popupPosition = transform.position + Vector3.up * 2f;
        DamagePopupHelper.ShowDamage(popupPosition, amount, isCritical: false, isHeal: true);
    }
}
```

* **Note**: The popup text is formatted as `"+{amount:F0}"` (no decimals). If you prefer decimals, change `F0` → `F1` or `F2` in the code.

---

### 7.4 Critical Hits & Color Coding

When an attack lands critically, call `ShowDamage` with `isCritical = true`. The popup appears in red (`CriticalColor`).

```csharp
public class PlayerAttack : MonoBehaviour
{
    [SerializeField] private float _baseDamage = 20f;
    [SerializeField] private float _criticalMultiplier = 2f;
    [SerializeField] private float _critChance = 0.25f;

    public void DealDamage(GameObject target)
    {
        bool isCritical = (Random.value < _critChance);
        float damage = isCritical ? _baseDamage * _criticalMultiplier : _baseDamage;
        
        // Suppose target has an EnemyHealth component
        var enemyHealth = target.GetComponent<EnemyHealth>();
        if (enemyHealth != null)
        {
            enemyHealth.TakeDamage(damage, isCritical);
        }
    }
}
```

* **Inside `TakeDamage(damage, isCritical)`**, the `DamagePopupHelper` will color the text red if `isCritical` is true (using `CriticalColor`).

---

### 7.5 Combo Attacks

If you want to display a burst of numbers for a multi-hit combo, use `ShowCombo(...)`:

```csharp
public class ComboAttack : MonoBehaviour
{
    // Example combo of 3 hits with varying damage
    private readonly float[] _comboDamages = new float[] { 15f, 20f, 30f };

    public void PerformCombo(GameObject target)
    {
        // For demonstration, assume all are non‐critical
        Vector3 popupPosition = target.transform.position + Vector3.up * 2f;
        DamagePopupHelper.ShowCombo(popupPosition, _comboDamages, isCritical: false);

        // Actually apply damage to the target in quick succession as well
        StartCoroutine(ApplyComboDamage(target));
    }

    private IEnumerator ApplyComboDamage(GameObject target)
    {
        var enemyHealth = target.GetComponent<EnemyHealth>();
        for (int i = 0; i < _comboDamages.Length; i++)
        {
            enemyHealth.TakeDamage(_comboDamages[i], isCritical: false);
            yield return new WaitForSeconds(0.1f); // matches the ShowCombo delay
        }
    }
}
```

* **`ShowCombo(...)`** schedules three popups at 0 s, 0.1 s, and 0.2 s.
* The coroutine `ApplyComboDamage` applies the health reduction in lockstep with the visual popups.

---

## 8. Customization & Tips

1. **Adjusting Popup Appearance**

   * **Font & Size**: In `CreateDamagePopupPrefab()`, change `tmp.fontSize` to suit your world scale (8 might be very small in a large world).
   * **Outline or Shadow**: If you want drop shadows or outlines, add `TextMeshProUGUI` components or use `tmp.fontSharedMaterial` to tweak the Material for outlines.

2. **Critical Scale Effect**

   * Although `CriticalScale` is defined (1.5f), the current code does not use it. If you want to make critical numbers bigger, modify the sequence in `ShowDamage`:

     ```csharp
     if (isCritical)
         popup.transform.localScale = Vector3.one * CriticalScale;
     else
         popup.transform.localScale = Vector3.one;
     sequence.Append(popup.transform.DOPunchScale(Vector3.one * 0.5f, 0.2f));
     ```
   * This ensures critical popups start at, say, 1.5× scale, then punch bigger from there.

3. **Using Elemental Colors**

   * The `ElementalColors` field is provided but not utilized. To show elemental damage (e.g., ice, fire), pass `isCritical = false, isHeal = false` and manually set `text.color = ElementalColors` before animating.

4. **Pooling Behavior**

   * The pool is configured to `AutoExpand = true` up to `MaxSize = 100`. Once you exceed 100 concurrent popups, `Get()` will start returning `null`. If your game expects more than 100 damage popups at once (unlikely unless thousands of hits land simultaneously), bump up `MaxSize`.

5. **Destroying the Pool**

   * If you need to clear the pool (e.g., on a major scene unload), call `ObjectPoolHelper.ClearPool(POOL_KEY)` (assuming your pooling system has that method), or simply let each popup’s `ReturnToPool()` itself disable it. The pool persists until you explicitly destroy it or shut down.

6. **Ensuring TextMeshPro Assets Exist**

   * Make sure your project has a `Resources/Fonts/Default.asset` (a `TMP_FontAsset`) so `Resources.Load<TMP_FontAsset>("Fonts/Default")` succeeds. Otherwise, loading will return `null` and text won’t render.

7. **Billboard Performance**

   * Each popup’s `Billboard` runs in `LateUpdate()`, which can be expensive if you have hundreds of popups simultaneously. For extremely high spawn rates, consider batching or using a shader that always faces camera instead of a script per-object.

8. **Thread Safety & `async void`**

   * `ShowDamage` is `async void`. If `Initialize()` has not yet completed, `ObjectPoolHelper.Get(...)` may fail. Always **await** `Initialize()` at startup. Do not rely on uninitialized pools.

---

## 9. Summary of Public API

| Method Signature                                                                                                    | Description                                                                                                                                                                                                                                                              |
| ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `public static async Task Initialize()`                                                                             | Builds and configures a pool of damage popup GameObjects (initial size 20, max 100). Must be called once before using any `ShowDamage` or `ShowCombo`.                                                                                                                   |
| `public static async void ShowDamage(Vector3 worldPos, float amount, bool isCritical = false, bool isHeal = false)` | Fetches a popup from the pool, positions it at `worldPos` (with slight random offset), sets its text (+/– amount) and color (critical red, heal green, or normal white), then animates (scale-punch, float up, fade out) over 1 second, before returning it to the pool. |
| `public static void ShowCombo(Vector3 worldPos, float[] damages, bool isCritical = false)`                          | For each element in `damages[]`, schedules a `ShowDamage` call with a delay of `i * 0.1f` seconds. Ideal for multi-hit combos where you want damage numbers to appear in rapid succession.                                                                               |
| *(Implicit)* `private static GameObject CreateDamagePopupPrefab()`                                                  | Constructs (in code) a new GameObject with a `TMP_Text` component (font, size), a `Billboard` component (so it faces the camera), and a `PoolableObject` (with `PoolKey = "DamagePopup"`). Used by `Initialize()`.                                                       |
| *(Component)* `public class Billboard : MonoBehaviour { … }`                                                        | Attaches to any popup so that in `LateUpdate()`, it sets its forward direction to match `Camera.main.transform.forward`, guaranteeing the text is always readable.                                                                                                       |

With these APIs, you have a complete system to display and animate damage/heal popups in world space with minimal boilerplate. Simply initialize once, then call `ShowDamage(...)` or `ShowCombo(...)` wherever appropriate in your game logic.
