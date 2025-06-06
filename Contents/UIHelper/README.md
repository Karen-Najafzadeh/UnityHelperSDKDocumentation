Below is a comprehensive guide to **UIHelper**, a static utility class that centralizes many common UI tasks in Unity—including pooling, animation, toast/popup management, dynamic creation, responsive layouts, and localization. At the end, you’ll find a summary table listing every public method and its purpose.

---

## 1. Overview & Purpose

`UIHelper` is designed to handle:

* **UI Element Pooling**: Reuse UI prefabs (e.g. toasts, spinners, popups) rather than repeatedly instantiating/destroying them.
* **Animations & Transitions**: Fade‐in/out, scale‐in/out, rotate, etc., via DOTween.
* **Toast Notifications**: Temporary “toast” messages that appear on screen, fade in, wait, then fade out.
* **Popup Dialogs**: Simple modal dialogs with title, message, and a set of buttons—returns the chosen index.
* **Loading Indicators & Screens**: Show/hide spinners or full‐screen loading overlays.
* **Dynamic UI Creation**: Create labels, buttons, overlays, and “dialogue choice” menus at runtime.
* **Responsive Layout Helpers**: Adjust UI elements’ scale based on screen size (via a `ResponsiveElement` component).
* **Pooling Infrastructure**: Internally manage dictionaries of prefab keys → prefab GameObjects, plus queues of pooled instances.
* **Localization Integration**: Wherever text is shown, it’s passed through `LocalizationManager.Get(...)`.

Before you call most of these features, you must call:

```csharp
UIHelper.Initialize();
```

or

```csharp
UIHelper.Initialize(myCanvas);
```

This sets up the main canvas and registers built‐in prefabs (“Toast,” “LoadingSpinner,” “Popup”) for pooling.

---

## 2. Initialization & Prefab Registration

### 2.1 `Initialize(Canvas mainCanvas)`

```csharp
public static void Initialize(Canvas mainCanvas)
```

* **Purpose**: Provide a reference to your project’s main `Canvas` so that all pooled UI elements (toasts, spinners, popups) can be parented there.
* **Parameters**:

  * `mainCanvas` (`Canvas`): The Canvas under which UI elements will be instantiated.
* **Behavior**:

  1. Stores `_mainCanvas = mainCanvas`.
  2. Calls `InitializeCommonPrefabs()` to register the built-in “Toast,” “LoadingSpinner,” and “Popup” prefabs into `_uiPrefabs` and set up empty queues in `_uiPools`.

```csharp
// Usage:
UIHelper.Initialize(myGameCanvas);
```

---

### 2.2 `Initialize()`

```csharp
public static void Initialize()
```

* **Purpose**: If you don’t have a reference to a Canvas, this overload will:

  1. Attempt to find an existing `Canvas` in the scene.
  2. If none is found, create a new `GameObject("MainCanvas")` with `Canvas`, `CanvasScaler`, and `GraphicRaycaster`.
* Then it registers the common prefabs exactly the same way as `Initialize(Canvas)`.

```csharp
// Usage when you don’t explicitly pass a Canvas:
UIHelper.Initialize();
```

---

### 2.3 `RegisterUIPrefab`

```csharp
public static void RegisterUIPrefab(string key, GameObject prefab)
```

* **Purpose**: Manually register any additional UI prefab under a key, so that `GetFromPool(key)` can retrieve pooled instances of it.
* **Parameters**:

  * `key` (`string`): Unique identifier (e.g. `"MyCustomPanel"`).
  * `prefab` (`GameObject`): The UI prefab asset to pool.
* **Behavior**:

  1. Stores `_uiPrefabs[key] = prefab;`
  2. Initializes `_uiPools[key] = new Queue<GameObject>();`

```csharp
// Usage (after Initialize()):
UIHelper.RegisterUIPrefab("MyPanel", myPanelPrefab);
```

---

### 2.4 `InitializeCommonPrefabs` (private)

Internally called by `Initialize(...)` to set up three built-in prefabs:

* **Key = `"Toast"`** → `CreateToastPrefab()`
* **Key = `"LoadingSpinner"`** → `CreateLoadingSpinnerPrefab()`
* **Key = `"Popup"`** → `CreatePopupPrefab()`

Each of those `Create…Prefab()` methods builds a simple GameObject (with Image/TextMeshPro/etc.) and returns it to be stored for pooling. For example, `CreateToastPrefab()` makes a semi-transparent background + child `TMP_Text` + `CanvasGroup`.

---

## 3. UI Element Pooling

Rather than instantiating/destroying UI objects repeatedly, `UIHelper` maintains:

