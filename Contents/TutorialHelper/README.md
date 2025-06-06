Below is an organized overview of the entire “Tutorial” feature, broken down by folder and file, with a consolidated table of all classes, their key methods/properties, and what they do. The goal is to give you a bird’s-eye view of how the editor-side assets tie into the runtime system, how data flows, and what entry points you’ll use when integrating or extending this tutorial framework.

---

## Folder Structure

```
/Assets
  /Editor
    TutorialDefinition.cs
    TutorialCategory.cs
    TutorialEditorWindow.cs
    TutorialTreeView.cs
  /Scripts (Runtime)
    TutorialHelper.cs
    TutorialRepository.cs
    … (other utility/helper classes)
```

---

## 1. Editor‐Side (Assets/Editor)

These ScriptableObjects and UI windows let you create, edit, and save tutorial definitions and categories in the Unity Editor. When you press “Save” in the Editor window, they serialize out to `.asset` files under `Assets/Resources/Tutorials`, and a separate JSON (`tutorial_definitions.json` / `tutorial_categories.json`) is kept in Resources for runtime loading.

### 1.1. `TutorialDefinition.cs`

**Purpose**: A `ScriptableObject` that wraps runtime `TutorialRepository.TutorialData` for the Editor. Each asset represents one tutorial, including ID, title, steps, dependencies, etc.

```csharp
namespace UnityHelperSDK.Editor
{
    public class TutorialDefinition : ScriptableObject
    {
        [SerializeField] private string id;
        [SerializeField] private string categoryId;
        [SerializeField] private string title;
        [SerializeField] private string description;
        [SerializeField] private bool onlyShowOnce = true;
        [SerializeField] private int requiredLevel;
        [SerializeField] private List<string> dependencies = new List<string>();
        [SerializeField] private List<string> startConditions = new List<string>();
        [SerializeField] private List<TutorialStepData> steps = new List<TutorialStepData>();

        public string Id => id;
        public string CategoryId => categoryId;
        public string Title => title;
        public string Description => description;
        public bool OnlyShowOnce => onlyShowOnce;
        public int RequiredLevel => requiredLevel;
        public List<string> Dependencies => dependencies;
        public List<string> StartConditions => startConditions;
        public List<TutorialStepData> Steps => steps;

        public void Initialize(string newId, string catId, string newTitle = "", string newDescription = "", int reqLevel = 1, bool showOnce = true) { ... }

        // Convert to runtime data
        public TutorialRepository.TutorialData ToRuntimeData() { ... }

        // Create a new ScriptableObject from runtime data
        public static TutorialDefinition FromRuntimeData(TutorialRepository.TutorialData data) { ... }
    }

    [Serializable]
    public class TutorialStepData
    {
        public string id;
        public string dialogueKey;
        public List<string> conditions;
        public string completionCondition;
    }
}
```

* **Key Methods/Properties**

  | Member                                           | Type     | Description                                                                     |
  | ------------------------------------------------ | -------- | ------------------------------------------------------------------------------- |
  | `string Id`                                      | Property | Unique identifier for this tutorial.                                            |
  | `string CategoryId`                              | Property | Category asset’s ID that this tutorial belongs to.                              |
  | `string Title`                                   | Property | Editor‐visible title for the tutorial.                                          |
  | `string Description`                             | Property | A brief description shown in the Editor.                                        |
  | `bool OnlyShowOnce`                              | Property | If true, runtime will skip if already completed.                                |
  | `int RequiredLevel`                              | Property | Minimum player level to trigger this tutorial.                                  |
  | `List<string> Dependencies`                      | Property | Other tutorial IDs that must be finished first.                                 |
  | `List<string> StartConditions`                   | Property | Custom condition IDs that gate whether to start this tutorial.                  |
  | `List<TutorialStepData> Steps`                   | Property | Sequence of steps (each with id, dialogueKey, conditions, completionCondition). |
  | `void Initialize(...)`                           | Method   | Editor helper to populate new instance.                                         |
  | `TutorialData ToRuntimeData()`                   | Method   | Creates a `TutorialRepository.TutorialData` from this ScriptableObject.         |
  | `static TutorialDefinition FromRuntimeData(...)` | Method   | Creates a new SO from existing runtime data (used during “Refresh” in Editor).  |

---

### 1.2. `TutorialCategory.cs`

**Purpose**: A `ScriptableObject` that wraps runtime `TutorialRepository.TutorialCategoryData`. Each asset represents one category (e.g. “Movement Tutorials,” “Combat Tutorials,” etc.).

```csharp
namespace UnityHelperSDK.Editor
{
    public class TutorialCategory : ScriptableObject
    {
        [SerializeField] private string id;
        [SerializeField] private string displayName;
        [SerializeField] private string description;
        [SerializeField] private int sortOrder;
        [SerializeField] private List<string> tutorialIds = new List<string>();

        public string Id => id;
        public string Name => displayName;
        public string Description => description;
        public int SortOrder => sortOrder;
        public List<string> TutorialIds => tutorialIds;

        public void Initialize(string newId, string newName = "", string newDescription = "", int order = 0) { ... }

        public TutorialRepository.TutorialCategoryData ToRuntimeData() { ... }

        public static TutorialCategory FromRuntimeData(TutorialRepository.TutorialCategoryData data) { ... }
    }
}
```

* **Key Methods/Properties**

  | Member                                         | Type     | Description                                         |
  | ---------------------------------------------- | -------- | --------------------------------------------------- |
  | `string Id`                                    | Property | Unique category ID (used as key in JSON).           |
  | `string Name`                                  | Property | Editor‐visible display name.                        |
  | `string Description`                           | Property | Optional description for this category.             |
  | `int SortOrder`                                | Property | Determines ordering in the TreeView.                |
  | `List<string> TutorialIds`                     | Property | List of tutorial IDs belonging to this category.    |
  | `void Initialize(string, string, string, int)` | Method   | Editor helper to initialize a new category.         |
  | `TutorialCategoryData ToRuntimeData()`         | Method   | Convert to runtime data format.                     |
  | `static TutorialCategory FromRuntimeData(...)` | Method   | Create a new SO from runtime data during “Refresh.” |

---

### 1.3. `TutorialTreeView.cs`

**Purpose**: A custom `VisualElement` (UIElements) that displays categories and their tutorials in a hierarchical `TreeView`. Selecting an entry invokes an event to show its details in the Inspector panel.

```csharp
namespace UnityHelperSDK.Editor
{
    public class TutorialTreeView : VisualElement
    {
        public delegate void TutorialSelectionChangedHandler(string categoryId, string tutorialId);
        public event TutorialSelectionChangedHandler OnTutorialSelectionChanged;

        private TreeView _treeView;
        private Dictionary<string, TutorialCategory> _categories;
        private Dictionary<string, TutorialDefinition> _tutorials;
        private List<TreeViewItemData<TutorialItemData>> _items;

        private const float ITEM_HEIGHT = 24f;

        public TutorialTreeView(Dictionary<string, TutorialCategory> categories, Dictionary<string, TutorialDefinition> tutorials)
        {
            _categories = categories;
            _tutorials = tutorials;
            style.flexGrow = 1;

            _treeView = new TreeView {
                fixedItemHeight = ITEM_HEIGHT,
                selectionType = SelectionType.Single,
                showBorder = true,
                showAlternatingRowBackgrounds = AlternatingRowBackground.All
            };
            _treeView.style.flexGrow = 1;
            _treeView.viewDataKey = "TutorialTreeView";

            // Setup how each label is created and bound
            _treeView.makeItem = () => CreateLabelItem();
            _treeView.bindItem = BindLabelItem;
            _treeView.selectionChanged += OnTreeSelectionChanged;

            Add(_treeView);
            Refresh(_categories, _tutorials);
        }

        public void Refresh(Dictionary<string, TutorialCategory> categories, Dictionary<string, TutorialDefinition> tutorials)
        {
            _categories = categories;
            _tutorials = tutorials;

            var items = new List<TreeViewItemData<TutorialItemData>>();
            int idCounter = 0;

            // Root node: “Tutorials”
            var root = new TreeViewItemData<TutorialItemData>(
                idCounter++,
                new TutorialItemData { displayName = "Tutorials", categoryId = "", tutorialId = "" },
                new List<TreeViewItemData<TutorialItemData>>()
            );

            // Build category → tutorial children
            foreach (var cat in _categories.Values.OrderBy(c => c.SortOrder))
            {
                var tutorialChildren = new List<TreeViewItemData<TutorialItemData>>();
                if (cat.TutorialIds != null)
                {
                    foreach (var tid in cat.TutorialIds)
                    {
                        if (_tutorials.TryGetValue(tid, out var td))
                        {
                            tutorialChildren.Add(new TreeViewItemData<TutorialItemData>(
                                idCounter++,
                                new TutorialItemData {
                                    displayName = td.Title ?? td.Id,
                                    categoryId = cat.Id,
                                    tutorialId = td.Id
                                }
                            ));
                        }
                    }
                }

                root.AddChild(new TreeViewItemData<TutorialItemData>(
                    idCounter++,
                    new TutorialItemData {
                        displayName = cat.Name,
                        categoryId = cat.Id,
                        tutorialId = ""
                    },
                    tutorialChildren
                ));
            }

            items.Add(root);
            _items = items;
            _treeView.SetRootItems(_items);
            _treeView.Rebuild();
        }

        private void OnTreeSelectionChanged(IEnumerable<object> selection)
        {
            int index = (int)selection.FirstOrDefault();
            var data = _treeView.GetItemDataForIndex<TutorialItemData>(index);
            if (data != null && !string.IsNullOrEmpty(data.tutorialId))
                OnTutorialSelectionChanged?.Invoke(data.categoryId, data.tutorialId);
        }

        private Label CreateLabelItem()
        {
            var lbl = new Label();
            lbl.style.paddingLeft = 4;
            lbl.style.unityTextAlign = TextAnchor.MiddleLeft;
            lbl.style.height = ITEM_HEIGHT;
            return lbl;
        }

        private void BindLabelItem(VisualElement element, int index)
        {
            var lbl = (Label)element;
            var data = _treeView.GetItemDataForIndex<TutorialItemData>(index);
            lbl.text = data.displayName;
            if (string.IsNullOrEmpty(data.tutorialId))
            {
                // Category styling
                lbl.style.color = new Color(0.7f, 0.7f, 0.7f);
                lbl.style.unityFontStyleAndWeight = FontStyle.Bold;
            }
            else
            {
                lbl.style.color = Color.white;
                lbl.style.unityFontStyleAndWeight = FontStyle.Normal;
            }
        }

        private class TutorialItemData
        {
            public string displayName;
            public string categoryId;
            public string tutorialId;
        }
    }
}
```

* **Key Methods/Properties**

  | Member                                                             | Type           | Description                                                                                                         |
  | ------------------------------------------------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------- |
  | `TutorialSelectionChangedHandler`                                  | Delegate       | Signature: `(string categoryId, string tutorialId)`—fired when a tutorial leaf is selected.                         |
  | `event TutorialSelectionChangedHandler OnTutorialSelectionChanged` | Event          | Subscribe to know when the user picks a tutorial node in the TreeView.                                              |
  | `void Refresh(...)`                                                | Method         | Rebuilds the internal `TreeView` hierarchy from `_categories` & `_tutorials`.                                       |
  | `void OnTreeSelectionChanged(...)`                                 | Method         | Internal callback—pulls the selected item’s data and, if it’s a tutorial (non‐empty `tutorialId`), fires the event. |
  | `Label CreateLabelItem()`                                          | Factory Method | Creates a new `Label` to display each node’s text.                                                                  |
  | `void BindLabelItem(VisualElement, int)`                           | Binder Method  | Populates a given `Label` with that node’s `displayName` + styling (bold/color for categories vs. tutorials).       |

---

### 1.4. `TutorialEditorWindow.cs`

