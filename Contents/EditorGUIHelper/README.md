Below is a complete breakdown of **EditorGUIHelper**—a static utility class (inside `#if UNITY_EDITOR`) designed to streamline the creation of custom inspectors, editor windows, and other in‐Editor GUI tasks. It provides:

* Consistent, styled **section headers** and **foldouts**
* Helpers for drawing **SerializedProperty** fields (with undo/redo and validation)
* A convenient **array field** drawer (with add/remove buttons)
* A generic **drag‐and‐drop area**
* Prebuilt **help/warning/error messages**
* A simple **search field**
* Layout helpers with spacing and indentation

Use these methods to ensure your editor UI looks polished, behaves consistently, and saves you from writing repetitive GUILayout/GUISkin code.

---

## Table of Contents

1. [Overview & Purpose](#overview--purpose)
2. [Internal Data & Fields](#internal-data--fields)
3. [Initialization (Style Cache)](#initialization-style-cache)
4. [Section Controls](#section-controls)

   * 4.1. `BeginSection(string title)`
   * 4.2. `EndSection()`
   * 4.3. `FoldoutSection(string title, bool expanded)`
5. [Property Fields](#property-fields)

   * 5.1. `PropertyField(SerializedProperty property, string label = null, string tooltip = null)`
   * 5.2. `ArrayPropertyField(SerializedProperty arrayProperty, string label = null)`
6. [Drag & Drop Area](#drag--drop-area)

   * 6.1. `DragDropArea<T>(string label, float height = 50) where T : UnityEngine.Object`
7. [Validation & Messages](#validation--messages)

   * 7.1. `HelpBox(string message, MessageType type = MessageType.Info)`
   * 7.2. `WarningMessage(string message)`
   * 7.3. `ErrorMessage(string message)`
8. [Search & Filtering](#search--filtering)

   * 8.1. `SearchField(string searchText, Action<string> onSearch = null)`
9. [Layout Helpers](#layout-helpers)

   * 9.1. `BeginHorizontalGroup(float spacing)` / `BeginHorizontalGroup()`
   * 9.2. `EndHorizontalGroup(float spacing)` / `EndHorizontalGroup()`
   * 9.3. `Indent(Action drawContent)`
10. [Real‐World Usage Examples](#real-world-usage-examples)

    * 10.1. Creating a Custom Inspector with Styled Sections
    * 10.2. Drawing a SerializedProperty with Validation & Undo
    * 10.3. Building an Array Field with Add/Remove Buttons
    * 10.4. Drag‐and‐Drop in an Editor Window
    * 10.5. Showing Help, Warning, and Error Messages
    * 10.6. Adding a Search Field to Filter Items
    * 10.7. Arranging Controls Horizontally with Spacing & Indentation
11. [Best Practices & Tips](#best-practices--tips)
12. [Summary of Public API](#summary-of-public-api)

---

## 1. Overview & Purpose

```csharp
#if UNITY_EDITOR

public static class EditorGUIHelper
{
    // … (see full code above) …
}

#endif
```

**EditorGUIHelper** packages up many common patterns you’ll use when writing custom editors, including:

* A small **GUIStyle cache** (`_styles`) to avoid recreating styles every frame
* `BeginSection`/`EndSection` to group related fields inside a boxed help‐box with a header
* A `FoldoutSection` that draws a foldable box using a custom “foldout‐bold” style
* `PropertyField` and `ArrayPropertyField` to draw `SerializedProperty` fields with undo/redo and simple validation
* A generic `DragDropArea<T>` to let users drop assets (e.g., textures, prefabs) directly onto a box
* `HelpBox`, `WarningMessage`, `ErrorMessage` to highlight validation messages in various colors
* `SearchField` to provide a toolbar‐style search input that invokes a callback when text changes
* Layout helpers (`BeginHorizontalGroup`, `EndHorizontalGroup`, `Indent`) to insert consistent spacing and indentation

All methods are wrapped in `#if UNITY_EDITOR`, so nothing appears in builds. This helper drastically reduces boilerplate, letting you focus on the “what” instead of the “how” of your editor UI.

---

## 2. Internal Data & Fields

```csharp
private static Dictionary<string, GUIStyle> _styles;
private static bool _stylesInitialized;

private static readonly float DefaultSpacing = 2f;
private static readonly float SectionSpacing = 10f;
private static readonly float IndentWidth = 15f;

private static readonly Color HeaderColor = new Color(0.6f, 0.6f, 0.6f);
private static readonly Color WarningColor = new Color(1f, 0.7f, 0.3f);
private static readonly Color ErrorColor = new Color(1f, 0.3f, 0.3f);
```

1. **Style Cache (`_styles`)**

   * A dictionary mapping string keys (`"Header"`, `"Section"`, `"Foldout"`) to preconfigured `GUIStyle` instances.
   * `_stylesInitialized` prevents reinitializing on every draw call.

2. **Layout Constants**

   * `DefaultSpacing = 2f`: Small vertical spacing between controls.
   * `SectionSpacing = 10f`: Extra space above each “section.”
   * `IndentWidth = 15f`: Horizontal indent amount when calling `Indent(...)`.

3. **Colors**

   * `HeaderColor` (dark gray), `WarningColor` (orange), and `ErrorColor` (red) used by validation message methods to tint help boxes.

---

## 3. Initialization (Style Cache)

```csharp
private static void InitializeStyles()
{
    if (_stylesInitialized) return;
    
    _styles = new Dictionary<string, GUIStyle>();
    
    // Header style
    var headerStyle = new GUIStyle(EditorStyles.boldLabel)
    {
        fontSize = 12,
        fontStyle = FontStyle.Bold,
        margin = new RectOffset(0, 0, 10, 5)
    };
    _styles["Header"] = headerStyle;
    
    // Section style
    var sectionStyle = new GUIStyle(EditorStyles.helpBox)
    {
        padding = new RectOffset(10, 10, 10, 10),
        margin = new RectOffset(0, 0, 5, 5)
    };
    _styles["Section"] = sectionStyle;
    
    // Custom foldout style
    var foldoutStyle = new GUIStyle(EditorStyles.foldout)
    {
        fontStyle = FontStyle.Bold
    };
    _styles["Foldout"] = foldoutStyle;
    
    _stylesInitialized = true;
}
```

* **When It Runs**:
  Each public helper that needs a style first calls `InitializeStyles()`, which populates `_styles` the first time. Subsequent calls do nothing.

* **Stored Styles**:

  * **`_styles["Header"]`**:

    * Based on `EditorStyles.boldLabel`
    * `fontSize = 12`, `FontStyle.Bold`
    * `margin = new RectOffset(0, 0, 10, 5)` gives extra top/bottom spacing.
  * **`_styles["Section"]`**:

    * Based on `EditorStyles.helpBox` (lightly shaded box)
    * `padding = new RectOffset(10, 10, 10, 10)` (space inside the box)
    * `margin = new RectOffset(0, 0, 5, 5)` (space outside the box).
  * **`_styles["Foldout"]`**:

    * Based on `EditorStyles.foldout`
    * `fontStyle = FontStyle.Bold` to make the foldout label bold.

These styles ensure that all “sections” and “foldouts” drawn by EditorGUIHelper look uniform across your inspectors.

---

## 4. Section Controls

### 4.1 `public static void BeginSection(string title)`

```csharp
public static void BeginSection(string title)
{
    InitializeStyles();
    
    EditorGUILayout.Space(SectionSpacing);
    EditorGUILayout.BeginVertical(_styles["Section"]);
    
    var headerRect = EditorGUILayout.GetControlRect(false, 20f);
    EditorGUI.LabelField(headerRect, title, _styles["Header"]);
    
    EditorGUILayout.Space(DefaultSpacing);
}
```

**Purpose**:
Start a new “boxed” section in your inspector or editor window. This creates a vertical group styled as a help‐box, then draws a bold, slightly larger header (20 px high) with spacing.

**Usage**:

1. **Call `InitializeStyles()`** to ensure `_styles` are available.
2. **`EditorGUILayout.Space(SectionSpacing)`** inserts 10 px of blank space before the section.
3. **`EditorGUILayout.BeginVertical(_styles["Section"])`** opens a vertical group with help‐box styling (gray background, rounded corners).
4. **`GetControlRect(false, 20f)`** reserves a 20‐pixel tall rect for the header.
5. **`EditorGUI.LabelField(headerRect, title, _styles["Header"])`** draws a bold 12‐pt header label.
6. **`EditorGUILayout.Space(DefaultSpacing)`** adds 2 px of space below the header before you start drawing fields inside this section.

**Example**:

```csharp
public override void OnInspectorGUI()
{
    serializedObject.Update();

    // Begin “General Settings” section
    EditorGUIHelper.BeginSection("General Settings");
    EditorGUIHelper.PropertyField(serializedObject.FindProperty("myString"), "My String");
    EditorGUIHelper.PropertyField(serializedObject.FindProperty("myInt"), "My Integer");
    EditorGUIHelper.EndSection();

    // Continue with other sections…
    serializedObject.ApplyModifiedProperties();
}
```

* The inspector will show a gray box labeled “General Settings,” with your two fields indented inside.

---

### 4.2 `public static void EndSection()`

```csharp
public static void EndSection()
{
    EditorGUILayout.EndVertical();
    EditorGUILayout.Space(DefaultSpacing);
}
```

**Purpose**:
Close the vertical help‐box group opened by `BeginSection`, then insert a small 2 px space below.

**Usage**:

Always call `EndSection()` after you finish adding fields in a `BeginSection` block. Failing to match begin/end will break the GUILayout hierarchy.

```csharp
EditorGUIHelper.BeginSection("Advanced Settings");
// … draw fields …
EditorGUIHelper.EndSection();
```

---

### 4.3 `public static bool FoldoutSection(string title, bool expanded)`

```csharp
public static bool FoldoutSection(string title, bool expanded)
{
    InitializeStyles();
    
    EditorGUILayout.BeginVertical(_styles["Section"]);
    expanded = EditorGUILayout.Foldout(expanded, title, true, _styles["Foldout"]);
    
    if (expanded)
    {
        EditorGUILayout.Space(DefaultSpacing);
    }
    
    return expanded;
}
```

**Purpose**:
Draw a foldable “section” inside a help‐box. Returns the updated `expanded` state so you can wrap the collapsible content accordingly.

**Parameters**:

* `title` (`string`): Label for the foldout.
* `expanded` (`bool`): Current state (true = open, false = closed).

**Behavior**:

1. Ensure styles are initialized.
2. `EditorGUILayout.BeginVertical(_styles["Section"])` opens a help‐box.
3. `EditorGUILayout.Foldout(expanded, title, true, _styles["Foldout"])` draws a bold foldout arrow and label. The `true` parameter means “use toggle style” (clickable row).
4. If `expanded == true`, insert a small vertical space (2 px) for the content that follows.
5. **Return** the new `expanded` state (the user may have clicked to toggle).

**Example**:

```csharp
private bool _showAdvanced;

public override void OnInspectorGUI()
{
    serializedObject.Update();

    // Simple field
    EditorGUIHelper.PropertyField(serializedObject.FindProperty("myString"), "My String");
    
    // Foldout for advanced settings
    _showAdvanced = EditorGUIHelper.FoldoutSection("Advanced Settings", _showAdvanced);
    if (_showAdvanced)
    {
        // Draw fields indented under the foldout
        EditorGUI.indentLevel++;
        EditorGUIHelper.PropertyField(serializedObject.FindProperty("advancedFloat"), "Advanced Float");
        EditorGUI.indentLevel--;
    }
    EditorGUILayout.EndVertical(); // close the Section from FoldoutSection

    serializedObject.ApplyModifiedProperties();
}
```

* The “Advanced Settings” header appears inside a boxed area with a fold arrow. When expanded, you can draw additional fields inside.

---

## 5. Property Fields

### 5.1 `public static void PropertyField(SerializedProperty property, string label = null, string tooltip = null)`

```csharp
public static void PropertyField(SerializedProperty property, string label = null, string tooltip = null)
{
    EditorGUI.BeginChangeCheck();
    
    if (string.IsNullOrEmpty(label))
    {
        EditorGUILayout.PropertyField(property, new GUIContent(property.displayName, tooltip));
    }
    else
    {
        EditorGUILayout.PropertyField(property, new GUIContent(label, tooltip));
    }
    
    if (EditorGUI.EndChangeCheck())
    {
        property.serializedObject.ApplyModifiedProperties();
    }
}
```

**Purpose**:
Wrap `EditorGUILayout.PropertyField` with a change‐check and automatic `ApplyModifiedProperties()`. This ensures that whenever the user edits this field, changes are recorded for undo/redo and applied to the serialized object.

**Parameters**:

* `property` (`SerializedProperty`): The property you want to draw (e.g., `serializedObject.FindProperty("myFloat")`).
* `label` (`string`, optional): Custom label text. If `null` or empty, uses `property.displayName`.
* `tooltip` (`string`, optional): Tooltip text for the field.

**Behavior**:

1. `EditorGUI.BeginChangeCheck()`: Start monitoring for changes.
2. Draw the property field with either `new GUIContent(property.displayName, tooltip)` or `new GUIContent(label, tooltip)`.
3. `if (EditorGUI.EndChangeCheck())`: If the field’s value changed, call `property.serializedObject.ApplyModifiedProperties()` to push changes to the target object and register undo.

**Example**:

```csharp
public override void OnInspectorGUI()
{
    serializedObject.Update();

    var speedProp = serializedObject.FindProperty("speed");
    EditorGUIHelper.PropertyField(speedProp, "Movement Speed", "Speed in units/sec");

    var nameProp = serializedObject.FindProperty("objectName");
    EditorGUIHelper.PropertyField(nameProp); // uses displayName automatically

    serializedObject.ApplyModifiedProperties();
}
```

* If the user edits “Movement Speed,” the change is recorded immediately (undoable) and pushed to the serialized target.

---

### 5.2 `public static void ArrayPropertyField(SerializedProperty arrayProperty, string label = null)`

```csharp
public static void ArrayPropertyField(SerializedProperty arrayProperty, string label = null)
{
    if (!arrayProperty.isArray) return;
    
    EditorGUILayout.BeginVertical();
    
    // Header with size field
    EditorGUILayout.BeginHorizontal();
    if (string.IsNullOrEmpty(label))
    {
        EditorGUILayout.PropertyField(arrayProperty);
    }
    else
    {
        EditorGUILayout.PropertyField(arrayProperty, new GUIContent(label));
    }
    EditorGUILayout.EndHorizontal();
    
    // Array elements
    if (arrayProperty.isExpanded)
    {
        EditorGUI.indentLevel++;
        
        for (int i = 0; i < arrayProperty.arraySize; i++)
        {
            EditorGUILayout.BeginHorizontal();
            EditorGUILayout.PropertyField(arrayProperty.GetArrayElementAtIndex(i));
            
            if (GUILayout.Button("-", GUILayout.Width(20)))
            {
                arrayProperty.DeleteArrayElementAtIndex(i);
                break;
            }
            EditorGUILayout.EndHorizontal();
        }
        
        // Add button
        if (GUILayout.Button("Add Element"))
        {
            arrayProperty.arraySize++;
        }
        
        EditorGUI.indentLevel--;
    }
    
    EditorGUILayout.EndVertical();
}
```

**Purpose**:
Draw a foldable array property with built‐in “–” (remove) buttons for each element and an “Add Element” button at the bottom. Automatically handles `arrayProperty.isExpanded`, indentation, and property drawing.

**Parameters**:

* `arrayProperty` (`SerializedProperty`): Must be an array or `List<T>` property.
* `label` (`string`, optional): Custom label for the array field; if null, uses the property’s default label.

**Behavior**:

1. **Guard**:

   * `if (!arrayProperty.isArray) return;` ensures you only call this on arrays/lists.

2. **Begin a vertical group** (so add/remove buttons and elements stack neatly).

3. **Header Row**:

   * `EditorGUILayout.BeginHorizontal();`
   * Draw `PropertyField(arrayProperty, label)`—this shows “MyArray (size)” in a foldout if it’s expanded.
   * `EditorGUILayout.EndHorizontal();`

4. **If `arrayProperty.isExpanded` (user clicked the foldout arrow)**:

   * `EditorGUI.indentLevel++` to indent array elements.
   * Loop over `i = 0 … arraySize - 1`:

     * `EditorGUILayout.BeginHorizontal();`
     * `EditorGUILayout.PropertyField(arrayProperty.GetArrayElementAtIndex(i))` draws the element’s field.
     * If user clicks `"-"` button (20 px wide), call `arrayProperty.DeleteArrayElementAtIndex(i)` and `break` out of the loop (to avoid invalidating the loop due to changed `arraySize`).
     * `EditorGUILayout.EndHorizontal();`
   * After listing all elements, draw an “Add Element” button. If clicked, `arrayProperty.arraySize++`.
   * `EditorGUI.indentLevel--` to restore original indentation.

5. **End the vertical group** with `EditorGUILayout.EndVertical()`.

**Example**:

```csharp
public override void OnInspectorGUI()
{
    serializedObject.Update();

    var myListProp = serializedObject.FindProperty("enemyPrefabs"); // e.g., List<GameObject>
    EditorGUIHelper.ArrayPropertyField(myListProp, "Enemy Prefabs");

    serializedObject.ApplyModifiedProperties();
}
```

* In the inspector, you’ll see:

  ```
  Enemy Prefabs (size 3)    [foldout arrow]
   └ Item 0    [object field]        [-]
   └ Item 1    [object field]        [-]
   └ Item 2    [object field]        [-]
      [Add Element button]
  ```

* Clicking “–” next to an item deletes it. “Add Element” appends a new `null` slot to the array.

---

## 6. Drag & Drop Area

### 6.1 `public static T DragDropArea<T>(string label, float height = 50) where T : UnityEngine.Object`

```csharp
public static T DragDropArea<T>(string label, float height = 50) where T : UnityEngine.Object
{
    Event evt = Event.current;
    var dropArea = GUILayoutUtility.GetRect(0.0f, height, GUILayout.ExpandWidth(true));
    var style = new GUIStyle(GUI.skin.box)
    {
        alignment = TextAnchor.MiddleCenter
    };
    
    GUI.Box(dropArea, label, style);
    
    switch (evt.type)
    {
        case EventType.DragUpdated:
        case EventType.DragPerform:
            if (!dropArea.Contains(evt.mousePosition))
                return null;
            
            DragAndDrop.visualMode = DragAndDropVisualMode.Copy;
            
            if (evt.type == EventType.DragPerform)
            {
                DragAndDrop.AcceptDrag();
                
                foreach (var draggedObject in DragAndDrop.objectReferences)
                {
                    if (draggedObject is T matchingObject)
                        return matchingObject;
                }
            }
            break;
    }
    
    return null;
}
```

**Purpose**:
Draws a rectangular box in the inspector or editor window that says `label` (centered). If the user drags any asset of type `T` over it (e.g., `Texture2D`, `GameObject`, etc.), then on drop (`DragPerform`) returns the first matching object. Otherwise, returns `null`.

**Parameters**:

* `T` (`UnityEngine.Object` subclass): The type of object you want to accept (e.g., `Texture2D`, `GameObject`, `MonoScript`).
* `label` (`string`): Text to show inside the box (e.g., “Drag Texture Here”).
* `height` (`float`): Height of the drop area in pixels (default 50).

**Behavior**:

1. **`GUILayoutUtility.GetRect(0.0f, height, GUILayout.ExpandWidth(true))`**:

   * Reserves a rectangle whose height is `height` and expands to available width.
   * Stored as `Rect dropArea`.

2. **Draw a Box**:

   * `GUI.Box(dropArea, label, style)` draws a simple box with the given label centered.

3. **Handle Drag Events** (`Event.current`):

   * If `evt.type` is `DragUpdated` or `DragPerform`:

     * If `dropArea.Contains(evt.mousePosition)` (mouse is over the box):

       * `DragAndDrop.visualMode = DragAndDropVisualMode.Copy` changes the cursor to indicate copy operation.
       * If `evt.type == DragPerform`:

         * `DragAndDrop.AcceptDrag()` to finalize the drag.
         * Iterate through `DragAndDrop.objectReferences` (these are the dragged objects):

           * If `draggedObject is T matchingObject`, return that object. (First match.)
   * Outside these cases, do nothing.

4. **Return**:

   * If no matching drop occurred, return `null`.
   * Caller can check for a non‐null return and assign it to a field.

**Example**:

```csharp
public override void OnInspectorGUI()
{
    serializedObject.Update();

    // Draw a drag area to accept a Texture2D
    Texture2D droppedTex = EditorGUIHelper.DragDropArea<Texture2D>("Drag a Texture Here", 60f);
    if (droppedTex != null)
    {
        var texProp = serializedObject.FindProperty("myTexture");
        texProp.objectReferenceValue = droppedTex;
        serializedObject.ApplyModifiedProperties();
    }

    // Alternatively, assign to a non‐serialized field:
    if (droppedTex != null)
    {
        Debug.Log($"Dropped texture: {droppedTex.name}");
    }

    serializedObject.ApplyModifiedProperties();
}
```

* When you drag a texture from the Project window over the box and release, `DragDropArea` returns that `Texture2D` instance. You then assign it to a `SerializedProperty` or store it in a field.

---

## 7. Validation & Messages

### 7.1 `public static void HelpBox(string message, MessageType type = MessageType.Info)`

```csharp
public static void HelpBox(string message, MessageType type = MessageType.Info)
{
    EditorGUILayout.HelpBox(message, type);
}
```

**Purpose**:
Wraps `EditorGUILayout.HelpBox` so you don’t have to specify a `MessageType` each time. By default, displays an “Info” help‐box (blue info icon). You can override `type` to `MessageType.Warning` or `MessageType.Error`.

**Example**:

```csharp
EditorGUIHelper.HelpBox("This is an informational tip.", MessageType.Info);
EditorGUIHelper.HelpBox("Be careful! Value is out of range.", MessageType.Warning);
```

---

### 7.2 `public static void WarningMessage(string message)`

```csharp
public static void WarningMessage(string message)
{
    var oldColor = GUI.color;
    GUI.color = WarningColor;
    EditorGUILayout.HelpBox(message, MessageType.Warning);
    GUI.color = oldColor;
}
```

**Purpose**:
Draws a warning‐style help‐box tinted with `WarningColor` (orange) so the warning stands out more. After drawing, resets `GUI.color` to its previous value.

**Example**:

```csharp
if (someValue < 0)
{
    EditorGUIHelper.WarningMessage("Value cannot be negative.");
}
```

---

### 7.3 `public static void ErrorMessage(string message)`

```csharp
public static void ErrorMessage(string message)
{
    var oldColor = GUI.color;
    GUI.color = ErrorColor;
    EditorGUILayout.HelpBox(message, MessageType.Error);
    GUI.color = oldColor;
}
```

**Purpose**:
Draws an error‐style help‐box tinted with `ErrorColor` (red). Use it to highlight critical issues in your data.

**Example**:

```csharp
if (requiredReference == null)
{
    EditorGUIHelper.ErrorMessage("A required asset reference is missing!");
}
```

---

## 8. Search & Filtering

### 8.1 `public static string SearchField(string searchText, Action<string> onSearch = null)`

```csharp
public static string SearchField(string searchText, Action<string> onSearch = null)
{
    EditorGUILayout.BeginHorizontal();
    
    var newSearchText = EditorGUILayout.TextField("Search", searchText, EditorStyles.toolbarSearchField);
    
    if (newSearchText != searchText)
    {
        onSearch?.Invoke(newSearchText);
    }
    
    if (GUILayout.Button("×", EditorStyles.toolbarButton, GUILayout.Width(20)) && !string.IsNullOrEmpty(searchText))
    {
        newSearchText = "";
        onSearch?.Invoke("");
        GUI.FocusControl(null);
    }
    
    EditorGUILayout.EndHorizontal();
    
    return newSearchText;
}
```

**Purpose**:
Create a search bar (toolbar style) with a clear (“×”) button on the right. Invokes `onSearch(newSearchText)` whenever the text changes (including when the user clicks “×”).

**Parameters**:

* `searchText` (`string`): Current search text.
* `onSearch` (`Action<string>`, optional): Callback invoked when text changes (passing new text).

**Behavior**:

1. Start a horizontal group (`BeginHorizontal()`).
2. Draw a text field with label `"Search"` and style `EditorStyles.toolbarSearchField`. Return the updated string into `newSearchText`.
3. If the text changed (`newSearchText != searchText`), call `onSearch?.Invoke(newSearchText)`.
4. Draw a small “×” button (`GUILayout.Button("×", EditorStyles.toolbarButton, GUILayout.Width(20))`). If clicked and `searchText` wasn’t already empty:

   * Reset `newSearchText = ""`.
   * Call `onSearch?.Invoke("")`.
   * `GUI.FocusControl(null)` to remove focus from the text field (so it loses the blinking cursor).
5. `EndHorizontal()`.
6. Return `newSearchText` so the caller can store it for the next frame.

**Example**:

```csharp
private string _searchQuery = "";

public override void OnInspectorGUI()
{
    serializedObject.Update();

    _searchQuery = EditorGUIHelper.SearchField(_searchQuery, query =>
    {
        // Filter your list of items based on `query`
        FilterMyList(query);
    });

    // Draw filtered items here…

    serializedObject.ApplyModifiedProperties();
}
```

* Each time the user types or clears, `FilterMyList(query)` runs to refresh the displayed list.

---

## 9. Layout Helpers

### 9.1 `public static void BeginHorizontalGroup(float spacing)` / `public static void BeginHorizontalGroup()`

```csharp
public static void BeginHorizontalGroup(float spacing)
{
    EditorGUILayout.BeginHorizontal();
    GUILayout.Space(spacing);
}

public static void BeginHorizontalGroup()
{
    BeginHorizontalGroup(DefaultSpacing);
}
```

**Purpose**:
Open a horizontal group and insert a left margin (`spacing`) before drawing child controls.

**Parameters**:

* `spacing` (`float`): Number of pixels to indent on the left (default `2f`).

**Usage**:

```csharp
EditorGUIHelper.BeginHorizontalGroup(20f); // begin a row with 20 px indentation
GUILayout.Label("Indented Label");
EditorGUILayout.EndHorizontal();
```

---

### 9.2 `public static void EndHorizontalGroup(float spacing)` / `public static void EndHorizontalGroup()`

```csharp
public static void EndHorizontalGroup(float spacing)
{
    GUILayout.Space(spacing);
    EditorGUILayout.EndHorizontal();
}

public static void EndHorizontalGroup()
{
    EndHorizontalGroup(DefaultSpacing);
}
```

**Purpose**:
End a horizontal group and insert a right margin (`spacing`) after drawing child controls.

**Example**:

```csharp
EditorGUIHelper.BeginHorizontalGroup(10f);
// … draw controls side by side …
EditorGUIHelper.EndHorizontalGroup(10f);
```

---

### 9.3 `public static void Indent(Action drawContent)`

```csharp
public static void Indent(Action drawContent)
{
    EditorGUI.indentLevel++;
    drawContent?.Invoke();
    EditorGUI.indentLevel--;
}
```

**Purpose**:
Temporarily increase the built‐in `EditorGUI.indentLevel`, invoke `drawContent()` to draw whatever you need, then restore the indent level. This is useful for hierarchical fields.

**Example**:

```csharp
EditorGUIHelper.Indent(() =>
{
    EditorGUIHelper.PropertyField(serializedObject.FindProperty("nestedFloat"), "Nested Float");
    EditorGUIHelper.PropertyField(serializedObject.FindProperty("nestedString"), "Nested String");
});
```

* Both fields will be drawn one indentation level to the right, matching Unity’s usual nested formatting.

---

## 10. Real‐World Usage Examples

Below are concrete examples demonstrating how you might integrate **EditorGUIHelper** into your custom inspectors and editor windows.

---

### 10.1 Creating a Custom Inspector with Styled Sections

```csharp
using UnityEditor;
using UnityEngine;

[CustomEditor(typeof(MyComponent))]
public class MyComponentEditor : Editor
{
    SerializedProperty _speedProp;
    SerializedProperty _healthProp;
    SerializedProperty _abilitiesProp;

    private bool _showAdvanced = false;

    private void OnEnable()
    {
        _speedProp = serializedObject.FindProperty("speed");
        _healthProp = serializedObject.FindProperty("health");
        _abilitiesProp = serializedObject.FindProperty("abilities"); // a List<string>
    }

    public override void OnInspectorGUI()
    {
        serializedObject.Update();

        // Basic Settings Section
        EditorGUIHelper.BeginSection("Basic Settings");
        EditorGUIHelper.PropertyField(_speedProp, "Movement Speed", "Units per second");
        EditorGUIHelper.PropertyField(_healthProp, "Max Health", "Hit points before death");
        EditorGUIHelper.EndSection();

        // Advanced Settings Foldout
        _showAdvanced = EditorGUIHelper.FoldoutSection("Advanced Settings", _showAdvanced);
        if (_showAdvanced)
        {
            EditorGUI.indentLevel++;
            // Draw abilities array
            EditorGUIHelper.ArrayPropertyField(_abilitiesProp, "Abilities");
            EditorGUI.indentLevel--;
        }
        // Close the section started by FoldoutSection
        if (_showAdvanced || !_showAdvanced)
            EditorGUILayout.EndVertical();

        serializedObject.ApplyModifiedProperties();
    }
}
```

* **Result**:

  * A “Basic Settings” box with two fields: “Movement Speed” and “Max Health.”
  * Below, an “Advanced Settings” box with a fold arrow. When expanded, “Abilities” (an array) is drawn with individual element fields and “–/Add Element” buttons.

---

### 10.2 Drawing a SerializedProperty with Validation & Undo

```csharp
using UnityEditor;
using UnityEngine;

[CustomEditor(typeof(PlayerStats))]
public class PlayerStatsEditor : Editor
{
    SerializedProperty _levelProp;
    SerializedProperty _nameProp;

    private void OnEnable()
    {
        _levelProp = serializedObject.FindProperty("level");
        _nameProp = serializedObject.FindProperty("playerName");
    }

    public override void OnInspectorGUI()
    {
        serializedObject.Update();

        // Player Name
        EditorGUIHelper.PropertyField(_nameProp, "Player Name");
        if (string.IsNullOrEmpty(_nameProp.stringValue))
        {
            EditorGUIHelper.WarningMessage("Player name should not be empty!");
        }

        // Level
        EditorGUIHelper.PropertyField(_levelProp, "Character Level");
        if (_levelProp.intValue < 1 || _levelProp.intValue > 100)
        {
            EditorGUIHelper.ErrorMessage("Level must be between 1 and 100.");
        }

        serializedObject.ApplyModifiedProperties();
    }
}
```

* **What Happens**:

  * If `playerName` is empty, a warning help‐box (orange) appears.
  * If `level` is outside `[1…100]`, an error help‐box (red) appears.
  * All changes to fields are automatically registered for undo/redo and immediately applied.

---

### 10.3 Building an Array Field with Add/Remove Buttons

```csharp
using UnityEditor;
using UnityEngine;

[CustomEditor(typeof(SpawnManager))]
public class SpawnManagerEditor : Editor
{
    SerializedProperty _spawnPointsProp; // an array of Transform references

    private void OnEnable()
    {
        _spawnPointsProp = serializedObject.FindProperty("spawnPoints");
    }

    public override void OnInspectorGUI()
    {
        serializedObject.Update();

        EditorGUIHelper.BeginSection("Spawn Points");
        EditorGUIHelper.ArrayPropertyField(_spawnPointsProp, "Spawn Transforms");
        EditorGUIHelper.EndSection();

        serializedObject.ApplyModifiedProperties();
    }
}
```

* **Result**:

  * A “Spawn Points” section containing a foldout labeled “Spawn Transforms (size X).”
  * When expanded, each element appears with a `Transform` object field plus a “–” button to remove it.
  * An “Add Element” button at the bottom increases the array size by one.

---

### 10.4 Drag‐and‐Drop in an Editor Window

```csharp
using UnityEditor;
using UnityEngine;

public class TexturePickerWindow : EditorWindow
{
    private Texture2D _pickedTexture;

    [MenuItem("Window/Texture Picker")]
    public static void ShowWindow()
    {
        GetWindow<TexturePickerWindow>("Texture Picker");
    }

    private void OnGUI()
    {
        GUILayout.Label("Drop a texture below:", EditorStyles.boldLabel);
        _pickedTexture = EditorGUIHelper.DragDropArea<Texture2D>("Drag Texture Here", 80f);

        if (_pickedTexture != null)
        {
            GUILayout.Space(10);
            GUILayout.Label("Selected Texture:", EditorStyles.label);
            GUILayout.Label(AssetPreview.GetAssetPreview(_pickedTexture), GUILayout.Width(80), GUILayout.Height(80));
        }
    }
}
```

* **What It Does**:

  * Creates a standalone editor window titled “Texture Picker.”
  * The user sees a large box “Drag Texture Here.”
  * When they drag any `Texture2D` asset over it and release, `_pickedTexture` is set.
  * Then a preview of the texture is displayed below.

---

### 10.5 Showing Help, Warning, and Error Messages

```csharp
[CustomEditor(typeof(ItemDatabase))]
public class ItemDatabaseEditor : Editor
{
    public override void OnInspectorGUI()
    {
        serializedObject.Update();

        var itemsProp = serializedObject.FindProperty("items"); // a List<Item>
        if (itemsProp == null || itemsProp.arraySize == 0)
        {
            EditorGUIHelper.HelpBox("No items defined. Please add at least one item.", MessageType.Info);
        }
        else if (itemsProp.arraySize < 5)
        {
            EditorGUIHelper.WarningMessage("Fewer than 5 items may result in limited gameplay variety.");
        }

        // Suppose the script requires a GameObject reference
        var prefabProp = serializedObject.FindProperty("itemPrefab");
        EditorGUIHelper.PropertyField(prefabProp, "Item Prefab");
        if (prefabProp.objectReferenceValue == null)
        {
            EditorGUIHelper.ErrorMessage("Item Prefab is required for the database to function.");
        }

        serializedObject.ApplyModifiedProperties();
    }
}
```

* **Result**:

  * If no items exist, an “Info” help‐box appears.
  * If fewer than 5 items, an orange “Warning” box appears.
  * If `itemPrefab` is null, a red “Error” box appears under that field.

---

### 10.6 Adding a Search Field to Filter Items

```csharp
[CustomEditor(typeof(ItemDatabase))]
public class ItemDatabaseEditor : Editor
{
    private string _searchFilter = "";

    public override void OnInspectorGUI()
    {
        serializedObject.Update();

        // Search bar
        _searchFilter = EditorGUIHelper.SearchField(_searchFilter, query =>
        {
            // e.g., filter `ItemDatabase.items` where item.name contains query
            ((ItemDatabase)target).FilterItems(query);
        });

        // Draw filtered items (implementation of FilterItems not shown)
        for (int i = 0; i < ((ItemDatabase)target).filteredItems.Count; i++)
        {
            EditorGUILayout.LabelField(((ItemDatabase)target).filteredItems[i].Name);
        }

        serializedObject.ApplyModifiedProperties();
    }
}
```

* Each time the user types or clears the search field, `FilterItems(query)` is called to update `filteredItems`. The inspector then redraws that filtered list.

---

### 10.7 Arranging Controls Horizontally with Spacing & Indentation

```csharp
[CustomEditor(typeof(UISpacingDemo))]
public class UISpacingDemoEditor : Editor
{
    public override void OnInspectorGUI()
    {
        serializedObject.Update();

        // Horizontal group with default spacing
        EditorGUIHelper.BeginHorizontalGroup();
        EditorGUILayout.LabelField("Left Label");
        GUILayout.FlexibleSpace();
        EditorGUILayout.LabelField("Right Label");
        EditorGUIHelper.EndHorizontalGroup();

        // Indented group
        EditorGUIHelper.Indent(() =>
        {
            EditorGUILayout.LabelField("Indented Label 1");
            EditorGUILayout.LabelField("Indented Label 2");
        });

        serializedObject.ApplyModifiedProperties();
    }
}
```

* **What You See**:

  * A row with “Left Label” on the left and “Right Label” on the right, separated by flexible space, with 2 px left/right margins.
  * Below, two labels indented by one indent level (15 px).

---

## 11. Best Practices & Tips

1. **Always Wrap Style Access in `InitializeStyles()`**

   * Any method before using `_styles["Header"]` or `Foldout` calls `InitializeStyles()`, ensuring styles are created only once.

2. **Match `BeginSection`/`EndSection` and `BeginVertical`/`EndVertical` Pairs**

   * Forgetting `EditorGUILayout.EndVertical()` after a `BeginSection` will break your layout. IDE warnings often appear if counts do not match.

3. **Use `PropertyField` Instead of Directly Calling `EditorGUILayout.PropertyField`**

   * This ensures undo/redo works and calls `ApplyModifiedProperties()` automatically only when needed. Saves boilerplate.

4. **When Editing Arrays, Always Check `isExpanded`**

   * `ArrayPropertyField` only draws elements if `arrayProperty.isExpanded` is true. That foldout arrow is part of the default `PropertyField(arrayProperty)` drawer.

5. **`DragDropArea<T>` Only Returns the First Matching Object**

   * If you drag multiple objects at once, it returns only the first that matches `T`. You can modify it to collect all matches (e.g., returning a `List<T>`).

6. **Reset GUI Color After Changing It**

   * `ErrorMessage` and `WarningMessage` manually change `GUI.color` then restore it. Always restore colors after drawing.

7. **Leverage `SearchField`’s Callback to Filter Data Immediately**

   * The moment the user types or clears, `onSearch` fires with the new text. Update your displayed list on the fly.

8. **Use `Indent` for Cleaner Nested Layouts**

   * Instead of manually adjusting `EditorGUI.indentLevel`, wrap related draws in `EditorGUIHelper.Indent(() => { … })`.

9. **Avoid Deep Nesting of Horizontal/Vertical Groups**

   * Each group adds overhead. Keep your inspectors reasonably flat where possible.

10. **Remember That All of These Helpers Only Exist in the Editor**

    * Because the entire class is wrapped in `#if UNITY_EDITOR`, none of this code will be included in a runtime build.

---

## 12. Summary of Public API

| Method Signature                                                                                            | Description                                                                                                                 |
| ----------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `public static void BeginSection(string title)`                                                             | Start a new boxed section with a bold header.                                                                               |
| `public static void EndSection()`                                                                           | Close the boxed section opened by `BeginSection` and add a small space.                                                     |
| `public static bool FoldoutSection(string title, bool expanded)`                                            | Draw a foldable section header inside a box. Returns updated `expanded` state.                                              |
| `public static void PropertyField(SerializedProperty property, string label = null, string tooltip = null)` | Draw a property field (with optional custom label/tooltip), wrapping change‐check and `ApplyModifiedProperties()`.          |
| `public static void ArrayPropertyField(SerializedProperty arrayProperty, string label = null)`              | Draw a foldable array field with remove (“–”) buttons per element and “Add Element” button at bottom.                       |
| `public static T DragDropArea<T>(string label, float height = 50) where T : UnityEngine.Object`             | Draw a box labeled `label` that accepts drag‐and‐drop for assets of type `T`. Returns the dropped object or `null`.         |
| `public static void HelpBox(string message, MessageType type = MessageType.Info)`                           | Draw a standard Unity help box (info, warning, or error).                                                                   |
| `public static void WarningMessage(string message)`                                                         | Draw a warning‐style help box tinted orange.                                                                                |
| `public static void ErrorMessage(string message)`                                                           | Draw an error‐style help box tinted red.                                                                                    |
| `public static string SearchField(string searchText, Action<string> onSearch = null)`                       | Create a search‐bar (toolbar style) with a clear (“×”) button. Returns updated text and invokes `onSearch` when it changes. |
| `public static void BeginHorizontalGroup(float spacing)` / `public static void BeginHorizontalGroup()`      | Begin a horizontal group, adding a left margin of `spacing`. Default spacing = 2 px.                                        |
| `public static void EndHorizontalGroup(float spacing)` / `public static void EndHorizontalGroup()`          | End a horizontal group, adding a right margin of `spacing`.                                                                 |
| `public static void Indent(Action drawContent)`                                                             | Increment `EditorGUI.indentLevel`, execute `drawContent()`, then restore the indent level.                                  |

By using **EditorGUIHelper**, your custom inspectors and editor windows will have a consistent, polished look, and you’ll spend less time writing boilerplate code. Each helper covers a common pattern—styled sections, array management, drag‐and‐drop, validation messages, search fields, and layout—so your editor scripts remain focused and concise.