* `_uiPrefabs`: `Dictionary<string, GameObject>` mapping a key to a prefab.
* `_uiPools`: `Dictionary<string, Queue<GameObject>>` of inactive instances ready to be reused.

### 3.1 `GetFromPool`

```csharp
public static GameObject GetFromPool(string prefabKey)
```

* **Purpose**: Return an existing pooled instance of the UI prefab under `prefabKey`, if available; otherwise instantiate a fresh copy from `_uiPrefabs[prefabKey]` and parent it under `_mainCanvas`.
* **Parameters**:

  * `prefabKey` (`string`): Must have been registered earlier.
* **Returns**: A **disabled** (inactive) `GameObject` instance (ready for configuration).

```csharp
GameObject toastObj = UIHelper.GetFromPool("Toast");
```

---

### 3.2 `ReturnToPool`

```csharp
public static void ReturnToPool(GameObject obj, string prefabKey)
```

or (overload)

```csharp
private static void ReturnToPool(string prefabKey, GameObject element)
```

* **Purpose**: Deactivate a UI element and enqueue it back into the pool under `prefabKey` (so it can be reused later).
* **Parameters**:

  * `obj` or `element` (`GameObject`): The instance to recycle.
  * `prefabKey` (`string`): The same key under which it was created.
* **Behavior**:

  1. `obj.SetActive(false);`
  2. `obj.transform.SetParent(_mainCanvas.transform)` (so it stays under the canvas but hidden).
  3. `_uiPools[prefabKey].Enqueue(obj);`

```csharp
// After using a toast:
UIHelper.ReturnToPool(toastObj, "Toast");
```

---

### 3.3 `GetOrCreateUIElement` (private)

```csharp
private static GameObject GetOrCreateUIElement(string prefabKey)
```

* **Purpose**: Convenience method that tries to `Dequeue()` from `_uiPools[prefabKey]` if available; otherwise calls `CreateNewUIElement(prefabKey)` to instantiate and return a new one.
* **Usage**: Internally used in methods like `ShowToast` and `ShowPopup` when they need an instance—no need for you to call this directly.

---

### 3.4 `CreateNewUIElement` (private)

```csharp
private static GameObject CreateNewUIElement(string prefabKey)
```

* **Purpose**: Instantiate a new copy of `_uiPrefabs[prefabKey]`, parent it to `_mainCanvas.transform`, deactivate it, and return it.
* **Usage**: Called by `GetOrCreateUIElement`.

---

## 4. Toast Notifications

Two overloads of `ShowToast` are provided: one uses the pooled “Toast” prefab (with `CanvasGroup` fade), and one dynamically creates a new `TMP_Text` if you call it with a custom duration. Both fade in, wait, then fade out.

### 4.1 `ShowToast(string message, Color? color = null)`

```csharp
public static async Task ShowToast(string message, Color? color = null)
```

* **Purpose**: Display a pooled “Toast” prefab for a default duration (2s).
* **Behavior**:

  1. Calls `GetOrCreateUIElement("Toast")` to get a toast GameObject.
  2. Retrieves the child `TMP_Text`, sets `.text = LocalizationManager.Get(message)`. If `color` is provided, uses it.
  3. Positions the toast horizontally centered at `Screen.height = ToastSize.y` (bottom of screen).
  4. Uses DOTween on its `CanvasGroup` to fade `alpha` 0 → 1 over `ToastFadeTime` (0.3s), then `await Task.Delay(ToastDuration * 1000)` (default 2s), then fade 1 → 0.
  5. Finally, calls `ReturnToPool("Toast", toast)`.

```csharp
await UIHelper.ShowToast("ui.toast.level_complete", color: Color.yellow);
```

---

### 4.2 `ShowToast(string message, float duration = 2f)`

```csharp
public static async Task ShowToast(string message, float duration = 2f)
```

* **Purpose**: A second overload that dynamically creates a new GameObject (not pooled).
* **Behavior**:

  1. If `_mainCanvas` is null, calls `Initialize()`.
  2. Creates a new `GameObject("Toast")`, parents to `_mainCanvas`.
  3. Adds a `TextMeshProUGUI` component, sets `.text = message` (not localized here), adjusts font/size/color.
  4. Positions at `ToastPosition` (Vector2(0,100)) in anchored space.
  5. Fades its `alpha` from 0→1, `await Task.Delay(duration * 1000)`, then fade 1→0.
  6. Destroys the `GameObject` at the end.

```csharp
await UIHelper.ShowToast("Hello World!", duration: 3f);
```

---