**Purpose**: The main Editor window (“Window → Tutorial Editor”) that lets you create/edit categories and tutorials, see them in a split‐pane TreeView (left) and Inspector (right), and save them to `Assets/Resources/Tutorials`. On Save, it also re‐writes the JSON definitions under `Resources/Tutorials/tutorial_definitions.json` and `tutorial_categories.json`.

```csharp
namespace UnityHelperSDK.Editor
{
    public class TutorialEditorWindow : EditorWindow
    {
        private const string TUTORIALS_PATH = "Assets/Resources/Tutorials";

        private Dictionary<string, TutorialCategory> _categories;
        private Dictionary<string, TutorialDefinition> _tutorials;
        private Vector2 _scrollPosition;
        private bool _isDirty;
        private string _selectedCategory;
        private string _selectedTutorial;
        private TutorialTreeView _treeView;
        private IMGUIContainer _inspectorContainer;

        [MenuItem("Window/Tutorial Editor")]
        public static void ShowWindow() { ... }

        private void OnEnable()
        {
            LoadTutorialData();
            CreateUI();
        }

        private void CreateUI()
        {
            var root = rootVisualElement;
            var splitView = new TwoPaneSplitView(0, 250, TwoPaneSplitViewOrientation.Horizontal);
            root.Add(splitView);

            // Left pane: TreeView
            _treeView = new TutorialTreeView(_categories, _tutorials);
            _treeView.OnTutorialSelectionChanged += OnTutorialSelected;
            splitView.Add(_treeView);

            // Right pane: IMGUI inspector
            _inspectorContainer = new IMGUIContainer(DrawInspector);
            splitView.Add(_inspectorContainer);

            // Toolbar (Save / Refresh / Add Category / Add Tutorial)
            var toolbar = new Toolbar();
            toolbar.Add(new ToolbarButton(() => SaveTutorialData()) { text = "Save" });
            toolbar.Add(new ToolbarButton(() => LoadTutorialData()) { text = "Refresh" });
            toolbar.Add(new ToolbarButton(() => AddNewCategory()) { text = "Add Category" });
            toolbar.Add(new ToolbarButton(() => AddNewTutorial()) { text = "Add Tutorial" });
            root.Insert(0, toolbar);
        }

        private void OnTutorialSelected(string categoryId, string tutorialId)
        {
            _selectedCategory = categoryId;
            _selectedTutorial = tutorialId;
            _inspectorContainer.MarkDirtyRepaint();
        }

        private void DrawInspector()
        {
            // If nothing selected
            if (string.IsNullOrEmpty(_selectedCategory) && string.IsNullOrEmpty(_selectedTutorial))
            {
                EditorGUILayout.HelpBox("Select a tutorial or category to edit", MessageType.Info);
                return;
            }

            _scrollPosition = EditorGUILayout.BeginScrollView(_scrollPosition);

            // Category inspector
            if (!string.IsNullOrEmpty(_selectedCategory) && _categories.TryGetValue(_selectedCategory, out var category))
                DrawCategoryInspector(category);

            // Tutorial inspector
            if (!string.IsNullOrEmpty(_selectedTutorial) && _tutorials.TryGetValue(_selectedTutorial, out var tutorial))
                DrawTutorialInspector(tutorial);

            EditorGUILayout.EndScrollView();

            // “Save Changes” button if dirty
            if (_isDirty)
            {
                EditorGUILayout.Space();
                EditorGUILayout.BeginHorizontal();
                GUILayout.FlexibleSpace();
                if (GUILayout.Button("Save Changes", GUILayout.Width(120)))
                {
                    SaveTutorialData();
                }
                EditorGUILayout.EndHorizontal();
            }
        }

        private void DrawCategoryInspector(TutorialCategory category)
        {
            EditorGUILayout.LabelField("Category Settings", EditorStyles.boldLabel);
            EditorGUI.BeginChangeCheck();
            var so = new SerializedObject(category);
            so.Update();

            // Lock ID field (not editable), expose name/description/sortOrder
            EditorGUI.BeginDisabledGroup(true);
            EditorGUILayout.PropertyField(so.FindProperty("id"));
            EditorGUI.EndDisabledGroup();
            EditorGUILayout.PropertyField(so.FindProperty("displayName"), new GUIContent("Name"));
            EditorGUILayout.PropertyField(so.FindProperty("description"));
            EditorGUILayout.PropertyField(so.FindProperty("sortOrder"));

            // List out tutorials (by ID) and “Remove” buttons
            EditorGUILayout.Space();
            EditorGUILayout.LabelField("Tutorials in Category", EditorStyles.boldLabel);
            var tutorialIdsProp = so.FindProperty("tutorialIds");
            if (tutorialIdsProp.arraySize > 0)
            {
                EditorGUI.indentLevel++;
                for (int i = 0; i < tutorialIdsProp.arraySize; i++)
                {
                    EditorGUILayout.BeginHorizontal();
                    var tutorialId = tutorialIdsProp.GetArrayElementAtIndex(i).stringValue;
                    if (_tutorials.TryGetValue(tutorialId, out var tutDef))
                        EditorGUILayout.LabelField($"{i + 1}. {tutDef.Title}");
                    else
                        EditorGUILayout.LabelField($"{i + 1}. [Missing Tutorial]");

                    if (GUILayout.Button("Remove", GUILayout.Width(60)))
                    {
                        tutorialIdsProp.DeleteArrayElementAtIndex(i);
                        _isDirty = true;
                        break;
                    }
                    EditorGUILayout.EndHorizontal();
                }
                EditorGUI.indentLevel--;
            }
            else
            {
                EditorGUILayout.HelpBox("No tutorials in this category", MessageType.Info);
            }

            EditorGUILayout.Space();
            if (GUILayout.Button("Delete Category"))
            {
                if (EditorUtility.DisplayDialog("Delete Category",
                        "Are you sure you want to delete this category? This will not delete the tutorials in it.",
                        "Delete", "Cancel"))
                {
                    var path = Path.Combine(TUTORIALS_PATH, $"{category.Id}.asset");
                    AssetDatabase.DeleteAsset(path);
                    _categories.Remove(category.Id);
                    _selectedCategory = null;
                    _selectedTutorial = null;
                    _isDirty = true;
                    _treeView.Refresh(_categories, _tutorials);
                    return;
                }
            }

            if (EditorGUI.EndChangeCheck() || so.hasModifiedProperties)
            {
                so.ApplyModifiedProperties();
                _isDirty = true;
            }
        }

        private void DrawTutorialInspector(TutorialDefinition tutorial)
        {
            EditorGUILayout.LabelField("Tutorial Settings", EditorStyles.boldLabel);
            EditorGUI.BeginChangeCheck();
            var so = new SerializedObject(tutorial);
            so.Update();

            // Lock ID
            EditorGUI.BeginDisabledGroup(true);
            EditorGUILayout.PropertyField(so.FindProperty("id"));
            EditorGUI.EndDisabledGroup();

            EditorGUILayout.PropertyField(so.FindProperty("title"));
            EditorGUILayout.PropertyField(so.FindProperty("description"));
            EditorGUILayout.PropertyField(so.FindProperty("requiredLevel"));
            EditorGUILayout.PropertyField(so.FindProperty("onlyShowOnce"));

            // Dependencies list
            EditorGUILayout.Space();
            EditorGUILayout.LabelField("Dependencies", EditorStyles.boldLabel);
            var depsProp = so.FindProperty("dependencies");
            if (depsProp.arraySize > 0)
            {
                EditorGUI.indentLevel++;
                for (int i = 0; i < depsProp.arraySize; i++)
                {
                    EditorGUILayout.BeginHorizontal();
                    var depId = depsProp.GetArrayElementAtIndex(i).stringValue;
                    if (_tutorials.TryGetValue(depId, out var depTut))
                        EditorGUILayout.LabelField($"{i + 1}. {depTut.Title}");
                    else
                        EditorGUILayout.LabelField($"{i + 1}. [Missing Tutorial]");

                    if (GUILayout.Button("Remove", GUILayout.Width(60)))
                    {
                        depsProp.DeleteArrayElementAtIndex(i);
                        _isDirty = true;
                        break;
                    }
                    EditorGUILayout.EndHorizontal();
                }
                EditorGUI.indentLevel--;
            }
            EditorGUILayout.BeginHorizontal();
            if (GUILayout.Button("Add Dependency"))
            {
                var menu = new GenericMenu();
                foreach (var kvp in _tutorials)
                {
                    if (kvp.Key != tutorial.Id && !tutorial.Dependencies.Contains(kvp.Key))
                    {
                        menu.AddItem(new GUIContent(kvp.Value.Title), false, () =>
                        {
                            depsProp.arraySize++;
                            depsProp.GetArrayElementAtIndex(depsProp.arraySize - 1).stringValue = kvp.Key;
                            so.ApplyModifiedProperties();
                            _isDirty = true;
                        });
                    }
                }
                menu.ShowAsContext();
            }
            EditorGUILayout.EndHorizontal();

            EditorGUILayout.Space();
            EditorGUILayout.PropertyField(so.FindProperty("steps"));

            EditorGUILayout.Space();
            if (GUILayout.Button("Delete Tutorial"))
            {
                if (EditorUtility.DisplayDialog("Delete Tutorial",
                        "Are you sure you want to delete this tutorial?",
                        "Delete", "Cancel"))
                {
                    // Remove from categories
                    foreach (var cat in _categories.Values)
                    {
                        if (cat.TutorialIds.Remove(tutorial.Id))
                            EditorUtility.SetDirty(cat);
                    }

                    // Remove from other tutorial dependencies
                    foreach (var otherTut in _tutorials.Values)
                    {
                        if (otherTut.Dependencies.Remove(tutorial.Id))
                            EditorUtility.SetDirty(otherTut);
                    }

                    var path = Path.Combine(TUTORIALS_PATH, $"{tutorial.Id}.asset");
                    AssetDatabase.DeleteAsset(path);
                    _tutorials.Remove(tutorial.Id);
                    _selectedTutorial = null;
                    _isDirty = true;
                    _treeView.Refresh(_categories, _tutorials);
                    return;
                }
            }

            if (EditorGUI.EndChangeCheck() || so.hasModifiedProperties)
            {
                so.ApplyModifiedProperties();
                _isDirty = true;
            }
        }

        private void LoadTutorialData()
        {
            try
            {
                _categories = new Dictionary<string, TutorialCategory>();
                _tutorials = new Dictionary<string, TutorialDefinition>();

                // Deserialize JSON from Resources/Tutorials/tutorial_definitions.json
                var tutorialJson = JsonHelper.DeserializeFromFile(Path.Combine(Application.dataPath, "Resources/Tutorials/tutorial_definitions.json"));
                var categoryJson = JsonHelper.DeserializeFromFile(Path.Combine(Application.dataPath, "Resources/Tutorials/tutorial_categories.json"));

                // Convert runtime JSON → ScriptableObjects
                if (tutorialJson != null)
                {
                    var tutorialData = JsonHelper.Deserialize<Dictionary<string, TutorialRepository.TutorialData>>(JsonHelper.Serialize(tutorialJson));
                    foreach (var kvp in tutorialData)
                    {
                        var tutDef = TutorialDefinition.FromRuntimeData(kvp.Value);
                        var path = Path.Combine(TUTORIALS_PATH, $"{tutDef.Id}.asset");
                        var existing = AssetDatabase.LoadAssetAtPath<TutorialDefinition>(path);
                        if (existing == null)
                            AssetDatabase.CreateAsset(tutDef, path);
                        else
                            EditorUtility.CopySerializedIfDifferent(tutDef, existing);

                        _tutorials[tutDef.Id] = existing ?? tutDef;
                    }
                }

                if (categoryJson != null)
                {
                    var categoryData = JsonHelper.Deserialize<Dictionary<string, TutorialRepository.TutorialCategoryData>>(JsonHelper.Serialize(categoryJson));
                    foreach (var kvp in categoryData)
                    {
                        var cat = TutorialCategory.FromRuntimeData(kvp.Value);
                        var path = Path.Combine(TUTORIALS_PATH, $"{cat.Id}.asset");
                        var existingCat = AssetDatabase.LoadAssetAtPath<TutorialCategory>(path);
                        if (existingCat == null)
                            AssetDatabase.CreateAsset(cat, path);
                        else
                            EditorUtility.CopySerializedIfDifferent(cat, existingCat);

                        _categories[cat.Id] = existingCat ?? cat;
                    }
                }

                AssetDatabase.SaveAssets();
                _isDirty = false;
                _treeView?.Refresh(_categories, _tutorials);
            }
            catch (Exception ex)
            {
                Debug.LogError($"Error loading tutorial data: {ex.Message}");
            }
        }

        private void SaveTutorialData()
        {
            if (!_isDirty) return;

            if (!Directory.Exists(TUTORIALS_PATH))
                Directory.CreateDirectory(TUTORIALS_PATH);

            // Save categories to asset files
            foreach (var cat in _categories.Values)
            {
                string assetPath = $"{TUTORIALS_PATH}/Category_{cat.Id}.asset";
                AssetDatabase.CreateAsset(cat, assetPath);
            }

            // Save tutorials to asset files
            foreach (var tut in _tutorials.Values)
            {
                string assetPath = $"{TUTORIALS_PATH}/Tutorial_{tut.Id}.asset";
                AssetDatabase.CreateAsset(tut, assetPath);
            }

            AssetDatabase.SaveAssets();
            AssetDatabase.Refresh();
            _isDirty = false;
            _treeView.Refresh(_categories, _tutorials);
            Debug.Log("Tutorial data saved successfully!");
        }

        private void AddNewCategory()
        {
            // Generate a unique ID
            var newId = "category_" + (_categories.Count + 1);
            while (_categories.ContainsKey(newId)) newId = "category_" + (_categories.Count + 2);

            var cat = CreateInstance<TutorialCategory>();
            cat.Initialize(newId, $"New Category {_categories.Count + 1}", "New category description", _categories.Count);

            var path = Path.Combine(TUTORIALS_PATH, $"{cat.Id}.asset");
            AssetDatabase.CreateAsset(cat, path);
            _categories[cat.Id] = cat;
            _isDirty = true;
            _selectedCategory = cat.Id;
            _selectedTutorial = null;
            _treeView.Refresh(_categories, _tutorials);
        }

        private void AddNewTutorial()
        {
            // Must have a selected category (or pick the first)
            var selectedCategoryId = _selectedCategory ?? _categories.Keys.FirstOrDefault();
            if (string.IsNullOrEmpty(selectedCategoryId))
            {
                EditorUtility.DisplayDialog("Error", "Please select or create a category first.", "OK");
                return;
            }

            // Generate unique tutorial ID
            var newId = "tutorial_" + (_tutorials.Count + 1);
            while (_tutorials.ContainsKey(newId)) newId = "tutorial_" + (_tutorials.Count + 2);

            var tut = CreateInstance<TutorialDefinition>();
            tut.Initialize(newId, selectedCategoryId, $"New Tutorial {_tutorials.Count + 1}", "Tutorial description");

            var path = Path.Combine(TUTORIALS_PATH, $"{tut.Id}.asset");
            AssetDatabase.CreateAsset(tut, path);
            _tutorials[tut.Id] = tut;

            // Add to category’s list
            if (_categories.TryGetValue(selectedCategoryId, out var catDef))
            {
                catDef.TutorialIds.Add(tut.Id);
                EditorUtility.SetDirty(catDef);
            }

            _isDirty = true;
            _selectedTutorial = tut.Id;
            _treeView.Refresh(_categories, _tutorials);
        }
    }
}
```

* **Key Methods/Properties**

  | Member                                                   | Type           | Description                                                                                                                                                         |
  | -------------------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `void OnEnable()`                                        | Unity Callback | Called when the window opens—loads data and builds UI.                                                                                                              |
  | `void CreateUI()`                                        | Method         | Constructs a `TwoPaneSplitView` with `TutorialTreeView` on the left and an IMGUI inspector on the right, plus a top toolbar.                                        |
  | `void DrawInspector()`                                   | Method         | Called whenever the IMGUI container repaints—draws either category or tutorial inspector, plus a “Save Changes” button if dirty.                                    |
  | `void DrawCategoryInspector(TutorialCategory)`           | Method         | Renders editable fields for the category’s name, description, sortOrder, and lists tutorials in that category.                                                      |
  | `void DrawTutorialInspector(TutorialDefinition)`         | Method         | Renders editable fields for tutorial’s title, description, requiredLevel, onlyShowOnce, dependencies, and steps array.                                              |
  | `void LoadTutorialData()`                                | Method         | Reads JSON from `Resources/Tutorials/*.json`, creates/updates `ScriptableObject` assets under `Assets/Resources/Tutorials`, populates `_categories` / `_tutorials`. |
  | `void SaveTutorialData()`                                | Method         | Writes each `TutorialCategory` and `TutorialDefinition` back to `.asset` files under `Assets/Resources/Tutorials`, then refreshes the TreeView.                     |
  | `void AddNewCategory()`                                  | Method         | Generates a new `TutorialCategory` asset with a unique ID, adds it to `_categories` and marks dirty.                                                                |
  | `void AddNewTutorial()`                                  | Method         | Requires a selected category; creates a new `TutorialDefinition` with unique ID, adds it to `_tutorials`, and appends its ID to the category’s `TutorialIds`.       |
  | `event Action<string,string> OnTutorialSelectionChanged` | Event          | Fired by `TutorialTreeView` when the user selects a tutorial leaf—used to populate the inspector.                                                                   |

---

## 2. Runtime‐Side (Assets/Scripts)

These two core classes consume the JSON definitions (from `Resources/Tutorials/*.json`), build in‐memory “sequences” of tutorial steps, and drive them via UI overlays, dialogue popups, condition checks, persistence (PlayerPrefs/Firebase), etc.

### 2.1. `TutorialHelper.cs`

**Purpose**: Orchestrates the execution of tutorials at runtime. Exposes static methods like `Initialize()`, `RegisterTutorial(...)`, `StartTutorial(...)`, and handles all step‐by‐step flow, condition checks, highlights, dialogue, progress tracking, etc.