## 5. Popup Management (Modal Dialogs)

### 5.1 `ShowPopup(string title, string message, params string[] buttons)`

```csharp
public static async Task<int> ShowPopup(string title, string message, params string[] buttons)
```

* **Purpose**: Display a pooled “Popup” prefab with a title, message, and a row of buttons—returns (awaits) an `int` corresponding to the index of the button clicked.
* **Parameters**:

  * `title` (`string`): Key string passed to `LocalizationManager.Get(title)` to set the title text.
  * `message` (`string`): Key string passed to `LocalizationManager.Get(message)` for the main message.
  * `buttons` (`params string[]`): Array of button‐label keys. Each one is localized, then a `Button` is created under the “ButtonContainer” child of the popup.
* **Behavior**:

  1. Calls `GetOrCreateUIElement("Popup")`.
  2. Finds child nodes “Title” and “Message” (both `TMP_Text`) and sets their `.text` via localization.
  3. Finds `ButtonContainer` (a `RectTransform`) and populates it with one button per label in `buttons`. Each button is created via `CreateButton(...)` which returns a `Button` with its `onClick` bound to `tcs.SetResult(i)`. The `TaskCompletionSource<int>` (`tcs`) lets us await the user’s choice.
  4. Pushes `popup` onto `_activePopups` stack.
  5. Calls `await AnimatePopupIn(popup)` (scales from 0→1).
  6. `int result = await tcs.Task;` awaits user click.
  7. Calls `await AnimatePopupOut(popup)` (scales from 1→0), pops from `_activePopups`, and `ReturnToPool("Popup", popup)`.
  8. Returns `result` to the caller.

```csharp
int choice = await UIHelper.ShowPopup(
    "ui.popup.confirm_title",
    "ui.popup.confirm_message",
    "Yes", "No", "Cancel"
);
// choice == 0 if “Yes,” 1 if “No,” 2 if “Cancel.”
```

---

### 5.2 `ShowPopup(GameObject popupPrefab)`

```csharp
public static GameObject ShowPopup(GameObject popupPrefab)
```

* **Purpose**: Instantly instantiate (not pooled) a custom `popupPrefab` under `_mainCanvas`, scale it in (0→1), push to `_activePopups` stack, and return the instance.
* **Parameters**:

  * `popupPrefab` (`GameObject`): Any prefab you supply.
* **Behavior**:

  1. Instantiates `popupPrefab` under `_mainCanvas`.
  2. Pushes onto `_activePopups`.
  3. `rect.localScale = Vector3.zero; rect.DOScale(1, 0.3f).SetEase(Ease.OutQuad);`
  4. Returns the `popup` so you can manually close it later.

```csharp
GameObject myCustomPopup = UIHelper.ShowPopup(myPrefab);
```

---

### 5.3 `ClosePopup`

```csharp
public static async Task ClosePopup(GameObject popup = null)
```

* **Purpose**: Animate a popup out (scale 1→0) and destroy it.
* **Parameters**:

  * `popup` (`GameObject`, optional): If null, it pops/animates the top‐of‐stack (`_activePopups.Peek()`).
* **Behavior**:

  1. Finds the top‐of‐stack popup if none is provided.
  2. `await popup.transform.DOScale(0, 0.3f).SetEase(Ease.OutQuad).AsyncWaitForCompletion()`.
  3. Remove it from `_activePopups` (reconstruct the stack without that item).
  4. `Destroy(popup)`.

```csharp
await UIHelper.ClosePopup(); // Closes the most recently opened popup
```

---

### 5.4 `CloseAllPopups`

```csharp
public static void CloseAllPopups()
```

* **Purpose**: Immediately destroy all popups in `_activePopups`. No animations, just `Destroy(...)`.
* **Behavior**:

  1. While `_activePopups.Count > 0`, pop each `popup` and `Destroy(popup)`.

```csharp
UIHelper.CloseAllPopups();
```

---

### 5.5 `AnimatePopupIn` & `AnimatePopupOut` (private)

```csharp
private static async Task AnimatePopupIn(GameObject popup)
private static async Task AnimatePopupOut(GameObject popup)
```

* **Purpose**: Internal helpers to scale a popup in (0→1) or out (1→0) over 0.3s with `Ease.OutQuad`, awaiting completion before returning. Used by `ShowPopup(...)` and `ClosePopup(...)`.

---

## 6. Loading Spinner & Loading Screen

### 6.1 `ShowLoadingSpinner`

```csharp
public static GameObject ShowLoadingSpinner(Transform parent = null)
```