```csharp
public static class TutorialHelper
{
    // Tracks all registered sequences in memory
    private static readonly Dictionary<string, TutorialSequence> _tutorials = new Dictionary<string, TutorialSequence>();
    private static TutorialSequence _activeTutorial;
    private static TutorialStep _currentStep;
    private static bool _isTutorialActive;

    // UI references
    private static Canvas _tutorialCanvas;
    private static GameObject _highlightPrefab;
    private static GameObject _arrowPrefab;
    private static readonly Color _highlightColor = new Color(1f, 1f, 0f, 0.3f);

    // Settings
    private static bool _canSkipTutorials = true;
    private static bool _useAnimations = true;
    private static float _defaultStepDelay = 0.5f;
    private static float _defaultAnimationDuration = 0.3f;

    // Events
    public static event Action<string> OnTutorialStarted;
    public static event Action<string> OnTutorialCompleted;
    public static event Action<TutorialStep> OnStepStarted;
    public static event Action<TutorialStep> OnStepCompleted;

    #region Initialization

    public static async Task Initialize()
    {
        // Create a full‐screen overlay canvas
        var canvasObj = new GameObject("TutorialCanvas");
        _tutorialCanvas = canvasObj.AddComponent<Canvas>();
        _tutorialCanvas.renderMode = RenderMode.ScreenSpaceOverlay;
        _tutorialCanvas.sortingOrder = 100;

        var scaler = canvasObj.AddComponent<CanvasScaler>();
        scaler.uiScaleMode = CanvasScaler.ScaleMode.ScaleWithScreenSize;
        scaler.referenceResolution = new Vector2(1920, 1080);

        // Load highlight/arrow prefabs from Resources/Tutorials/
        _highlightPrefab = Resources.Load<GameObject>("Tutorials/TutorialHighlight");
        _arrowPrefab = Resources.Load<GameObject>("Tutorials/TutorialArrow");

        // Restore any saved progress (PlayerPrefs / Firebase)
        await LoadTutorialProgress();
    }

    #endregion

    #region Tutorial Management

    public static void RegisterTutorial(TutorialSequence tutorial)
    {
        if (!_tutorials.ContainsKey(tutorial.Id))
            _tutorials[tutorial.Id] = tutorial;
    }

    public static async Task StartTutorial(string tutorialId, bool force = false)
    {
        if (_isTutorialActive && !force) return;
        if (!_tutorials.TryGetValue(tutorialId, out var tutorial))
        {
            Debug.LogWarning($"Tutorial '{tutorialId}' not found!");
            return;
        }

        // Check if we should skip (OnlyShowOnce, level, start conditions)
        if (!force && !ShouldShowTutorial(tutorial)) return;

        _isTutorialActive = true;
        _activeTutorial = tutorial;
        OnTutorialStarted?.Invoke(tutorialId);
        DebugHelper.Log($"Starting tutorial: {tutorialId}");

        await ProcessTutorialSequence(tutorial);
    }

    private static async Task ProcessTutorialSequence(TutorialSequence tutorial)
    {
        foreach (var step in tutorial.Steps)
        {
            bool stepContinues = await ProcessTutorialStep(step);
            if (!stepContinues) return; // user skipped
        }
        CompleteTutorial(tutorial);
    }

    private static async Task<bool> ProcessTutorialStep(TutorialStep step)
    {
        _currentStep = step;
        OnStepStarted?.Invoke(step);

        // If step has Conditions, skip if not met
        if (step.Conditions != null && !step.Conditions.All(c => c.IsMet()))
            return true; // continue next step

        // Create highlight/arrow visuals
        var highlightObj = await SetupStepVisuals(step);

        // If there’s a dialogue key, open dialogue
        if (!string.IsNullOrEmpty(step.DialogueKey))
        {
            await DialogueHelper.StartDialogue(new DialogueHelper.DialogueNode {
                DialogueKey = step.DialogueKey,
                Text = LocalizationManager.GetLocalizedText(step.DialogueKey)
            });
        }

        bool completed = false, skipped = false;
        while (!completed && !skipped)
        {
            // Press ESC to skip
            if (_canSkipTutorials && Input.GetKeyDown(KeyCode.Escape))
            {
                skipped = true;
                break;
            }

            // Check completion condition if provided
            if (step.CompletionCondition != null)
                completed = step.CompletionCondition.IsMet();
            else
                completed = Input.GetMouseButtonDown(0);

            await Task.Yield();
        }

        // Destroy highlight
        if (highlightObj != null) GameObject.Destroy(highlightObj);
        OnStepCompleted?.Invoke(step);
        return !skipped;
    }

    #endregion

    #region Visual Helpers

    private static async Task<GameObject> SetupStepVisuals(TutorialStep step)
    {
        if (step.Target == null) return null;

        var highlightObj = GameObject.Instantiate(_highlightPrefab);
        highlightObj.transform.SetParent(_tutorialCanvas.transform, false);

        // If UI element (RectTransform), match its position/size
        var targetRect = step.Target.GetComponent<RectTransform>();
        if (targetRect != null)
        {
            var highlightRect = highlightObj.GetComponent<RectTransform>();
            highlightRect.position = targetRect.position;
            highlightRect.sizeDelta = targetRect.sizeDelta;
        }
        else
        {
            // World‐space: use Renderer bounds
            var rend = step.Target.GetComponent<Renderer>();
            if (rend != null)
            {
                highlightObj.transform.position = rend.bounds.center;
                highlightObj.transform.localScale = rend.bounds.size;
            }
        }

        // Color = semi‐transparent yellow
        var img = highlightObj.GetComponent<UnityEngine.UI.Image>();
        if (img != null) img.color = _highlightColor;

        // If there’s an arrow prefab, instantiate and position it
        if (_arrowPrefab != null)
        {
            var arrowObj = GameObject.Instantiate(_arrowPrefab);
            arrowObj.transform.SetParent(highlightObj.transform, false);
            var screenPos = Camera.main.WorldToScreenPoint(step.Target.transform.position);
            var arrowRect = arrowObj.GetComponent<RectTransform>();
            arrowRect.anchoredPosition = screenPos;
        }

        // Pulsing animation if enabled
        if (_useAnimations)
        {
            await highlightObj.transform.DOScale(Vector3.one * 1.1f, _defaultAnimationDuration)
                .SetEase(DG.Tweening.Ease.OutBack)
                .SetLoops(-1, DG.Tweening.LoopType.Yoyo)
                .AsyncWaitForCompletion();
        }

        return highlightObj;
    }

    #endregion

    #region Progress Management

    private static Dictionary<string, TutorialProgress> _tutorialProgress = new Dictionary<string, TutorialProgress>();

    private static async Task SaveTutorialProgress(string tutorialId)
    {
        var prog = new TutorialProgress {
            CompletedAt = DateTime.UtcNow,
            StepsCompleted = _activeTutorial?.Steps?.Count ?? 0,
            CustomData = new Dictionary<string, object>()
        };
        _tutorialProgress[tutorialId] = prog;

        // Save locally
        var saveData = new TutorialSaveData { Progress = _tutorialProgress };
        var json = JsonUtility.ToJson(saveData);
        PlayerPrefs.SetString("TutorialProgress", json);
        PlayerPrefs.Save();

        // Also attempt cloud save
        try {
            await FirebaseHelper.SetDocumentAsync("tutorials", "progress", _tutorialProgress);
        }
        catch (Exception ex) {
            Debug.LogWarning($"Failed to save tutorial progress to cloud: {ex.Message}");
        }
    }

    private static async Task LoadTutorialProgress()
    {
        // First try cloud
        try {
            var cloudProg = await FirebaseHelper.GetDocumentAsync<Dictionary<string, TutorialProgress>>("tutorials", "progress");
            if (cloudProg != null) {
                _tutorialProgress = cloudProg;
                return;
            }
        }
        catch {
            // fallback
        }

        // Then local
        string json = PlayerPrefs.GetString("TutorialProgress", "");
        if (!string.IsNullOrEmpty(json)) {
            try {
                var data = JsonUtility.FromJson<TutorialSaveData>(json);
                if (data?.Progress != null)
                    _tutorialProgress = data.Progress;
            }
            catch (Exception ex) {
                Debug.LogError($"Failed to load tutorial progress: {ex.Message}");
            }
        }
    }

    public static bool IsTutorialCompleted(string tutorialId)
        => _tutorialProgress.ContainsKey(tutorialId);

    public static DateTime? GetTutorialCompletionTime(string tutorialId)
        => _tutorialProgress.TryGetValue(tutorialId, out var p) ? (DateTime?)p.CompletedAt : null;

    public static async Task ResetTutorialProgress(string tutorialId) { ... }
    public static async Task ResetAllProgress() { ... }

    #endregion

    #region Helpers

    private static void CompleteTutorial(TutorialSequence tutorial)
    {
        _isTutorialActive = false;
        _activeTutorial = null;
        _currentStep = null;

        // Save progress (asynchronously)
        _ = SaveTutorialProgress(tutorial.Id);

        OnTutorialCompleted?.Invoke(tutorial.Id);
        DebugHelper.Log($"Completed tutorial: {tutorial.Id}");
    }

    private static bool ShouldShowTutorial(TutorialSequence tutorial)
    {
        if (tutorial.OnlyShowOnce && IsTutorialCompleted(tutorial.Id))
            return false;

        // Example level check (replace with your own system)
        if (tutorial.RequiredLevel > 0) {
            int playerLevel = 1;
            if (playerLevel < tutorial.RequiredLevel) return false;
        }

        // Check custom start conditions
        return tutorial.StartConditions == null || tutorial.StartConditions.All(c => c.IsMet());
    }

    #endregion

    #region Nested Classes

    public class TutorialSequence
    {
        public string Id { get; set; }
        public List<TutorialStep> Steps { get; set; } = new List<TutorialStep>();
        public List<TutorialCondition> StartConditions { get; set; }
        public bool OnlyShowOnce { get; set; } = true;
        public int RequiredLevel { get; set; }
        public string Category { get; set; }
        public UnityEvent OnComplete { get; set; }
    }

    public class TutorialStep
    {
        public string Id { get; set; }
        public string DialogueKey { get; set; }
        public GameObject Target { get; set; }
        public List<TutorialCondition> Conditions { get; set; }
        public TutorialCondition CompletionCondition { get; set; }
        public UnityEvent OnStart { get; set; }
        public UnityEvent OnComplete { get; set; }
    }

    public abstract class TutorialCondition
    {
        public abstract bool IsMet();
        public virtual string GetDescription() { return "Tutorial condition"; }
        public virtual void Initialize() { }
        public virtual void Cleanup() { }
    }

    public delegate bool TutorialConditionCheck();

    public class DelegateCondition : TutorialCondition
    {
        private readonly TutorialConditionCheck _check;
        private readonly string _description;
        public DelegateCondition(TutorialConditionCheck check, string description = null) { ... }
        public override bool IsMet() { return _check(); }
        public override string GetDescription() { return _description ?? base.GetDescription(); }
    }

    public class CompositeCondition : TutorialCondition
    {
        private readonly TutorialCondition[] _conditions;
        private readonly bool _requireAll;
        public CompositeCondition(IEnumerable<TutorialCondition> conditions, bool requireAll = true) { ... }
        public override bool IsMet() { ... }
        public override string GetDescription() { ... }
    }

    private class TutorialProgress
    {
        public DateTime CompletedAt { get; set; }
        public int StepsCompleted { get; set; }
        public Dictionary<string, object> CustomData { get; set; }
    }

    [Serializable]
    private class TutorialSaveData
    {
        public Dictionary<string, TutorialProgress> Progress;
    }

    #endregion
}
```

* **Key Methods/Properties**

  | Member                                                                              | Type      | Description                                                                                                                                                 |
  | ----------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `static Task Initialize()`                                                          | Method    | Creates overlay Canvas, loads highlight/arrow prefabs, and loads saved progress.                                                                            |
  | `static void RegisterTutorial(TutorialSequence)`                                    | Method    | Adds a `TutorialSequence` to the internal dictionary.                                                                                                       |
  | `static Task StartTutorial(string tutorialId, bool force)`                          | Method    | Kicks off a tutorial if not already active, subject to “OnlyShowOnce”, level, and start conditions.                                                         |
  | `static Task<bool> ProcessTutorialStep(TutorialStep step)`                          | Method    | Executes a single step: applies conditions, shows highlight, opens dialogue if needed, waits for completion, then cleans up. Returns false if user skipped. |
  | `static Task<GameObject> SetupStepVisuals(TutorialStep)`                            | Method    | Instantiates highlight/arrow overlay on `step.Target` (UI or world), applies pulsing animation, returns the GameObject to be destroyed later.               |
  | `static bool IsTutorialCompleted(string tutorialId)`                                | Method    | Checks `PlayerPrefs`/cloud progress dictionary to determine if already finished.                                                                            |
  | `static async Task ResetTutorialProgress(string tutorialId)` / `ResetAllProgress()` | Method(s) | Clears local and cloud‐saved progress for one or all tutorials.                                                                                             |
  | `static bool ShouldShowTutorial(TutorialSequence)`                                  | Method    | Combines “only once” flag, requiredLevel, `StartConditions`, and progress to decide whether to run.                                                         |
  | `event Action<string> OnTutorialStarted`                                            | Event     | Fired just before a tutorial sequence begins.                                                                                                               |
  | `event Action<string> OnTutorialCompleted`                                          | Event     | Fired as soon as a tutorial finishes.                                                                                                                       |
  | `event Action<TutorialStep> OnStepStarted`                                          | Event     | Fired right before each step begins.                                                                                                                        |
  | `event Action<TutorialStep> OnStepCompleted`                                        | Event     | Fired just after each step completes (before going to the next step).                                                                                       |

---

### 2.2. `TutorialRepository.cs`

**Purpose**: A `MonoBehaviour` singleton that lives in the scene. On Awake, it loads the JSON definitions from `Resources/Tutorials/tutorial_definitions.json` and `tutorial_categories.json`, builds out in‐memory `TutorialSequence` objects, registers them with `TutorialHelper`, and listens for progress changes. It also exposes an API (`StartTutorial`, `IsTutorialCompleted`, `GetTutorialsByCategory`, etc.) for game code to trigger any tutorial by ID.

```csharp
public class TutorialRepository : MonoBehaviour
{
    #region Singleton
    private static TutorialRepository _instance;
    public static TutorialRepository Instance
    {
        get
        {
            if (_instance == null)
            {
                var go = new GameObject("[TutorialRepository]");
                _instance = go.AddComponent<TutorialRepository>();
                DontDestroyOnLoad(go);
            }
            return _instance;
        }
    }
    #endregion

    #region Data Structures

    [Serializable]
    public class TutorialData
    {
        public string Id;
        public string CategoryId;
        public bool OnlyShowOnce;
        public int RequiredLevel;
        public List<string> Dependencies;
        public List<string> StartConditions;
        public List<TutorialStepData> Steps;
    }

    [Serializable]
    public class TutorialStepData
    {
        public string Id;
        public string DialogueKey;
        public List<string> Conditions;
        public string CompletionCondition;
    }

    [Serializable]
    public class TutorialCategoryData
    {
        public string Id;
        public string Name;
        public string Description;
        public int Order;
    }

    #endregion

    private Dictionary<string, TutorialData> _tutorialDefinitions;
    private Dictionary<string, TutorialCategoryData> _categories;
    private Dictionary<string, TutorialHelper.TutorialSequence> _activeSequences;
    private HashSet<string> _completedTutorials;

    // Events
    public event Action<string> OnTutorialRegistered;
    public event Action<string> OnTutorialUnregistered;
    public event Action<string> OnTutorialCompleted;

    private async void Awake()
    {
        await InitializeAsync();
    }

    private async Task InitializeAsync()
    {
        _tutorialDefinitions = new Dictionary<string, TutorialData>();
        _categories = new Dictionary<string, TutorialCategoryData>();
        _activeSequences = new Dictionary<string, TutorialHelper.TutorialSequence>();
        _completedTutorials = new HashSet<string>();

        await LoadTutorialDefinitions();
        await LoadTutorialProgress();
    }

    private async Task LoadTutorialDefinitions()
    {
        // Load tutorial_definitions.json
        var text = await LoadTutorialJson();
        if (!string.IsNullOrEmpty(text))
        {
            var data = JsonHelper.Deserialize<Dictionary<string, TutorialData>>(text);
            if (data != null)
            {
                _tutorialDefinitions = data;
                foreach (var kvp in data)
                {
                    CreateTutorialSequence(kvp.Value);
                }
            }
        }

        // Load tutorial_categories.json
        text = await LoadCategoryJson();
        if (!string.IsNullOrEmpty(text))
        {
            var cats = JsonHelper.Deserialize<Dictionary<string, TutorialCategoryData>>(text);
            if (cats != null) _categories = cats;
        }
    }

    private Task<string> LoadTutorialJson()
    {
        var asset = Resources.Load<TextAsset>("Tutorials/tutorial_definitions");
        return Task.FromResult(asset?.text ?? "");
    }

    private Task<string> LoadCategoryJson()
    {
        var asset = Resources.Load<TextAsset>("Tutorials/tutorial_categories");
        return Task.FromResult(asset?.text ?? "");
    }

    private Task LoadTutorialProgress()
    {
        var json = PlayerPrefs.GetString("CompletedTutorials", "");
        if (!string.IsNullOrEmpty(json))
        {
            try {
                var done = JsonHelper.Deserialize<List<string>>(json);
                _completedTutorials = new HashSet<string>(done);
            }
            catch (Exception e) {
                Debug.LogError($"Error loading tutorial progress: {e.Message}");
            }
        }
        return Task.CompletedTask;
    }

    public async Task<bool> StartTutorial(string tutorialId)
    {
        if (!_tutorialDefinitions.ContainsKey(tutorialId))
        {
            Debug.LogWarning($"Tutorial '{tutorialId}' not found!");
            return false;
        }

        var tdata = _tutorialDefinitions[tutorialId];
        if (tdata.Dependencies != null)
        {
            foreach (var dep in tdata.Dependencies)
            {
                if (!_completedTutorials.Contains(dep))
                {
                    Debug.LogWarning($"Tutorial '{tutorialId}' requires '{dep}' first!");
                    return false;
                }
            }
        }

        if (!_activeSequences.ContainsKey(tutorialId))
            CreateTutorialSequence(tdata);

        await TutorialHelper.StartTutorial(tutorialId);
        return true;
    }

    protected virtual void CreateTutorialSequence(TutorialData data)
    {
        var seq = new TutorialHelper.TutorialSequence {
            Id = data.Id,
            OnlyShowOnce = data.OnlyShowOnce,
            RequiredLevel = data.RequiredLevel,
            Steps = new List<TutorialHelper.TutorialStep>()
        };

        foreach (var sd in data.Steps)
        {
            var step = new TutorialHelper.TutorialStep {
                Id = sd.Id,
                DialogueKey = sd.DialogueKey,
                Conditions = CreateConditions(sd.Conditions),
                CompletionCondition = CreateCondition(sd.CompletionCondition)
            };
            seq.Steps.Add(step);
        }

        seq.StartConditions = CreateConditions(data.StartConditions);
        _activeSequences[data.Id] = seq;
        TutorialHelper.RegisterTutorial(seq);
        OnTutorialRegistered?.Invoke(data.Id);
    }

    protected virtual List<TutorialHelper.TutorialCondition> CreateConditions(List<string> conditionIds)
    {
        if (conditionIds == null) return null;
        var list = new List<TutorialHelper.TutorialCondition>();
        foreach (var cid in conditionIds)
        {
            var cond = CreateCondition(cid);
            if (cond != null) list.Add(cond);
        }
        return list;
    }

    protected virtual TutorialHelper.TutorialCondition CreateCondition(string conditionId)
    {
        if (string.IsNullOrEmpty(conditionId)) return null;
        var parts = conditionId.Split(':');
        var type = parts[0].ToLower();

        switch (type)
        {
            case "tutorial":
                var required = parts[1].Split(',');
                return new TutorialCompletionCondition(conditionId, required);

            case "level":
                if (int.TryParse(parts[1], out int lvl))
                    return new LevelCondition(conditionId, lvl);
                break;

            case "custom":
                return new CustomTutorialCondition(parts[1]);

            default:
                return new CustomTutorialCondition(conditionId);
        }

        Debug.LogWarning($"Invalid condition format: {conditionId}");
        return null;
    }

    // Query methods

    public List<TutorialData> GetTutorialsByCategory(string categoryId)
    {
        return _tutorialDefinitions.Values.Where(t => t.CategoryId == categoryId).ToList();
    }

    public List<TutorialCategoryData> GetCategories()
    {
        return _categories.Values.ToList();
    }

    // Progress management

    public void CompleteTutorial(string tutorialId)
    {
        if (!_completedTutorials.Contains(tutorialId))
        {
            _completedTutorials.Add(tutorialId);
            SaveProgress();
            OnTutorialCompleted?.Invoke(tutorialId);
        }
    }

    public bool IsTutorialCompleted(string tutorialId)
        => _completedTutorials.Contains(tutorialId);

    private void SaveProgress()
    {
        var list = _completedTutorials.ToList();
        var json = JsonHelper.Serialize(list);
        PlayerPrefs.SetString("CompletedTutorials", json);
        PlayerPrefs.Save();
    }

    #region Nested Condition Classes

    public class CustomTutorialCondition : TutorialHelper.TutorialCondition
    {
        protected readonly string conditionId;
        private readonly Func<bool> _checkCondition;

        public CustomTutorialCondition(string conditionId, Func<bool> checkCondition = null)
        {
            this.conditionId = conditionId;
            _checkCondition = checkCondition;
        }

        public override bool IsMet()
        {
            return _checkCondition != null ? _checkCondition() : true;
        }

        public override string GetDescription()
        {
            return $"Custom condition: {conditionId}";
        }
    }

    public class TutorialCompletionCondition : CustomTutorialCondition
    {
        private readonly string[] _requiredTutorials;
        public TutorialCompletionCondition(string conditionId, params string[] requiredTutorials)
            : base(conditionId) { _requiredTutorials = requiredTutorials; }

        public override bool IsMet()
        {
            return _requiredTutorials == null ||
                   _requiredTutorials.All(tid => Instance.IsTutorialCompleted(tid));
        }

        public override string GetDescription()
        {
            return $"Requires tutorials: {string.Join(", ", _requiredTutorials)}";
        }
    }

    public class LevelCondition : CustomTutorialCondition
    {
        private readonly int _requiredLevel;
        public LevelCondition(string conditionId, int requiredLevel) : base(conditionId)
        {
            _requiredLevel = requiredLevel;
        }

        public override bool IsMet()
        {
            int playerLevel = 1; // Replace with your actual level check
            return playerLevel >= _requiredLevel;
        }

        public override string GetDescription()
        {
            return $"Requires level: {_requiredLevel}";
        }
    }

    #endregion
}
```

* **Key Methods/Properties**

  | Member                                                         | Type   | Description                                                                                                                                                         |
  | -------------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `async Task InitializeAsync()`                                 | Method | Loads JSON definitions (`tutorial_definitions.json`, `tutorial_categories.json`), constructs `TutorialSequence` objects, and loads saved progress from PlayerPrefs. |
  | `async Task LoadTutorialDefinitions()`                         | Method | Reads the JSON files from Resources, deserializes into dictionaries, and calls `CreateTutorialSequence(...)` for each tutorial.                                     |
  | `async Task LoadTutorialProgress()`                            | Method | Reads `PlayerPrefs["CompletedTutorials"]` (a JSON array of strings) into `_completedTutorials`.                                                                     |
  | `async Task<bool> StartTutorial(string tutorialId)`            | Method | Public API—checks dependencies, builds sequence if necessary, then calls `TutorialHelper.StartTutorial(...)`.                                                       |
  | `protected virtual void CreateTutorialSequence(TutorialData)`  | Method | Converts one `TutorialData` → `TutorialHelper.TutorialSequence`, including step‐by‐step conditions/CompletionCondition.                                             |
  | `protected virtual TutorialCondition CreateCondition(string)`  | Method | Factory: given a condition ID (e.g. `“tutorial:foo,bar”`, `“level:5”`, `“custom:xyz”`), returns the correct `TutorialCondition` subclass.                           |
  | `void CompleteTutorial(string tutorialId)`                     | Method | Marks as completed, updates local PlayerPrefs, and fires `OnTutorialCompleted`.                                                                                     |
  | `bool IsTutorialCompleted(string tutorialId)`                  | Method | Queries the local in‐memory `_completedTutorials` set.                                                                                                              |
  | `List<TutorialData> GetTutorialsByCategory(string categoryId)` | Method | Returns all definitions whose `CategoryId` equals the given.                                                                                                        |
  | `List<TutorialCategoryData> GetCategories()`                   | Method | Returns the list of all category definitions loaded from JSON.                                                                                                      |
  | **Nested Condition Classes**                                   | –      | Provides `CustomTutorialCondition`, `TutorialCompletionCondition`, and `LevelCondition` for runtime checks.                                                         |

---

## 3. How It All Fits Together

1. **Editor → Asset Creation**

   * You open **Window → Tutorial Editor**.
   * The `TutorialEditorWindow` calls `LoadTutorialData()`, which:

     * Reads JSON files under `Resources/Tutorials/` (if they exist).
     * Deserializes them into runtime DTOs (`TutorialData`, `TutorialCategoryData`).
     * For each entry, creates or updates a corresponding ScriptableObject in `Assets/Resources/Tutorials` (via `TutorialDefinition.FromRuntimeData(...)` / `TutorialCategory.FromRuntimeData(...)`).
     * Populates `_categories` and `_tutorials` dictionaries.
   * You navigate the TreeView on the left, select a category or tutorial, and the IMGUI inspector on the right allows you to edit fields.
   * Clicking **Save**:

     * Writes each `TutorialCategory` and `TutorialDefinition` back to `*.asset` files under `Assets/Resources/Tutorials`.
     * (You’ll still need to update the JSON files if you want them to reflect new changes at runtime—this EditorWindow does not automatically rewrite the JSON for you. You can export them manually or extend `SaveTutorialData()` to also serialize back into JSON.)

2. **Build Process → Runtime JSON**

   * Typically, you’d check the edited `.asset` files into source control, then you’d run a separate build tool (or write an Editor script) that takes all existing `TutorialDefinition` and `TutorialCategory` assets and re‐serializes them into `Resources/Tutorials/tutorial_definitions.json` and `tutorial_categories.json`. That JSON must remain in `Resources/Tutorials/` so that the next step can load them.

3. **Runtime Initialization (`TutorialRepository`)**

   * On startup, `TutorialRepository.Instance` is created. In `Awake()`, it calls `InitializeAsync()`:

     1. `LoadTutorialDefinitions()`

        * Loads `"Tutorials/tutorial_definitions.json"` and `"Tutorials/tutorial_categories.json"` as `TextAsset`s.
        * Deserializes them into `Dictionary<string, TutorialData>` and `Dictionary<string, TutorialCategoryData>`.
        * For each entry, calls `CreateTutorialSequence(...)`, which turns DTOs into `TutorialHelper.TutorialSequence` objects and registers them with `TutorialHelper.RegisterTutorial(...)`.
     2. `LoadTutorialProgress()`

        * Reads `PlayerPrefs["CompletedTutorials"]`, deserializes into a `HashSet<string>` of completed tutorial IDs.

4. **Tutorial Execution (`TutorialHelper`)**

   * When game code calls `await TutorialRepository.Instance.StartTutorial("tutorial_id")`:

     1. `TutorialRepository` ensures dependencies are met, calls `TutorialHelper.StartTutorial(tutorial_id)`.
     2. `TutorialHelper.StartTutorial(...)` spawns a new overlay Canvas (if not already created), loads highlight/arrow prefabs, checks “should show,” then calls `ProcessTutorialSequence(...)`.
     3. For each `TutorialStep` in `TutorialSequence.Steps`:

        * Checks `Conditions` (skip if not met),
        * Calls `SetupStepVisuals(...)` to instantiate highlight/arrow,
        * If there's a `DialogueKey`, calls `DialogueHelper.StartDialogue(...)` to show localized text,
        * Waits in a loop until `CompletionCondition.IsMet()` or user clicks/skips,
        * Destroys highlight object and proceeds to next step.
     4. When all steps finish, `CompleteTutorial(...)` saves progress to `PlayerPrefs` (and tries Firebase), fires `OnTutorialCompleted`, and tears down the overlay.

5. **Progress Tracking**

   * Each time a tutorial completes, `TutorialHelper` writes a JSON string of `_tutorialProgress` (a dictionary from `tutorialId → TutorialProgress`) into `PlayerPrefs["TutorialProgress"]`, and attempts to push that dictionary to Firestore via `FirebaseHelper.SetDocumentAsync("tutorials", "progress", _tutorialProgress)`.
   * At startup, `TutorialHelper` first attempts to fetch the same document from Firestore; if found, it uses cloud data; otherwise, it loads local.