* **Purpose**: Display a pooled “LoadingSpinner” (rotating image) under `parent` (or under `_mainCanvas` if `parent==null`).
* **Parameters**:

  * `parent` (`Transform`, optional): Where to attach the spinner. If omitted, parent is `_mainCanvas.transform`.
* **Behavior**:

  1. Calls `GetOrCreateUIElement("LoadingSpinner")`.
  2. If `parent != null`, reparent to `parent`; otherwise keep under `_mainCanvas`.
  3. Applies a continuous rotation via DOTween: `.DORotate(new Vector3(0,0,-360), 1f, LoopType.Incremental).SetEase(Ease.Linear).SetLoops(-1)`.
  4. `spinner.SetActive(true)` and return it.

```csharp
GameObject spinner = UIHelper.ShowLoadingSpinner(dialoguePanelTransform);
// Later: UIHelper.HideLoadingSpinner(spinner);
```

---

### 6.2 `HideLoadingSpinner`

```csharp
public static void HideLoadingSpinner(GameObject spinner)
```

* **Purpose**: Stop the DOTween rotation on `spinner` and return it to the pool under `"LoadingSpinner"`.
* **Parameters**:

  * `spinner` (`GameObject`): The instance returned earlier by `ShowLoadingSpinner`.
* **Behavior**:

  1. `spinner.transform.DOKill()` (stop animations).
  2. `ReturnToPool("LoadingSpinner", spinner)`.

```csharp
UIHelper.HideLoadingSpinner(spinner);
```

---

### 6.3 `ShowLoadingIndicator`

```csharp
public static void ShowLoadingIndicator()
```

* **Purpose**: Create a brand‐new (non‐pooled) full‐screen “LoadingIndicator” with an `Image` sprite named `"LoadingSpinner"` (from `Resources/LoadingSpinner`), centered on screen, and rotate it continuously.
* **Behavior**:

  1. If `_mainCanvas` is null, `Initialize()`.
  2. If `_loadingIndicator` already exists, return (prevents duplicates).
  3. Create a `GameObject("LoadingIndicator")`, parent under `_mainCanvas`.
  4. Add an `Image` component, load `Resources.Load<Sprite>("LoadingSpinner")`. Size it to (100×100) via `RectTransform`.
  5. Start a DOTween rotation:

     ```csharp
     DOTween.To(() => rect.localRotation.eulerAngles.z,
                z => rect.localRotation = Quaternion.Euler(0,0,z),
                360f, 1f)
            .SetLoops(-1, LoopType.Incremental)
            .SetEase(Ease.Linear);
     ```
  6. Store in `_loadingIndicator`.

```csharp
UIHelper.ShowLoadingIndicator();
```

---

### 6.4 `HideLoadingIndicator`

```csharp
public static void HideLoadingIndicator()
```

* **Purpose**: Destroy the `_loadingIndicator` GameObject (if any), and set the reference to `null`.
* **Behavior**:

  1. If `_loadingIndicator != null`:

     * `Destroy(_loadingIndicator);`
     * `_loadingIndicator = null;`

```csharp
UIHelper.HideLoadingIndicator();
```

---

### 6.5 `ShowLoadingScreen`

```csharp
public static async Task ShowLoadingScreen()
```

* **Purpose**: Display a **full‐screen** loading overlay (semi‐transparent black background, “Loading…” text, and spinner), fade it in over 0.3s.
* **Behavior**:

  1. If `_mainCanvas` is null, `Initialize()`.
  2. If `_loadingScreen == null`, build a `GameObject("LoadingScreen")` as follows:

     * Add a `RectTransform` that stretches full screen (`anchorMin=(0,0), anchorMax=(1,1), sizeDelta=(0,0)`).
     * Add an `Image` (color = `(0,0,0,0.9f)`).
     * Create a child “Loading…” label via `CreateLabel("Loading...", parent)`, centered.
     * Call `ShowLoadingSpinner(_loadingScreen.transform)` to add a rotating spinner under it.
     * Add `CanvasGroup` to `_loadingScreen`, set `alpha=0`.
  3. `SetActive(true)`, then `await _loadingScreenCanvasGroup.DOFade(1f, 0.3f).AsyncWaitForCompletion()`.

```csharp
await UIHelper.ShowLoadingScreen();
// ... do async loading ...
await UIHelper.HideLoadingScreen();
```

---

### 6.6 `HideLoadingScreen`

```csharp
public static async Task HideLoadingScreen()
```