---

## 4. Method & Class Reference Tables

Below is a consolidated reference of every class and its primary methods/properties, grouped by Editor vs. Runtime.

---

### 4.1. Editor‐Side Reference

#### 4.1.1. `TutorialDefinition` (ScriptableObject)

| Member                                                           | Signature                                                                                                                | Description                                                                   |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| Constructor / Instance                                           | `public class TutorialDefinition : ScriptableObject`                                                                     | Unity will call `CreateInstance<TutorialDefinition>()` to create a new asset. |
| `void Initialize(...)`                                           | `(string newId, string catId, string newTitle = "", string newDescription = "", int reqLevel = 1, bool showOnce = true)` | Populate all serialized fields on a brand‐new SO.                             |
| `public string Id { get; }`                                      | `get => id;`                                                                                                             | Unique tutorial identifier.                                                   |
| `public string CategoryId { get; }`                              | `get => categoryId;`                                                                                                     | Which category this tutorial belongs to.                                      |
| `public string Title { get; }`                                   | `get => title;`                                                                                                          | Editor‐visible name/title.                                                    |
| `public string Description { get; }`                             | `get => description;`                                                                                                    | Editor‐visible description.                                                   |
| `public bool OnlyShowOnce { get; }`                              | `get => onlyShowOnce;`                                                                                                   | If true, runtime will skip if already completed.                              |
| `public int RequiredLevel { get; }`                              | `get => requiredLevel;`                                                                                                  | Level requirement for runtime.                                                |
| `public List<string> Dependencies { get; }`                      | `get => dependencies;`                                                                                                   | IDs of tutorials that must be done first.                                     |
| `public List<string> StartConditions { get; }`                   | `get => startConditions;`                                                                                                | Custom condition IDs to check before starting.                                |
| `public List<TutorialStepData> Steps { get; }`                   | `get => steps;`                                                                                                          | List of steps (id, dialogueKey, conditions, completionCondition).             |
| `public TutorialData ToRuntimeData()`                            | Returns a new `TutorialRepository.TutorialData` object (runtime DTO) based on these fields.                              | Used by `TutorialEditorWindow` when Serializing back to JSON.                 |
| `public static TutorialDefinition FromRuntimeData(TutorialData)` | Creates a new SO whose serialized fields match `TutorialData`.                                                           | Helps “Refresh” existing assets from JSON-based runtime definitions.          |

#### 4.1.2. `TutorialCategory` (ScriptableObject)

| Member                                                                                          | Signature                                                                  | Description                                                       |
| ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `void Initialize(string newId, string newName = "", string newDescription = "", int order = 0)` | Populates all serialized fields for a new category.                        | Used when creating a brand‐new category asset.                    |
| `public string Id { get; }`                                                                     | `get => id;`                                                               | Unique category identifier (must match JSON).                     |
| `public string Name { get; }`                                                                   | `get => displayName;`                                                      | Editor‐visible name.                                              |
| `public string Description { get; }`                                                            | `get => description;`                                                      | Editor‐visible description.                                       |
| `public int SortOrder { get; }`                                                                 | `get => sortOrder;`                                                        | Controls TreeView ordering.                                       |
| `public List<string> TutorialIds { get; }`                                                      | `get => tutorialIds;`                                                      |  IDs of tutorials in this category (maintained by EditorWindow). |
| `public TutorialCategoryData ToRuntimeData()`                                                   | Returns a new `TutorialRepository.TutorialCategoryData` with these fields. | For JSON export.                                                  |
| `public static TutorialCategory FromRuntimeData(TutorialCategoryData data)`                     | Creates a new SO from that runtime DTO (to populate Editor from JSON).     | Used by `TutorialEditorWindow` on Refresh.                        |

#### 4.1.3. `TutorialTreeView` (UIElements)

| Member                                                                                                                       | Signature                                                                                                                                                                                                               | Description                                                                                    |
| ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `public TutorialTreeView(Dictionary<string, TutorialCategory> categories, Dictionary<string, TutorialDefinition> tutorials)` | Constructor: builds a `TreeView` with `makeItem` + `bindItem` lambdas, subscribes to selectionChanged, and calls `Refresh()`.                                                                                           | Creates and embeds a hierarchical list of categories and their child tutorials.                |
| `public event TutorialSelectionChangedHandler OnTutorialSelectionChanged`                                                    | Fired when user selects a **tutorial leaf** in the TreeView.                                                                                                                                                            | Signature: `void Handler(string categoryId, string tutorialId)`.                               |
| `public void Refresh(Dictionary<string, TutorialCategory> categories, Dictionary<string, TutorialDefinition> tutorials)`     | Rebuilds `_categories/_tutorials`, constructs root → category → tutorial items, and calls `_treeView.SetRootItems(...)` + `Rebuild()`.                                                                                  | Call whenever data changes so the UI re‐renders.                                               |
| `private void OnTreeSelectionChanged(IEnumerable<object> selectedIndices)`                                                   | Looks up the first selected index, retrieves its `TutorialItemData` via `_treeView.GetItemDataForIndex<TutorialItemData>(index)`, and fires `OnTutorialSelectionChanged(categoryId, tutorialId)` if `tutorialId != ""`. | Hooked as `_treeView.selectionChanged += OnTreeSelectionChanged;` to notify the editor window. |

#### 4.1.4. `TutorialEditorWindow` (EditorWindow)

| Member                                                                 | Signature                                                                                            | Description                             |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | --------------------------------------- |
| `[MenuItem("Window/Tutorial Editor")] public static void ShowWindow()` | Opens or focuses the Tutorial Editor window.                                                         | Entry point—add to Unity’s Window menu. |
| `private void OnEnable()`                                              | Unity callback when the window is created or re‐loaded. Calls `LoadTutorialData()` and `CreateUI()`. | Initializes data and builds the UI.     |
| `private void CreateUI()`                                              | Constructs a `TwoPaneSplitView` with:                                                                |                                         |

1. Left: `TutorialTreeView` (categories + tutorials)
2. Right: `IMGUIContainer(DrawInspector)`
3. Top: `Toolbar` with “Save/Refresh/Add Category/Add Tutorial” buttons | Sets up the entire Editor window layout.                                                        |
   \| `private void OnTutorialSelected(string categoryId, string tutorialId)` | Callback from the TreeView selection event—stores `_selectedCategory/_selectedTutorial`, then calls `_inspectorContainer.MarkDirtyRepaint()` so the inspector draws the appropriate fields. | Syncs TreeView selection to Inspector.                                                         |
   \| `private void DrawInspector()`                                        | IMGUI code that:

* If no selection, shows a HelpBox “Select a tutorial or category to edit.”
* If `_selectedCategory != null`, calls `DrawCategoryInspector(category)`.
* If `_selectedTutorial != null`, calls `DrawTutorialInspector(tutorial)`.
* If `_isDirty`, shows a “Save Changes” button. | Render loop for the right‐hand inspector panel.                                                 |
  \| `private void DrawCategoryInspector(TutorialCategory category)`      | Uses a `SerializedObject` to expose fields: `displayName`, `description`, `sortOrder`, and list out `tutorialIds` with “Remove” buttons. Also a “Delete Category” button. Applies changes via `serializedObject.ApplyModifiedProperties()`. | Edits a category’s properties and member tutorial references.                                     |
  \| `private void DrawTutorialInspector(TutorialDefinition tutorial)`    | Uses a `SerializedObject` to expose fields: ID (disabled), `title`, `description`, `requiredLevel`, `onlyShowOnce`, `dependencies` (with “Add/Remove”), `steps` array, plus a “Delete Tutorial” button. | Edits a tutorial’s core metadata (title/desc/etc.) and its `dependencies` and `steps`.             |
  \| `private void LoadTutorialData()`                                    | 1. Clears `_categories/_tutorials` dictionaries

2. Reads JSON via `JsonHelper.DeserializeFromFile(".../tutorial_definitions.json")` and `.tutorial_categories.json`.
3. For each runtime DTO:
   • Call `TutorialDefinition.FromRuntimeData(...)` or `TutorialCategory.FromRuntimeData(...)`
   • Load or create the corresponding `.asset` under `Assets/Resources/Tutorials` via `AssetDatabase.LoadAssetAtPath<>` / `AssetDatabase.CreateAsset(...)`, or `EditorUtility.CopySerializedIfDifferent(...)` to update
   • Add to `_tutorials` / `_categories`
4. `AssetDatabase.SaveAssets()`, clears `_isDirty`, and `_treeView.Refresh(...)` | Rehydrates ScriptableObjects from the JSON definitions at startup or refresh. |
   \| `private void SaveTutorialData()`                                    | If `_isDirty`:
5. Ensure `Assets/Resources/Tutorials` folder exists.
6. Loop `_categories.Values`: `AssetDatabase.CreateAsset(category, path)`
7. Loop `_tutorials.Values`: `AssetDatabase.CreateAsset(tutorial, path)`
8. `AssetDatabase.SaveAssets()`, `AssetDatabase.Refresh()`, clear `_isDirty`, `Debug.Log("… saved!")`, `Refresh(...)` | Writes out every `TutorialCategory` and `TutorialDefinition` SO to `.asset` files.             |
   \| `private void AddNewCategory()`                                       | 1. Generate a unique ID `"category_X"`.
9. `CreateInstance<TutorialCategory>()`, call `Initialize(newId, $"New Category N", "…", N)`.
10. `AssetDatabase.CreateAsset(cat, path)`, `_categories[newId] = cat`, \_isDirty = true, select it, refresh tree. | Creates a brand‐new category asset and adds it to the dictionary.                                |
    \| `private void AddNewTutorial()`                                       | 1. Ensure a category is selected (or pick the first).
11. Generate a unique ID `"tutorial_X"`.
12. `CreateInstance<TutorialDefinition>()`, call `Initialize(newId, selectedCategory, $"New Tutorial N", "…")`.
13. `AssetDatabase.CreateAsset(tut, path)`, `_tutorials[newId] = tut`, add `newId` to that category’s `TutorialIds`, `EditorUtility.SetDirty(cat)`, `_isDirty` = true, select it, refresh tree. | Creates a new tutorial asset, automatically appends its ID to the category.                         |

---

### 4.2. Runtime‐Side Reference

#### 4.2.1. `TutorialHelper` (Static)