* **Purpose**: Fade the full‐screen loading overlay (`_loadingScreen`) out over 0.3s, then deactivate it.
* **Behavior**:

  1. If `_loadingScreen` and `_loadingScreenCanvasGroup` are not null:

     * `await _loadingScreenCanvasGroup.DOFade(0f, 0.3f).AsyncWaitForCompletion();`
     * `_loadingScreen.SetActive(false);`

```csharp
await UIHelper.HideLoadingScreen();
```

---

## 7. Overlays & Dynamic Creation

### 7.1 `CreateOverlay`

```csharp
public static GameObject CreateOverlay(Color color)
```

* **Purpose**: Create a simple full‐screen UI overlay (an `Image` stretched over the entire canvas) with the specified `color` and a `CanvasGroup` (alpha = 0) so you can fade it in/out.
* **Parameters**:

  * `color` (`Color`): The overlay’s base color (e.g. semi-transparent black).
* **Behavior**:

  1. If `_mainCanvas` is null, `Initialize()`.
  2. Create a `GameObject("Overlay")`, parent to `_mainCanvas`.
  3. Add `RectTransform` → set `anchorMin=(0,0)`, `anchorMax=(1,1)`, `sizeDelta = (0,0)`.
  4. Add an `Image` component, set `.color = color`.
  5. Add a `CanvasGroup` with `alpha = 0`.
  6. `rect.SetAsLastSibling()` to ensure it’s on top.
  7. Return the `overlay` GameObject (so you can fade its CanvasGroup, then destroy later).

```csharp
GameObject overlay = UIHelper.CreateOverlay(new Color(0, 0, 0, 0.5f));
CanvasGroup cg = overlay.GetComponent<CanvasGroup>();
await cg.DOFade(1f, 0.3f).AsyncWaitForCompletion();
// … later:
await cg.DOFade(0f, 0.3f).AsyncWaitForCompletion();
GameObject.Destroy(overlay);
```

---

### 7.2 `CreateLabel`

```csharp
public static TMP_Text CreateLabel(string text, Transform parent = null)
```

* **Purpose**: Dynamically create a new `GameObject("Label")` with a `TMP_Text` component, set its `text` via localization, assign default font/size/alignment, and parent it under `parent` (if provided).
* **Parameters**:

  * `text` (`string`): Localization key passed to `LocalizationManager.Get(text)`.
  * `parent` (`Transform`, optional): Where to parent this label.
* **Returns**: The `TMP_Text` component, so you can further adjust it.

```csharp
// Example: Create a label under a panel
TMP_Text lbl = UIHelper.CreateLabel("ui.score", myPanelTransform);
lbl.fontSize = 18;
lbl.alignment = TextAlignmentOptions.Left;
```

---

### 7.3 `CreateButton`

```csharp
public static Button CreateButton(string text, int result, TaskCompletionSource<int> tcs)
```

* **Purpose**: Dynamically create a `GameObject("Button")` with a `Button + Image + TMP_Text` child, set its label via localization, and hook its `onClick` to call `tcs.SetResult(result)`.
* **Parameters**:

  * `text` (`string`): Key passed to `LocalizationManager.Get(text)`.
  * `result` (`int`): The integer to pass back when this button is clicked (to the awaiting `TaskCompletionSource<int>`).
  * `tcs` (`TaskCompletionSource<int>`): The TCS that will complete with `result` once clicked.
* **Returns**: The created `Button` component, so you can parent it and adjust its RectTransform as needed.

```csharp
var tcs = new TaskCompletionSource<int>();
Button okButton = UIHelper.CreateButton("ui.ok", 0, tcs);
okButton.transform.SetParent(buttonContainerTransform, false);
// Later: int choice = await tcs.Task;
```

---

## 8. Responsive Layout Helpers

### 8.1 `MakeResponsive`

```csharp
public static void MakeResponsive(RectTransform rect, float minWidth = 0, float maxWidth = float.MaxValue)
```

* **Purpose**: Attach a `ResponsiveElement` component to `rect.gameObject`, and set its `MinWidth`/`MaxWidth`. That component will scale the UI element up or down based on the current `Screen.width`.
* **Parameters**:

  * `rect` (`RectTransform`): The UI element to make responsive.
  * `minWidth` (float): If `Screen.width < minWidth`, the element’s scale is reduced proportionally.
  * `maxWidth` (float): If `Screen.width > maxWidth`, the element’s scale is reduced proportionally.

```csharp
UIHelper.MakeResponsive(myPanelRect, minWidth: 320f, maxWidth: 800f);
```

---

### 8.2 `UpdateResponsiveLayout`

```csharp
public static void UpdateResponsiveLayout()
```

* **Purpose**: Finds all `ResponsiveElement` components in the scene and calls their `UpdateLayout()` to recalculate their scale based on current `Screen.width`.
* **Behavior**:

  1. `FindObjectsOfType<ResponsiveElement>()`, then calls `.UpdateLayout()` on each.

Call this method whenever the screen size changes (e.g., in a handler for orientation change, or in your MonoBehaviour’s `Update()` if you want continuous responsiveness).

```csharp
// Example, in a MonoBehaviour handling resolution/orientation changes:
UIHelper.UpdateResponsiveLayout();
```

---

## 9. Dialogue Choice Menu

### 9.1 `ShowDialogueChoices`

```csharp
public static async Task<string> ShowDialogueChoices(Dictionary<string, DialogueHelper.DialogueNode> choices)
```

* **Purpose**: Display a vertical list of buttons—one per key in `choices`—anchored at the bottom of screen. When the user clicks one, the returned `Task<string>` completes with that key.
* **Parameters**:

  * `choices` (`Dictionary<string, DialogueHelper.DialogueNode>`): Each entry’s `Key` is the ID returned when clicked; `Value.Text` is displayed on the button (localized via `CreateChoiceButton`).
* **Behavior**:

  1. If `_mainCanvas` is null, call `Initialize()`.
  2. Create a new `GameObject("ChoiceContainer")` with a `RectTransform` under `_mainCanvas`. Anchor it to the bottom center (`anchorMin = (0.5,0), anchorMax = (0.5,0), pivot = (0.5,0), anchoredPosition=(0,100)`).
  3. Add `VerticalLayoutGroup` with spacing & padding.
  4. For each `(choiceId, node)` in `choices`, call `CreateChoiceButton(choiceId, node.Text)`, parent it under the container, and add `onClick` to set the TCS’s result and destroy the container.
  5. `return await tcs.Task;` completing when a button is clicked.

```csharp
var choices = new Dictionary<string, DialogueHelper.DialogueNode> {
    { "yes", new DialogueHelper.DialogueNode { Text="Yes" } },
    { "no", new DialogueHelper.DialogueNode { Text="No" } }
};
string choiceId = await UIHelper.ShowDialogueChoices(choices);
// choiceId == "yes" or "no" depending on which button was clicked.
```

---

## 10. Helper Classes

### 10.1 `ResponsiveElement` (MonoBehaviour)

```csharp
public class ResponsiveElement : MonoBehaviour
{
    public float MinWidth { get; set; }
    public float MaxWidth { get; set; }
    private RectTransform _rect;

    private void Awake()
    {
        _rect = GetComponent<RectTransform>();
    }

    public void UpdateLayout()
    {
        float screenWidth = Screen.width;
        float scale = 1f;

        if (screenWidth < MinWidth)
            scale = screenWidth / MinWidth;
        else if (screenWidth > MaxWidth)
            scale = screenWidth / MaxWidth;

        transform.localScale = Vector3.one * scale;
    }
}
```

* **Usage**: Automatically scales the attached `RectTransform` proportionally if the current `Screen.width` falls outside `[MinWidth,MaxWidth]`.
* You attach it by calling `UIHelper.MakeResponsive(myRectTransform, min, max)`.
* To apply changes, call `UIHelper.UpdateResponsiveLayout()`.

---

## 11. Internal Prefab Creation Methods

These three private methods build simple UI GameObjects that get registered into `_uiPrefabs` for pooling.

### 11.1 `CreateToastPrefab`

```csharp
private static GameObject CreateToastPrefab()
```

* **Returns**: A new `GameObject("Toast")` configured like:

  1. `RectTransform.sizeDelta = (400,80)`
  2. `Image` (semi-transparent black background).
  3. Child `GameObject("Text")` with `TMP_Text` (white, centered, 24pt).
  4. `CanvasGroup` (for fading).

---

### 11.2 `CreateLoadingSpinnerPrefab`

```csharp
private static GameObject CreateLoadingSpinnerPrefab()
```

* **Returns**: A new `GameObject("LoadingSpinner")` configured like:

  1. `RectTransform.sizeDelta = (80,80)`.
  2. `Image` component using `Resources.Load<Sprite>("UI/LoadingSpinner")`.

---

### 11.3 `CreatePopupPrefab`

```csharp
private static GameObject CreatePopupPrefab()
```

* **Returns**: A new `GameObject("Popup")` configured like:

  1. `RectTransform.sizeDelta = (500,300)`.
  2. `Image` (solid black, 90% opacity).
  3. Child `Title` (TMP\_Text, 28pt, centered).
  4. Child `Message` (TMP\_Text, 20pt, centered).
  5. Child `ButtonContainer` (`RectTransform`)—initially empty; buttons are added dynamically.