| Member                                                                                                                                                | Signature                                                                                                                                                                                                                                                              | Description                                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| `public static async Task Initialize()`                                                                                                               | Creates an overlay Canvas (“TutorialCanvas”), loads highlight/arrow prefabs from `Resources/Tutorials/`, and calls `LoadTutorialProgress()`.                                                                                                                           | Must be called once at startup (e.g. in a manager or GameController). |
| `public static void RegisterTutorial(TutorialSequence tutorial)`                                                                                      | Adds `tutorial` to internal `_tutorials` dictionary if not already present.                                                                                                                                                                                            | Called by `TutorialRepository` when building sequences from JSON.     |
| `public static async Task StartTutorial(string tutorialId, bool force = false)`                                                                       | If no tutorial is active (or `force == true`), finds the `TutorialSequence` by `tutorialId`, checks `ShouldShowTutorial(...)` (OnlyShowOnce, level, start conditions), sets `_isTutorialActive`, fires `OnTutorialStarted`, then calls `ProcessTutorialSequence(...)`. | Entry point for runtime code to begin a tutorial.                     |
| `private static async Task ProcessTutorialSequence(TutorialSequence tutorial)`                                                                        | Iterates through `tutorial.Steps` one by one, calling `ProcessTutorialStep(...)`. If any step returns `false`, stops (user skipped). Otherwise, after all steps, calls `CompleteTutorial(tutorial)`.                                                                   | Drives sequential execution of each step.                             |
| `private static async Task<bool> ProcessTutorialStep(TutorialStep step)`                                                                              |  Fires `OnStepStarted(step)`.                                                                                                                                                                                                                                         |                                                                       |
|  If `step.Conditions != null` and any condition fails, returns `true` (skip to next step).                                                           |                                                                                                                                                                                                                                                                        |                                                                       |
|  Calls `SetupStepVisuals(step)` to instantiate highlight & arrow.                                                                                    |                                                                                                                                                                                                                                                                        |                                                                       |
|  If `step.DialogueKey != ""`, calls `DialogueHelper.StartDialogue(...)`.                                                                             |                                                                                                                                                                                                                                                                        |                                                                       |
|  Loops until either ESC pressed (skip) or `step.CompletionCondition.IsMet()` is true (or mouse click if no condition).                               |                                                                                                                                                                                                                                                                        |                                                                       |
|  Destroys highlight object, fires `OnStepCompleted(step)`.                                                                                           |                                                                                                                                                                                                                                                                        |                                                                       |
|  Returns `!skipped` (so caller knows if user aborted).                                                                                               | Core per‐step logic, including condition checks, UI highlight, dialogue, wait for completion, cleanup.                                                                                                                                                                 |                                                                       |
| `private static async Task<GameObject> SetupStepVisuals(TutorialStep step)`                                                                           |  If `step.Target == null`, return `null`.                                                                                                                                                                                                                             |                                                                       |
|  Instantiate `_highlightPrefab` under `_tutorialCanvas`.                                                                                             |                                                                                                                                                                                                                                                                        |                                                                       |
|  If `step.Target` has a `RectTransform`, position/size highlight to match it; otherwise, if it has a `Renderer`, place highlight around its bounds.  |                                                                                                                                                                                                                                                                        |                                                                       |
|  Set `highlight.color = _highlightColor`.                                                                                                            |                                                                                                                                                                                                                                                                        |                                                                       |
|  If `_arrowPrefab != null`, instantiate arrow, attach as child of highlight, place it at `Camera.main.WorldToScreenPoint(step.Target.position)`.     |                                                                                                                                                                                                                                                                        |                                                                       |
|  If `_useAnimations`, start a pulsing `DOScale(...).SetLoops(-1)` tween, `AsyncWaitForCompletion()` to let it run.                                   |                                                                                                                                                                                                                                                                        |                                                                       |
|  Return the new GameObject (so `ProcessTutorialStep` can destroy it later).                                                                          | Creates a semi‐transparent highlight overlay and optional arrow pointing at the `Target`.                                                                                                                                                                              |                                                                       |
| `private static bool ShouldShowTutorial(TutorialSequence tutorial)`                                                                                   | Returns false if:                                                                                                                                                                                                                                                      |                                                                       |
|  `tutorial.OnlyShowOnce == true` and `IsTutorialCompleted(tutorial.Id)`                                                                              |                                                                                                                                                                                                                                                                        |                                                                       |
|  Or player level < `tutorial.RequiredLevel`                                                                                                          |                                                                                                                                                                                                                                                                        |                                                                       |
|  Or any `tutorial.StartConditions` fails.                                                                                                            |                                                                                                                                                                                                                                                                        |                                                                       |
| Otherwise returns true.                                                                                                                               | Gates whether to run a tutorial.                                                                                                                                                                                                                                       |                                                                       |
| `private static async Task LoadTutorialProgress()`                                                                                                    | Tries to load `Dictionary<string, TutorialProgress>` from Firestore (`tutorials/progress`).                                                                                                                                                                            |                                                                       |
| If found, sets `_tutorialProgress = that dictionary`. Otherwise, reads `PlayerPrefs["TutorialProgress"]`, deserializes JSON into `_tutorialProgress`. | Restores progress dictionary so “OnlyShowOnce” logic can work.                                                                                                                                                                                                         |                                                                       |
| `private static async Task SaveTutorialProgress(string tutorialId)`                                                                                   | Builds a new `TutorialProgress` object (with `CompletedAt`, `StepsCompleted`, `CustomData`), stores in `_tutorialProgress[tutorialId]`.                                                                                                                                |                                                                       |
| Writes `TutorialSaveData { Progress = _tutorialProgress }` to JSON and stores in `PlayerPrefs`.                                                       |                                                                                                                                                                                                                                                                        |                                                                       |
| Also calls `await FirebaseHelper.SetDocumentAsync("tutorials", "progress", _tutorialProgress)` to sync to cloud.                                      | Persists progress locally and attempts to sync to Firestore.                                                                                                                                                                                                           |                                                                       |
| `public static bool IsTutorialCompleted(string tutorialId)`                                                                                           | Returns `_tutorialProgress.ContainsKey(tutorialId)`.                                                                                                                                                                                                                   | Used to check “OnlyShowOnce” or “Dependency” conditions.              |
| `public static DateTime? GetTutorialCompletionTime(string tutorialId)`                                                                                | Returns `_tutorialProgress[tutorialId].CompletedAt` if present, else null.                                                                                                                                                                                             | Helper to query when it finished.                                     |
| `public static async Task ResetTutorialProgress(string tutorialId)`                                                                                   | If `_tutorialProgress.ContainsKey(tutorialId)`, removes it, calls `SaveTutorialProgress(tutorialId)`.                                                                                                                                                                  | Clears one tutorial’s saved progress.                                 |
| `public static async Task ResetAllProgress()`                                                                                                         | Clears `_tutorialProgress`, deletes `PlayerPrefs["TutorialProgress"]`, and calls `FirebaseHelper.DeleteDocumentAsync("tutorials","progress")`.                                                                                                                         | Clears all tutorial progress both locally and (attempts) cloud.       |
| **Nested Classes**                                                                                                                                    |                                                                                                                                                                                                                                                                        |                                                                       |

* `TutorialSequence`
* `TutorialStep`
* `TutorialCondition` (abstract)
* `DelegateCondition`, `CompositeCondition`
* `TutorialProgress` + `TutorialSaveData` (internal data containers) | Data structures for building and executing sequences.                                    |

---

#### 4.2.2. `TutorialRepository` (MonoBehaviour Singleton)

| Member                                               | Signature                                                                                          | Description                                                                                               |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `public static TutorialRepository Instance { get; }` | Singleton accessor. Creates a GameObject named `[TutorialRepository]` with this component if null. | Ensures there’s exactly one repository in the scene that persists across loads.                           |
| `private async void Awake()`                         | Calls `await InitializeAsync()` on startup.                                                        | Kicks off loading definitions and progress as soon as the scene loads (Before any tutorial is triggered). |
| `private async Task InitializeAsync()`               | Instantiates `_tutorialDefinitions`, `_categories`, `_activeSequences`, `_completedTutorials`.     |                                                                                                           |

* Calls `await LoadTutorialDefinitions()`

* Calls `await LoadTutorialProgress()`
  \|                                                                                                | Sets up in‐memory caches and registers sequences with `TutorialHelper`.                                                                                    |
  \| `private async Task LoadTutorialDefinitions()`              | Loads `"Tutorials/tutorial_definitions.json"` from Resources, deserializes to `Dictionary<string,TutorialData>`.

* For each entry, calls `CreateTutorialSequence()`.

* Loads `"Tutorials/tutorial_categories.json"` similarly into `_categories`.   | Populates runtime definitions from JSON at startup.                                                                                  |
  \| `private Task<string> LoadTutorialJson()`                    | Returns `Resources.Load<TextAsset>("Tutorials/tutorial_definitions")?.text ?? ""`.                | Async‐ready wrapper for loading the definitions file.                                                                                |
  \| `private Task<string> LoadCategoryJson()`                    | Returns `Resources.Load<TextAsset>("Tutorials/tutorial_categories")?.text ?? ""`.                  | Async‐ready wrapper for loading the categories file.                                                                                 |
  \| `private Task LoadTutorialProgress()`                         | Reads `PlayerPrefs["CompletedTutorials"]` (JSON string), deserializes to `List<string>`, builds `_completedTutorials` HashSet. | Restores local progress—no cloud logic here (cloud is handled in `TutorialHelper`).                                                  |
  \| `public async Task<bool> StartTutorial(string tutorialId)`   | Checks `_tutorialDefinitions` for key; verifies any `Dependencies` are in `_completedTutorials`.

* If okay, ensures a `TutorialSequence` exists (`_activeSequences[tutorialId]`), else calls `CreateTutorialSequence(...)`.

* Calls `await TutorialHelper.StartTutorial(tutorialId)`.

* Returns `true` if started successfully.                                      | Public API for game code to trigger a tutorial by ID, respecting dependency checks.                                                   |
  \| `protected virtual void CreateTutorialSequence(TutorialData data)` | Converts `TutorialData` → `TutorialHelper.TutorialSequence`:

* Copies `Id`, `OnlyShowOnce`, `RequiredLevel`.

* Loops through each `TutorialStepData` in `data.Steps`, building new `TutorialHelper.TutorialStep` with `DialogueKey`, `Conditions`, `CompletionCondition` via `CreateCondition(...)`.

* Builds `StartConditions` similarly from `data.StartConditions`.

* Stores in `_activeSequences[data.Id] = sequence`.

* Calls `TutorialHelper.RegisterTutorial(sequence)` + `OnTutorialRegistered?.Invoke(data.Id)`. | Takes raw JSON DTO and builds the in‐memory sequence for execution.                                                                       |
  \| `protected virtual TutorialCondition CreateCondition(string conditionId)` | Parses `conditionId` (e.g. `“tutorial:foo,bar”`, `“level:5”`, `“custom:xyz”`):

* For `tutorial:…`, instantiates `TutorialCompletionCondition(conditionId, tutorialsArray)`.

* For `level:…`, instantiates `LevelCondition(conditionId, requiredLevel)`.

* Otherwise, returns `new CustomTutorialCondition(conditionId)`.          | Factory to produce the correct subclass of `TutorialCondition` based on a string‐encoded ID.                                            |
  \| `public List<TutorialData> GetTutorialsByCategory(string categoryId)` | Returns `_tutorialDefinitions.Values.Where(t => t.CategoryId == categoryId).ToList()`.            | Query all definitions that belong to a given category.                                                                                 |
  \| `public List<TutorialCategoryData> GetCategories()`           | Returns `_categories.Values.ToList()`.                                                            | Query all loaded category definitions.                                                                                                  |
  \| `public void CompleteTutorial(string tutorialId)`             | If not in `_completedTutorials`, adds it, calls `SaveProgress()`, and fires `OnTutorialCompleted?.Invoke(tutorialId)`.               | Marks a tutorial as done.                                                                                                               |
  \| `public bool IsTutorialCompleted(string tutorialId)`           | Returns `_completedTutorials.Contains(tutorialId)`.                                               | Convenience check for completion.                                                                                                        |
  \| `private void SaveProgress()`                                  | Serializes `_completedTutorials.ToList()` to JSON via `JsonHelper.Serialize(...)`, stores in `PlayerPrefs["CompletedTutorials"]`, calls `PlayerPrefs.Save()`.   | Persists the completed tutorials set locally.                                                                                             |

* **Nested Condition Classes**

  | Class                         | Extends                            | Signature                                                                                   | Description                                                                                   |
  | ----------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
  | `CustomTutorialCondition`     | `TutorialHelper.TutorialCondition` | `public CustomTutorialCondition(string conditionId, Func<bool> checkCondition = null)`      | Base “catch‐all” condition that invokes a delegate or always returns true.                    |
  | `TutorialCompletionCondition` | `CustomTutorialCondition`          | `public TutorialCompletionCondition(string conditionId, params string[] requiredTutorials)` | Checks that all `requiredTutorials` are in `TutorialRepository.Instance._completedTutorials`. |
  | `LevelCondition`              | `CustomTutorialCondition`          | `public LevelCondition(string conditionId, int requiredLevel)`                              | Checks that the player’s level (stubbed as `1` here) ≥ `requiredLevel`.                       |

---

## 5. Putting It Into Practice