---

## 12. Usage Examples

Below are a few common usage patterns, assuming you have a `Canvas` in your scene (or let `UIHelper.Initialize()` create one).

### 12.1 Displaying a Localized Toast

```csharp
public class Example : MonoBehaviour
{
    private async void Start()
    {
        UIHelper.Initialize(); // Find or create a Canvas automatically
        await UIHelper.ShowToast("ui.welcome_message", color: Color.green);
        // After 2s + fade time, the toast disappears automatically.
    }
}
```

---

### 12.2 Showing a Confirmation Popup

```csharp
public class ConfirmExample : MonoBehaviour
{
    private async void OnButtonClick()
    {
        UIHelper.Initialize(myCanvas); // pass your Canvas reference
        int choiceIndex = await UIHelper.ShowPopup(
            title: "ui.popup.confirm_title",
            message: "ui.popup.delete_warning",
            buttons: new[] { "ui.popup.yes", "ui.popup.no" }
        );
        if (choiceIndex == 0)
            DeleteItem();
        else
            Debug.Log("User cancelled");
    }

    private void DeleteItem() { /* ... */ }
}
```

---

### 12.3 Displaying a Full‐Screen Loading Screen

```csharp
public class LevelLoader : MonoBehaviour
{
    private async void LoadLevelAsync(string sceneName)
    {
        UIHelper.Initialize();
        await UIHelper.ShowLoadingScreen();
        
        var op = SceneManager.LoadSceneAsync(sceneName);
        while (!op.isDone)
        {
            await Task.Yield();
        }
        
        await UIHelper.HideLoadingScreen();
    }
}
```

---

### 12.4 Responsive UI Example

```csharp
public class ResponsiveDemo : MonoBehaviour
{
    public RectTransform panelRect;

    private void Start()
    {
        UIHelper.MakeResponsive(panelRect, minWidth: 320f, maxWidth: 800f);
    }

    private void Update()
    {
        // Whenever the resolution/orientation changes, call:
        UIHelper.UpdateResponsiveLayout();
    }
}
```

---

## 13. Public Method Summary Table

| **Method**                                                                                        | **Description**                                                                                                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **`Initialize(Canvas mainCanvas)`**                                                               | Store a reference to your Canvas, then register built-in “Toast,” “LoadingSpinner,” and “Popup” prefabs for pooling.                                                                                                                                   |
| **`Initialize()`**                                                                                | Try to find an existing Canvas; if none, create a new one. Then register built-in prefabs for pooling.                                                                                                                                                 |
| **`RegisterUIPrefab(string key, GameObject prefab)`**                                             | Manually register any additional UI prefab under `key`; prepares an empty queue for pooling.                                                                                                                                                           |
| **`GetFromPool(string prefabKey) → GameObject`**                                                  | Return a pooled instance of the registered prefab, or instantiate a new one if none available. The returned object is inactive by default.                                                                                                             |
| **`ReturnToPool(GameObject obj, string prefabKey)`**                                              | Deactivate `obj` and enqueue it back into the pool under `prefabKey` so it can be reused.                                                                                                                                                              |
| **`ShowToast(string message, Color? color = null) → Task`**                                       | Display a pooled “Toast” prefab (fade in, wait 2s, fade out). Localizes `message` via `LocalizationManager.Get(message)`; optionally override text color.                                                                                              |
| **`ShowToast(string message, float duration = 2f) → Task`**                                       | Dynamically create a new `GameObject("Toast")` with `TMP_Text`, fade in/out over `ToastFadeTime`, then destroy. No pooling.                                                                                                                            |
| **`ShowPopup(string title, string message, params string[] buttons) → Task<int>`**                | Display a pooled “Popup” prefab with localized `title` & `message`, create `buttons.Length` buttons (localized), await user click, return the button index, then close/return to pool.                                                                 |
| **`ShowPopup(GameObject popupPrefab) → GameObject`**                                              | Instantiate `popupPrefab` under the main canvas, scale it in (0→1), push onto `_activePopups` stack, and return the instance for manual handling.                                                                                                      |
| **`ClosePopup(GameObject popup = null) → Task`**                                                  | Animate scale out (1→0) on the top‐of‐stack popup (or the provided `popup`), remove it from `_activePopups`, then destroy it.                                                                                                                          |
| **`CloseAllPopups()`**                                                                            | Immediately destroy all popups in `_activePopups` (no animation).                                                                                                                                                                                      |
| **`ShowLoadingSpinner(Transform parent = null) → GameObject`**                                    | Display a pooled “LoadingSpinner” (rotating image) under `parent` (or main canvas if `null`), returns the instance.                                                                                                                                    |
| **`HideLoadingSpinner(GameObject spinner)`**                                                      | Stop DOTween animations on `spinner` and return it to the “LoadingSpinner” pool.                                                                                                                                                                       |
| **`ShowLoadingIndicator()`**                                                                      | Create a full‐screen “LoadingIndicator” GameObject (Image sprite “LoadingSpinner” from `Resources`) under main canvas, rotate continuously.                                                                                                            |
| **`HideLoadingIndicator()`**                                                                      | Destroy the singular `_loadingIndicator` GameObject if it exists.                                                                                                                                                                                      |
| **`ShowLoadingScreen() → Task`**                                                                  | Display (and fade in) a full‐screen loading overlay with semi-transparent black background, “Loading…” text, and spinner.                                                                                                                              |
| **`HideLoadingScreen() → Task`**                                                                  | Fade out (over 0.3s) and deactivate the full‐screen loading overlay created by `ShowLoadingScreen()`.                                                                                                                                                  |
| **`CreateOverlay(Color color) → GameObject`**                                                     | Create a new full‐screen `GameObject("Overlay")` with `Image` color = `color` and `CanvasGroup(alpha=0)`, parented under main canvas. Returns the overlay so you can fade it in/out.                                                                   |
| **`CreateLabel(string text, Transform parent = null) → TMP_Text`**                                | Dynamically create a `GameObject("Label")` with `TMP_Text` (localized), sets font, size, alignment. Optionally parent under `parent`. Returns the `TMP_Text` component.                                                                                |
| **`CreateButton(string text, int result, TaskCompletionSource<int> tcs) → Button`**               | Dynamically create a `GameObject("Button")` with `Button + Image + TMP_Text` (localized `text`). When clicked, calls `tcs.SetResult(result)`. Returns the `Button` so you can parent it and adjust layout.                                             |
| **`MakeResponsive(RectTransform rect, float minWidth = 0, float maxWidth = float.MaxValue)`**     | Attach a `ResponsiveElement` to `rect.gameObject` that scales the element proportionally whenever `Screen.width < minWidth` or `Screen.width > maxWidth`.                                                                                              |
| **`UpdateResponsiveLayout()`**                                                                    | Find all `ResponsiveElement` components in the scene and call their `UpdateLayout()` so they adjust their `transform.localScale` based on the current `Screen.width`.                                                                                  |
| **`ShowDialogueChoices(Dictionary<string, DialogueHelper.DialogueNode> choices) → Task<string>`** | Display a dynamic vertical list of buttons—one per key in `choices`—anchored at bottom center of screen. Returns `await`ed `string` ID corresponding to whichever button the user clicked. Automatically destroys the container once a choice is made. |
| **`CreateChoiceButton(string choiceId, string text) → Button`**                                   | Helper for `ShowDialogueChoices`: Create a `Button` with a `RectTransform(200×40)`, `Image(color=white)`, and a child `TextMeshProUGUI(text)`. Returns the `Button` so the calling code can set `onClick` behavior.                                    |

---

### Notes on `ResponsiveElement`

Attaching `ResponsiveElement` to any `GameObject` with a `RectTransform` (typically by calling `UIHelper.MakeResponsive(...)`) causes that element’s `UpdateLayout()` to be called in `UIHelper.UpdateResponsiveLayout()`. It scales the element down if the screen is narrower than `MinWidth` or wider than `MaxWidth`.

---

## 14. Final Remarks

1. **Always call `UIHelper.Initialize(...)` once at startup** (before any UI code).
2. **Use `GetFromPool` and `ReturnToPool`** for any prefabs you registered, to maximize performance.
3. **Await the asynchronous methods** (`ShowToast`, `ShowPopup`, `ShowLoadingScreen`, etc.) so you can chain logic after UI transitions complete.
4. **Localization**: Whenever you pass a `string key` into `ShowToast`, `ShowPopup`, or `CreateLabel`, it’s looked up via `LocalizationManager.Get(key)`. Make sure your localization tables contain those keys.
5. **Responsive Layout**: Call `UIHelper.UpdateResponsiveLayout()` whenever the screen resolution or orientation changes so that all `ResponsiveElement` components can re-scale themselves.

With this guide and summary table, you now have everything needed to integrate **UIHelper** into your Unity project—handling toasts, popups, spinners, overlays, dynamic buttons, responsive scaling, and more—while minimizing boilerplate and maximizing reuse.