1. **Creating / Editing Tutorials in the Editor**

   * Launch **Window → Tutorial Editor**.
   * Click **Add Category** → fill out “Name,” “Description,” “Sort Order.”
   * Click **Save** to write out `Category_<id>.asset`.
   * Select that category in the TreeView; click **Add Tutorial** → edit its “Title,” “Description,” “Required Level,” etc.
   * Expand the **Steps** array to add one or more `TutorialStepData` entries (each step has: `id`, `dialogueKey`, `conditions`, `completionCondition`).

     * For a simple tutorial step, you might set `dialogueKey = "tutorial_start"`, `conditions = []`, and `completionCondition = "custom:someConditionId"`.
   * Hit **Save** again.
   * In your build pipeline (or via a small Editor script), re‐export all categories and tutorials into JSON files under `Resources/Tutorials/tutorial_categories.json` and `Resources/Tutorials/tutorial_definitions.json`. (That step isn’t automated here, but you can easily write a short script that loops through all `TutorialCategory` and `TutorialDefinition` assets in `Assets/Resources/Tutorials` and uses `JsonHelper.Serialize(...)` to generate those JSON files.)

2. **Runtime Setup**

   * Somewhere very early in the game (e.g. a `GameManager`’s `Start()`), call:

     ```csharp
     await TutorialHelper.Initialize();
     var repo = TutorialRepository.Instance; // automatically loads definitions from Resources and registers sequences
     ```
   * The moment a script calls `await repo.StartTutorial("tutorial_id")`, the entire step sequence runs:

     1. Highlight/arrow overlay on specified `GameObject` target.
     2. Optional dialogue popups via `DialogueHelper`.
     3. Wait until completion condition is met (can be pressing a button, clicking a UI, or custom check).
     4. Clean up highlights, fire “OnStepCompleted,” and move to the next step.
     5. When done, persist progress locally and in Firestore if available.

3. **Dependency & OnlyOnce Logic**

   * If tutorial “combat\_1” depends on “movement\_1,” set `Dependencies = ["movement_1"]`. The EditorWindow will let you pick from a dropdown. At runtime, `TutorialRepository.StartTutorial("combat_1")` will check `_completedTutorials.Contains("movement_1")` first.
   * If `OnlyShowOnce = true`, `TutorialHelper` will skip rerunning that tutorial if it’s already in the `_tutorialProgress` dictionary.

4. **Adding Custom Conditions**

   * If you want a condition that only becomes true when the player picks up a certain item, create a `TutorialStepData` with `completionCondition = "custom:HasKey_X"`, and in `TutorialRepository.CreateCondition()`, return a `CustomTutorialCondition("custom:HasKey_X")` that wraps a `Func<bool>` to check if the player has that key. You can override `CreateCondition(...)` in a subclass of `TutorialRepository` to interpret `"custom:HasKey_X"` however you like.

5. **Localization & Dialogue**

   * Each `TutorialStep` may specify `DialogueKey = "tutorial_step_1_start"`. At runtime, `TutorialHelper` calls `DialogueHelper.StartDialogue(new DialogueNode { DialogueKey = key, Text = LocalizationManager.GetLocalizedText(key) })`, so be sure that your `LocalizationManager` has entries under `Resources/Localization/<lang>.json` for `"tutorial_step_1_start"`.

---

## 6. Summary Table

#### 6.1. Editor Classes

| Class                    | File                           | Key Responsibilities                                                                                                                                                                              |
| ------------------------ | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **TutorialDefinition**   | `TutorialDefinition.cs`        | ScriptableObject representing one tutorial. Holds metadata (id, category, title, description, dependencies, steps), and converts to/from runtime DTO (`TutorialRepository.TutorialData`).         |
| **TutorialCategory**     | `TutorialCategory.cs`          | ScriptableObject representing one category. Holds metadata (id, name, description, sortOrder, list of tutorial IDs) and converts to/from runtime DTO (`TutorialRepository.TutorialCategoryData`). |
| **TutorialStepData**     | *Nested in TutorialDefinition* | Simple data container with `id`, `dialogueKey`, `conditions` (List<string>), `completionCondition` (string). Used only in Editor context.                                                         |
| **TutorialTreeView**     | `TutorialTreeView.cs`          | Custom UIElements `VisualElement` that displays categories as bold, child tutorials as normal items. Fires an event on selection. Creates and binds Labels to `TutorialItemData`.                 |
| **TutorialEditorWindow** | `TutorialEditorWindow.cs`      | Top‐level EditorWindow (`Window → Tutorial Editor`).                                                                                                                                              |

* Loads JSON from `Resources/Tutorials/*.json`
* Creates/updates `TutorialDefinition` & `TutorialCategory` assets
* Displays a split view: TreeView (left), IMGUI inspector (right)
* Saves `.asset` files under `Assets/Resources/Tutorials` on “Save”
* Handles “Add Category” / “Add Tutorial” / “Delete” / “Refresh” logic. |

#### 6.2. Runtime Classes

| Class                          | File                        | Key Responsibilities                                                                                                   |
|------------------------------- |----------------------------|------------------------------------------------------------------------------------------------------------------------|
| **TutorialHelper**             | `TutorialHelper.cs`         | Static façade that drives tutorials at runtime:<br>• Maintains registered sequences<br>• Executes step‐by‐step (highlight, dialogue, wait for completion)<br>• Manages a `TutorialCanvas` overlay<br>• Saves/loads progress from PlayerPrefs & Firestore<br>• Exposes events for “OnTutorialStarted,” “OnStepStarted,” “OnStepCompleted,” “OnTutorialCompleted.” |
| **TutorialSequence**           | Nested in TutorialHelper     | Immutable container with `Id`, `Steps`, `StartConditions`, `OnlyShowOnce`, `RequiredLevel`, `Category`, `OnComplete`.  |
| **TutorialStep**               | Nested in TutorialHelper     | Single step: `Id`, `DialogueKey`, `Target` (GameObject), `Conditions` (list), `CompletionCondition`.                   |
| **TutorialCondition**          | Nested (abstract)           | Base class for runtime conditions.<br>• `IsMet()` (abstract)<br>• `GetDescription()`<br>• `Initialize()` / `Cleanup()` hooks. |
| **DelegateCondition**          | Nested in TutorialHelper     | Wraps a `Func<bool>` so you can plug in custom checks at runtime.                                                      |
| **CompositeCondition**         | Nested in TutorialHelper     | Combines multiple `TutorialCondition`s with AND/OR logic.                                                              |
| **TutorialProgress**           | Nested in TutorialHelper     | Data container for saving: `CompletedAt` (DateTime), `StepsCompleted` (int), `CustomData` (Dictionary<string,object>). |
| **TutorialSaveData**           | Nested in TutorialHelper     | Wrapper that holds `Dictionary<string, TutorialProgress>` so we can JSON‐serialize the whole progress map.             |
| **TutorialRepository**         | `TutorialRepository.cs`      | MonoBehaviour singleton that:<br>• Loads JSON from `Resources/Tutorials/*.json`<br>• Deserializes into `TutorialData` / `TutorialCategoryData`<br>• Builds `TutorialSequence` objects and registers with `TutorialHelper`<br>• Manages a set of completed IDs (`_completedTutorials`)<br>• Provides methods `StartTutorial(id)`, `IsTutorialCompleted(id)`, `GetTutorialsByCategory(...)`, `GetCategories()` |
| **TutorialData**               | Nested in TutorialRepository | DTO for one tutorial definition from JSON:<br>• `Id`, `CategoryId`, `OnlyShowOnce`, `RequiredLevel`, `Dependencies`, `StartConditions`, `List<TutorialStepData> Steps` |
| **TutorialStepData**           | Nested in TutorialRepository | DTO for one step in JSON:<br>• `Id`, `DialogueKey`, `Conditions` (List<string>), `CompletionCondition` (string)        |
| **TutorialCategoryData**       | Nested in TutorialRepository | DTO for one category from JSON:<br>• `Id`, `Name`, `Description`, `Order`                                              |
| **CustomTutorialCondition**    | Nested in TutorialRepository | Base for any condition keyed by a string. You can pass in a `Func<bool>`.                                              |
| **TutorialCompletionCondition**| Nested                      | Checks if a set of tutorials (by ID) are completed.                                                                    |
| **LevelCondition**             | Nested in TutorialRepository | Checks if player level ≥ required.                                                                                      |

---

## 7. Typical Usage Patterns

1. **Editor Workflow**

   * Open “Tutorial Editor” and create categories/tutorials.
   * Save assets → manually (or via a small script) export JSON under `Resources/Tutorials/`.
   * Commit the `.asset` files and JSON to version control.

2. **Runtime Setup**

   * In your game’s main `Awake`/`Start`:

     ```csharp
     await TutorialHelper.Initialize();
     // Implicitly, TutorialRepository.Instance is created in Awake() and registers everything with TutorialHelper
     ```
   * To trigger a tutorial from code, call (e.g. in a button onClick or some game event):

     ```csharp
     bool started = await TutorialRepository.Instance.StartTutorial("tutorial_id");
     if (!started)
     {
         Debug.LogWarning("Could not start tutorial (missing or unmet dependencies).");
     }
     ```
   * If you need to customize conditions, derive from `TutorialRepository`, override `CreateCondition(...)` to handle new condition‐type IDs.

3. **Condition & Dependency Logic**

   * In the **Editor**, for each `TutorialStep`, you can type arbitrary condition strings. Examples:

     * `“tutorial:movement_1,combat_1”` → means “wait until both `movement_1` and `combat_1` have completed.”
     * `“level:5”` → means “wait until the player has reached level 5.”
     * `“custom:HasSword”` → calls `CreateCondition("custom:HasSword")`, which returns a `CustomTutorialCondition` whose `IsMet()` defaults to `true` (unless you supply a delegate). To actually check “HasSword,” override `CreateCondition` in a subclass of `TutorialRepository` and do something like:

       ```csharp
       if (type == "custom" && parts[1] == "HasSword")
           return new CustomTutorialCondition(conditionId, () => PlayerInventory.Has("Sword"));
       ```
   * Dependencies on other tutorials are declared in the Editor under “Dependencies.” At runtime, before starting, `TutorialRepository.StartTutorial(...)` checks if all those IDs are in `_completedTutorials`.

4. **Progress Persistence**

   * By default, progress is saved:

     * Locally to `PlayerPrefs["TutorialProgress"]` as a JSON string of `Dictionary<string, TutorialProgress>`.
     * Cloud (Firestore) at document `tutorials → progress`.
   * On initialization, `TutorialHelper` tries Firebase first (if available). If not, falls back to local PlayerPrefs.

5. **UI/Highlight Prefabs**

   * Drop your highlight prefab at `Resources/Tutorials/TutorialHighlight.prefab` (should be a semi‐transparent `Image` or similar).
   * Drop your arrow prefab at `Resources/Tutorials/TutorialArrow.prefab`.
   * The code automatically instantiates and positions them around `step.Target` (either a UI element or a 3D GameObject).

---

## 8. Conclusion

* **Editor Side**

  * The `TutorialDefinition` and `TutorialCategory` ScriptableObjects provide a user‐friendly way to author tutorials and categories in the Unity Editor.
  * `TutorialEditorWindow` + `TutorialTreeView` constitute a custom UI for browsing, editing, adding, deleting, and saving tutorial data.
  * The recommended workflow is: author → save → export JSON → runtime.

* **Runtime Side**

  * `TutorialRepository` reads those JSON files on `Awake`, builds `TutorialSequence` objects, and registers them with `TutorialHelper`.
  * `TutorialHelper` drives the step‐by‐step tutorial logic: condition checks, highlights, dialogues, waiting for user interactions, and progress persistence.

By following this pattern, you can easily extend or modify tutorials—adding new condition types, customizing highlight visuals, integrating with your game’s specific level or inventory system, or changing how/where tutorials are stored or displayed.
