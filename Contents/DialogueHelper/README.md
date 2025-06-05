Below is a comprehensive explanation of **DialogueHelper**—a static utility class that orchestrates branching conversation flows, text‐typing animations, choice handling, event triggers, and state management. It leverages **TextMeshPro** for rich text display, `UnityEvent` for node‐specific callbacks, and optional localization. After the overview, you’ll find detailed descriptions of each method, relevant data structures, and real‐world usage examples you can drop into your Unity project.

---

## Table of Contents

1. [Overview & Purpose](#overview--purpose)
2. [Internal Data & Fields](#internal-data--fields)
3. [Dialogue Control](#dialogue-control)

   * 3.1. `StartDialogue(DialogueNode rootNode, DialogueUIConfig uiConfig = null)`
   * 3.2. `ProcessNode(DialogueNode node)`
   * 3.3. `WaitForInput()`
   * 3.4. `AnimateText(string text, TMP_Text textComponent, float delay)`
4. [State Management](#state-management)

   * 4.1. `SaveState()`
   * 4.2. `RestoreState(DialogueState state, Dictionary<string, DialogueNode> nodeMap)`
   * 4.3. `EndDialogue()`
   * 4.4. `GoBack()` & `ProcessCurrentNode()`
5. [Helper Methods](#helper-methods)

   * 5.1. `TypeText(string text)`
   * 5.2. `IsPunctuation(char c)`
6. [Data Structures & Helper Classes](#data-structures--helper-classes)

   * 6.1. `DialogueNode`
   * 6.2. `DialogueUIConfig`
   * 6.3. `DialogueState`
7. [Events](#events)

   * 7.1. `OnDialogueNodeStart`
   * 7.2. `OnDialogueNodeComplete`
   * 7.3. `OnCharacterSpeak`
8. [Real‐World Usage Examples](#real-world-usage-examples)

   * 8.1. Defining a Simple Dialogue Tree
   * 8.2. Hooking Up UI Components (`DialogueUIConfig`)
   * 8.3. Starting a Dialogue and Handling Choices
   * 8.4. Saving & Restoring Dialogue Progress (e.g., on Level Reload)
   * 8.5. Going Back to a Previous Node
   * 8.6. Subscribing to Node Enter/Exit Callbacks
9. [Best Practices & Tips](#best-practices--tips)
10. [Summary of Public API](#summary-of-public-api)

---

## 1. Overview & Purpose

```csharp
/// <summary>
/// A comprehensive dialogue system helper that handles conversations,
/// branching dialogue, and character interactions.
///
/// Features:
/// - Dialogue tree management
/// - Character-based conversations
/// - Text animation effects
/// - Branching dialogue support
/// - Event triggers
/// - Localization integration
/// - UI automation
/// - Response handling
/// </summary>
public static class DialogueHelper { … }
```

**DialogueHelper** centralizes most tasks involved in running a conversation:

* **Dialogue Tree Management**: A tree of `DialogueNode` objects, each node containing text, optional localization key, optional speaker name, and a dictionary of “choices → next nodes” for branching.
* **Character‐by‐Character Text Animation**: Gradually reveals text in a `TMP_Text` component, with extra pause on punctuation.
* **Choice Presentation**: If a node has multiple `Choices`, waits for the player’s selection via a UI helper (`UIHelper.ShowDialogueChoices`) and then continues to the selected node.
* **Localization**: If `LocalizationManager.GetLocalizedText(node.DialogueKey)` returns a string, uses that instead of `node.Text`.
* **Event Hooks**:

  * `OnDialogueNodeStart(DialogueNode)` fires when a node begins.
  * `OnDialogueNodeComplete(DialogueNode)` fires when a node finishes.
  * `OnCharacterSpeak(string characterName)` fires when a node has a `CharacterName`, so UI can update the speaker’s portrait or name label.
* **State Saving & Restoration**: Captures the current node’s ID and history stack for save/load.
* **Backtracking**: `GoBack()` pops the last visited node off the history stack and re‐processes it.
* **Skip Functionality**: Pressing **Space** during text animation immediately reveals the full text.

All asynchronous operations (`StartDialogue`, `ProcessNode`, `AnimateText`, etc.) return `Task` so you can `await` them, ensuring a natural sequential flow.

---

## 2. Internal Data & Fields

```csharp
// Dialogue settings
private static float _defaultCharacterDelay = 0.05f;
private static float _defaultPunctuationDelay = 0.2f;
private static bool _skipEnabled = true;

// Active dialogue tracking
private static DialogueNode _currentNode;
private static bool _isDialogueActive;
private static readonly Stack<DialogueNode> _dialogueHistory = new Stack<DialogueNode>();

// Event handlers
public static event Action<DialogueNode> OnDialogueNodeStart;
public static event Action<DialogueNode> OnDialogueNodeComplete;
public static event Action<string> OnCharacterSpeak;
```

1. **Animation Speeds**:

   * `_defaultCharacterDelay = 0.05f`: 50 ms per character by default.
   * `_defaultPunctuationDelay = 0.2f`: 200 ms pause if the character is `. , ! ? ;`.
   * `_skipEnabled = true`: If `true`, pressing **Space** during typing immediately displays the remaining text.

2. **State Tracking**:

   * `_currentNode`: The `DialogueNode` currently being processed.
   * `_isDialogueActive`: `true` if a dialogue sequence is underway.
   * `_dialogueHistory`: A stack storing previously visited nodes; used for “GoBack” functionality.

3. **Events**:

   * **`OnDialogueNodeStart(DialogueNode)`**: Invoked at the beginning of `ProcessNode(node)`.
   * **`OnDialogueNodeComplete(DialogueNode)`**: Invoked at the end of `ProcessNode(node)`.
   * **`OnCharacterSpeak(string characterName)`**: Invoked to notify that a speaker name should appear (before text animates).

---

## 3. Dialogue Control

### 3.1 Method: `public static async Task StartDialogue(DialogueNode rootNode, DialogueUIConfig uiConfig = null)`

```csharp
public static async Task StartDialogue(DialogueNode rootNode, DialogueUIConfig uiConfig = null)
{
    if (_isDialogueActive) return;
    
    _isDialogueActive = true;
    _currentNode = rootNode;
    _dialogueHistory.Clear();
    
    // Use UI Helper to show dialogue UI if needed
    if (uiConfig != null)
    {
        // Initialize UI components
        if (uiConfig.DialogueText != null)
        {
            _currentNode.TextComponent = uiConfig.DialogueText;
        }
        if (uiConfig.NameText != null && !string.IsNullOrEmpty(_currentNode.CharacterName))
        {
            uiConfig.NameText.text = _currentNode.CharacterName;
        }
    }
    
    await ProcessNode(rootNode);
}
```

**Purpose**:
Begin a full dialogue sequence, starting at `rootNode`. Optionally passes `uiConfig` so that the dialogue system can hook up `TMP_Text` UI elements for displaying name and dialogue text.

**Parameters**:

* `rootNode` (`DialogueNode`): The starting node of the conversation tree.
* `uiConfig` (`DialogueUIConfig`, optional): Holds references to UI components (`NameText`, `DialogueText`, a container for choice buttons, etc.). If non‐null, the method applies `rootNode.TextComponent = uiConfig.DialogueText` so subsequent nodes will know where to render text.

**Behavior**:

1. **Guard**: If `_isDialogueActive` is already `true`, immediately return (prevents re‐entrant calls).
2. **Set Flags**: `_isDialogueActive = true`; `_currentNode = rootNode`; clear `_dialogueHistory`.
3. **UI Initialization** (if `uiConfig != null`):

   * If `uiConfig.DialogueText` is assigned, store it in `rootNode.TextComponent` so that `ProcessNode` knows where to animate text.
   * If `uiConfig.NameText` is assigned and `rootNode.CharacterName` is nonempty, set the text component to `rootNode.CharacterName`.
4. **Invoke** `await ProcessNode(rootNode)`: Asynchronously handle the node and subsequent ones (recursively or via choices) until the dialogue ends.

**Example Usage**:

```csharp
public class NPCController : MonoBehaviour
{
    [SerializeField] private TMP_Text _nameLabel;
    [SerializeField] private TMP_Text _dialogueLabel;
    [SerializeField] private RectTransform _choicesContainer;
    [SerializeField] private GameObject _choiceButtonPrefab;

    private DialogueNode _rootDialogueNode;

    private void Awake()
    {
        // Build or load the dialogue tree ahead of time
        _rootDialogueNode = BuildDialogueTree();
    }

    private async void OnPlayerInteract()
    {
        var uiConfig = new DialogueHelper.DialogueUIConfig
        {
            NameText = _nameLabel,
            DialogueText = _dialogueLabel,
            ChoicesContainer = _choicesContainer,
            ChoiceButtonPrefab = _choiceButtonPrefab
        };

        // Start the conversation
        await DialogueHelper.StartDialogue(_rootDialogueNode, uiConfig);
    }

    private DialogueNode BuildDialogueTree()
    {
        // Example: create two nodes and link them
        var node1 = new DialogueHelper.DialogueNode
        {
            Id = "greeting1",
            CharacterName = "NPC",
            Text = "Hello, traveler! Care to hear about our village?",
            DialogueKey = null,
            CharacterDelay = 0.04f,
            Choices = new Dictionary<string, DialogueHelper.DialogueNode>()
        };

        var node2 = new DialogueHelper.DialogueNode
        {
            Id = "greeting2",
            CharacterName = "NPC",
            Text = "Our village was founded centuries ago... [more lore]",
            DialogueKey = null,
            CharacterDelay = 0.04f,
            Choices = null // No choices here; will wait for input to end
        };

        node1.Choices.Add("Tell me more.", node2);
        node1.Choices.Add("I’m in a hurry.", null); // null → simply end dialogue

        return node1;
    }
}
```

* When the player interacts with the NPC, `StartDialogue` is called.
* The UI components (`_nameLabel`, `_dialogueLabel`) are passed via `uiConfig`.
* Node1 displays text, then shows two buttons: “Tell me more.” or “I’m in a hurry.”

  * If “Tell me more.” is clicked, `ProcessNode(node2)` runs.
  * If “I’m in a hurry.” is clicked, `nextNode` is `null`, so the dialogue ends after completing node1.

---

### 3.2 Method: `private static async Task ProcessNode(DialogueNode node)`

```csharp
private static async Task ProcessNode(DialogueNode node)
{
    OnDialogueNodeStart?.Invoke(node);
    
    // Handle localization if available
    string text = LocalizationManager.GetLocalizedText(node.DialogueKey) ?? node.Text;
    
    // Display character name if present
    if (!string.IsNullOrEmpty(node.CharacterName))
    {
        OnCharacterSpeak?.Invoke(node.CharacterName);
    }
    
    // Animate text display
    if (node.TextComponent != null)
    {
        await AnimateText(text, node.TextComponent, node.CharacterDelay ?? _defaultCharacterDelay);
    }
    
    // Wait for input if no choices
    if (node.Choices == null || node.Choices.Count == 0)
    {
        await WaitForInput();
    }
    // Show choices if any
    else
    {
        var choice = await UIHelper.ShowDialogueChoices(node.Choices);
        _dialogueHistory.Push(node);
        
        if (node.Choices.TryGetValue(choice, out var nextNode))
        {
            await ProcessNode(nextNode);
        }
    }
    
    OnDialogueNodeComplete?.Invoke(node);
}
```

**Purpose**:
Handle a single `DialogueNode` from start to completion. This involves:

1. Invoking `OnDialogueNodeStart(node)`.
2. Resolving localized text (if `node.DialogueKey` is provided to a hypothetical `LocalizationManager`).
3. Invoking `OnCharacterSpeak(node.CharacterName)` if `CharacterName` is nonempty.
4. Animating the text in `node.TextComponent` character by character (`AnimateText`).
5. If there are **no** choices (i.e., `Choices` is null or empty), wait for the player to press **Space** or click (via `WaitForInput`).
6. Otherwise (if `Choices` exists), call `UIHelper.ShowDialogueChoices(node.Choices)`—this is expected to return a `Task<string>` that completes when the player picks an option.

   * Push the current `node` to `_dialogueHistory` so you can backtrack later.
   * Find the `nextNode` via `node.Choices[choice]`.
   * If `nextNode` is non‐null, recursively `await ProcessNode(nextNode)`. If `null`, the dialogue naturally ends.
7. Invoke `OnDialogueNodeComplete(node)` once the node is fully resolved (after input or after choice handling).

**Key Points**:

* **Recursive Flow**: If a choice leads to a next node, `ProcessNode` calls itself. This continues until you reach a node with no choices (which simply waits for input and then completes).
* **History Stack**: The stack of previously visited nodes is used by `GoBack()` if the player wants to revisit the last line.
* **UIHelper Dependency**: The method calls `UIHelper.ShowDialogueChoices(node.Choices)`—this is presumed to be a method you implement that displays each key of `node.Choices` as a button, awaits a click, and returns the chosen key string.

---

### 3.3 Method: `private static async Task WaitForInput()`

```csharp
private static async Task WaitForInput()
{
    bool inputReceived = false;
    while (!inputReceived)
    {
        if (Input.GetKeyDown(KeyCode.Space) || Input.GetMouseButtonDown(0))
        {
            inputReceived = true;
        }
        await Task.Yield();
    }
}
```

**Purpose**:
Pause execution until the player presses **Space** or left‐mouse‐click. This allows the player to read the static text at their own pace.

**Behavior**:

* `while (!inputReceived)`: loops each frame (`await Task.Yield()` yields to the next frame).
* Checks `Input.GetKeyDown(KeyCode.Space)` or `Input.GetMouseButtonDown(0)` to break the loop.
* Returns when input is detected.

---

### 3.4 Method: `private static async Task AnimateText(string text, TMP_Text textComponent, float delay)`

```csharp
private static async Task AnimateText(string text, TMP_Text textComponent, float delay)
{
    textComponent.text = "";
    
    for (int i = 0; i < text.Length; i++)
    {
        if (_skipEnabled && Input.GetKeyDown(KeyCode.Space))
        {
            textComponent.text = text;
            break;
        }
        
        textComponent.text += text[i];
        
        // Add extra delay for punctuation
        if (IsPunctuation(text[i]))
        {
            await Task.Delay(Mathf.RoundToInt(_defaultPunctuationDelay * 1000));
        }
        else
        {
            await Task.Delay(Mathf.RoundToInt(delay * 1000));
        }
    }
}
```

**Purpose**:
Reveal `text` in a `TMP_Text` component one character at a time. If the user presses **Space** during this animation, the rest of the text appears immediately. Punctuation ( `. , ! ? ;` ) causes a longer pause (`_defaultPunctuationDelay`) for dramatic effect.

**Parameters**:

* `text` (`string`): The full dialogue line to display.
* `textComponent` (`TMP_Text`): The UI component where text is typed.
* `delay` (`float`): Milliseconds to wait between non‐punctuation characters (e.g., 0.05 s).

**Behavior**:

1. **Clear** the `textComponent.text`.

2. **Loop** over each character in `text`:

   * If `_skipEnabled` is `true` **and** the player pressed **Space**, set `textComponent.text = text` (display all remaining) and `break` out of the loop.
   * Otherwise, append `text[i]` to `textComponent.text`.
   * If `text[i]` is punctuation (`IsPunctuation(text[i])`), `await Task.Delay(200 ms)`.
   * Else, `await Task.Delay(delay ms)` (e.g., 50 ms).

3. When finished (either loop completes or user skipped), return so the caller (`ProcessNode`) can proceed to waiting for input or choices.

---

## 4. State Management

### 4.1 Method: `public static DialogueState SaveState()`

```csharp
public static DialogueState SaveState()
{
    return new DialogueState
    {
        CurrentNodeId = _currentNode?.Id,
        History = new Stack<string>(_dialogueHistory.Select(n => n.Id))
    };
}
```

**Purpose**:
Capture the current dialogue state—namely, the ID of the current node and the stack of previously visited node IDs—so you can later restore exactly where the player left off.

**Returns**:
A new `DialogueState` object containing:

* `CurrentNodeId`: The `string Id` of `_currentNode`; `null` if no dialogue is active.
* `History`: A new `Stack<string>` containing the IDs of nodes in `_dialogueHistory`, preserving LIFO order.

**Example**:

```csharp
// Player triggers a save:
DialogueHelper.DialogueState saved = DialogueHelper.SaveState();
// Serialize `saved.CurrentNodeId` and `saved.History` into your save file.
```

---

### 4.2 Method: `public static void RestoreState(DialogueState state, Dictionary<string, DialogueNode> nodeMap)`

```csharp
public static void RestoreState(DialogueState state, Dictionary<string, DialogueNode> nodeMap)
{
    if (state == null || nodeMap == null) return;
    
    _dialogueHistory.Clear();
    foreach (var nodeId in state.History)
    {
        if (nodeMap.TryGetValue(nodeId, out var node))
        {
            _dialogueHistory.Push(node);
        }
    }
    
    if (nodeMap.TryGetValue(state.CurrentNodeId, out var currentNode))
    {
        _currentNode = currentNode;
    }
}
```

**Purpose**:
Resume a previously‐saved dialogue state by converting stored node IDs back into `DialogueNode` references, rebuilding `_dialogueHistory` and setting `_currentNode`.

**Parameters**:

* `state` (`DialogueState`): The object returned from `SaveState()`, containing `CurrentNodeId` and `History` of node IDs.
* `nodeMap` (`Dictionary<string, DialogueNode>`): A lookup table mapping node IDs to actual `DialogueNode` instances. You must have constructed or loaded this map beforehand (e.g., from your dialogue data).

**Behavior**:

1. **Guard**: If `state` or `nodeMap` is null, do nothing.
2. **Clear** `_dialogueHistory`.
3. **Rebuild** the history:

   * For each `nodeId` in `state.History`, look up the corresponding `DialogueNode` in `nodeMap`.
   * If found, push that `node` onto `_dialogueHistory`.
   * This reconstructs the call stack of previously visited nodes.
4. **Set `_currentNode`**:

   * Lookup `state.CurrentNodeId` in `nodeMap`. If found, assign to `_currentNode`.
   * If not found, `_currentNode` remains whatever it was or null if not found.

**Example**:

```csharp
// On load:
var loadedState = /* deserialize from save file */;
var nodeMap = BuildNodeMap(); // e.g., from JSON, scriptable objects, etc.
DialogueHelper.RestoreState(loadedState, nodeMap);
// Now `DialogueHelper` is primed to continue from `_currentNode`.
```

---

### 4.3 Method: `public static void EndDialogue()`

```csharp
public static void EndDialogue()
{
    _isDialogueActive = false;
    _currentNode = null;
    _dialogueHistory.Clear();
}
```

**Purpose**:
Terminate the current dialogue entirely—reset flags and clear history. Any subsequent calls to `StartDialogue` can begin a new conversation afresh.

**Behavior**:

* Sets `_isDialogueActive` to `false`.
* Clears `_currentNode` and `_dialogueHistory`.

Typically you call this when you reach a node that represents “goodbye” or when the player explicitly ends the conversation.

---

### 4.4 Method: `public static async Task GoBack()`

```csharp
public static async Task GoBack()
{
    if (_dialogueHistory.Count > 0)
    {
        _currentNode = _dialogueHistory.Pop();
        await ProcessCurrentNode();
    }
}
```

**Purpose**:
Return to the previous node in the history stack (if any), and re‐process that node’s text and events.

**Behavior**:

1. If `_dialogueHistory.Count > 0`:

   * Pop the last node from `_dialogueHistory` into `_currentNode`.
   * `await ProcessCurrentNode()` to replay the node’s text, fire events, etc.

If there is no previous node (the stack is empty), this method does nothing.

---

#### 4.4.1 Helper: `private static async Task ProcessCurrentNode()`

```csharp
private static async Task ProcessCurrentNode()
{
    if (_currentNode == null)
    {
        EndDialogue();
        return;
    }

    OnDialogueNodeStart?.Invoke(_currentNode);
    
    // Display text with typing effect
    if (_currentNode.Text != null)
    {
        OnCharacterSpeak?.Invoke(_currentNode.Text);
        await TypeText(_currentNode.Text);
    }

    OnDialogueNodeComplete?.Invoke(_currentNode);
}
```

**Purpose**:
Independently replays the current node’s text in a simplified manner (without handling choices or waiting for input). Useful for “GoBack” to re‐show the line.

**Behavior**:

1. If `_currentNode` is `null`, call `EndDialogue()` and return.
2. Invoke `OnDialogueNodeStart(_currentNode)`.
3. If `_currentNode.Text` is non‐null, invoke `OnCharacterSpeak(_currentNode.Text)` (though this likely should be `CharacterName`); then `await TypeText(...)` to animate text (slower version).
4. Invoke `OnDialogueNodeComplete(_currentNode)`.

**Note**:
This method duplicates some logic of `ProcessNode` but omits branching and waiting. It simply replays the text.

---

## 5. Helper Methods

### 5.1 Method: `private static async Task TypeText(string text)`

```csharp
private static async Task TypeText(string text)
{
    for (int i = 0; i < text.Length; i++)
    {
        await Task.Delay((int)(IsPunctuation(text[i]) ?
            _defaultPunctuationDelay * 1000 :
            _defaultCharacterDelay * 1000));

        if (_skipEnabled && Input.anyKeyDown)
            break;
    }
}
```

**Purpose**:
A simplified version of `AnimateText` that does not actually append characters to a `TMP_Text` component, but merely waits the appropriate delays—useful for “GoBack” where you might relay only timing, not actual UI updates.

**Behavior**:

* Loops over each character in `text` and `await` a delay.

  * If `IsPunctuation(text[i])`, wait `_defaultPunctuationDelay`.
  * Else wait `_defaultCharacterDelay`.
* If the user presses **any key** (because `Input.anyKeyDown`), break out and finish early.

---

### 5.2 Method: `private static bool IsPunctuation(char c)`

```csharp
private static bool IsPunctuation(char c)
{
    return c == '.' || c == ',' || c == '!' || c == '?' || c == ';';
}
```

**Purpose**:
Determine if a character is a punctuation mark that warrants an extra pause when typing. Used both by `AnimateText` and `TypeText`.

---

## 6. Data Structures & Helper Classes

### 6.1 Class: `DialogueNode`

```csharp
public class DialogueNode
{
    public string Id { get; set; }
    public string CharacterName { get; set; }
    public string Text { get; set; }
    public string DialogueKey { get; set; }
    public float? CharacterDelay { get; set; }
    public Dictionary<string, DialogueNode> Choices { get; set; }
    public TMP_Text TextComponent { get; set; }
    public UnityEvent OnNodeEnter { get; set; }
    public UnityEvent OnNodeExit { get; set; }
}
```

**Represents a single “line” or “node” in the dialogue tree. Fields:**

* `Id` (`string`): A unique identifier for this node (used for saving/restoring state).
* `CharacterName` (`string`): The name of the speaker (e.g., “Bob”). Used to invoke `OnCharacterSpeak` and possibly update a UI name label.
* `Text` (`string`): The fallback text if localization fails or if no `DialogueKey` is provided.
* `DialogueKey` (`string`): If non‐null, passed to `LocalizationManager.GetLocalizedText(...)` to get localized text.
* `CharacterDelay` (`float?`): If set, overrides `_defaultCharacterDelay` for this node.
* `Choices` (`Dictionary<string, DialogueNode>`): Maps each choice label (e.g., “Yes, please.”) → the next `DialogueNode`. If null or empty, there are no choices; the system simply waits for a keypress.
* `TextComponent` (`TMP_Text`): The UI component where text is displayed. This is set at runtime by `StartDialogue` using `DialogueUIConfig`.
* `OnNodeEnter` / `OnNodeExit` (`UnityEvent`): Optional events you can assign in code or inspector to trigger when the node begins or ends. This class does not automatically invoke them—you would need to hook them up manually in `ProcessNode`.

---

### 6.2 Class: `DialogueUIConfig`

```csharp
public class DialogueUIConfig
{
    public TMP_Text NameText { get; set; }
    public TMP_Text DialogueText { get; set; }
    public RectTransform ChoicesContainer { get; set; }
    public GameObject ChoiceButtonPrefab { get; set; }
}
```

**Encapsulates UI references required by the dialogue system:**

* `NameText` (`TMP_Text`): A UI text component to display the speaker’s name. Before processing each node, `StartDialogue` sets `NameText.text = node.CharacterName`.
* `DialogueText` (`TMP_Text`): The UI component where the dialogue line is typed out. Passed into `DialogueNode.TextComponent` so that `AnimateText` knows where to render.
* `ChoicesContainer` (`RectTransform`): A UI container (e.g., a vertical layout group) where the system can create choice buttons. This is used internally by `UIHelper.ShowDialogueChoices`, not by `DialogueHelper` itself.
* `ChoiceButtonPrefab` (`GameObject`): A prefab for each choice button, containing a `TMP_Text` child and a `Button` component. Also used by `UIHelper.ShowDialogueChoices`.

---

### 6.3 Class: `DialogueState`

```csharp
public class DialogueState
{
    public string CurrentNodeId { get; set; }
    public Stack<string> History { get; set; }
}
```

**Represents a snapshot of the dialogue’s progress:**

* `CurrentNodeId` (`string`): The ID of the node where the player is currently at.
* `History` (`Stack<string>`): A stack of previously visited node IDs (in LIFO order), so you can reconstruct the call history.

Use `SaveState` to create one, and `RestoreState` (with a `nodeMap`) to resume.

---

## 7. Events

### 7.1 `public static event Action<DialogueNode> OnDialogueNodeStart`

* **Fired** at the very beginning of `ProcessNode(node)`, before localization or text animation.
* **Parameter**: The `DialogueNode` that is starting.
* **Use Case**: Play a voice over, update character portrait, or fire custom logic when a node begins.

### 7.2 `public static event Action<DialogueNode> OnDialogueNodeComplete`

* **Fired** at the very end of `ProcessNode(node)`, after text animation and either input wait or choice handling.
* **Parameter**: The `DialogueNode` that has just completed.
* **Use Case**: Trigger quests, play sound effects, or advance game state when a line finishes.

### 7.3 `public static event Action<string> OnCharacterSpeak`

* **Fired** just before text animation if `node.CharacterName` is non‐empty.
* **Parameter**: The speaker’s name (`string`).
* **Use Case**: Update the `NameText` UI, switch a 2D/3D portrait to the speaking character, or play a generic “character speak” audio cue.

---

## 8. Real‐World Usage Examples

Below are several scenarios illustrating how to use **DialogueHelper** in practice.

---

### 8.1 Defining a Simple Dialogue Tree

```csharp
using System.Collections.Generic;
using TMPro;
using UnityEngine;

public class DialogueSetup : MonoBehaviour
{
    public TMP_Text nameLabel;
    public TMP_Text dialogueLabel;
    public RectTransform choicesContainer;
    public GameObject choiceButtonPrefab;

    private DialogueHelper.DialogueNode _rootNode;

    private void Start()
    {
        // Build a small conversation:
        // Node A: “Hello!”
        //   Choice: “Who are you?” → Node B
        //   Choice: “Bye.” → end
        // Node B: “I am the village elder.” → no choices (end)

        var nodeA = new DialogueHelper.DialogueNode
        {
            Id = "A",
            CharacterName = "Elder",
            Text = "Hello, friend! What do you wish to know?",
            DialogueKey = null,
            CharacterDelay = 0.04f,
            Choices = new Dictionary<string, DialogueHelper.DialogueNode>()
        };

        var nodeB = new DialogueHelper.DialogueNode
        {
            Id = "B",
            CharacterName = "Elder",
            Text = "I am the village elder. I have watched over this place for decades.",
            DialogueKey = null,
            CharacterDelay = 0.04f,
            Choices = null
        };

        nodeA.Choices.Add("Who are you?", nodeB);
        nodeA.Choices.Add("Goodbye.", null);

        _rootNode = nodeA;
    }

    public async void TriggerDialogue()
    {
        var uiConfig = new DialogueHelper.DialogueUIConfig
        {
            NameText = nameLabel,
            DialogueText = dialogueLabel,
            ChoicesContainer = choicesContainer,
            ChoiceButtonPrefab = choiceButtonPrefab
        };

        await DialogueHelper.StartDialogue(_rootNode, uiConfig);
        Debug.Log("Dialogue ended.");
    }
}
```

* **What Happens**:

  1. `TriggerDialogue()` calls `StartDialogue(_rootNode, uiConfig)`.
  2. `ProcessNode(nodeA)` animates “Hello, friend! What do you wish to know?” one character at a time.
  3. Because `Choices` has two entries, `UIHelper.ShowDialogueChoices` is invoked to display two buttons under `choicesContainer`.
  4. If the player picks “Who are you?”, `ProcessNode(nodeB)` runs, animating “I am the village elder...” and then waiting for input to continue. After pressing **Space**, `nodeB` completes, `nodeA` completes, and `StartDialogue` returns.
  5. If the player picks “Goodbye.”, `nextNode` is `null`, so `ProcessNode` simply calls `OnDialogueNodeComplete(nodeA)` and the dialogue ends.

---

### 8.2 Hooking Up UI Components (`DialogueUIConfig`)

```csharp
public class UIHelper : MonoBehaviour
{
    // A helper method implemented elsewhere in your project:
    // public static Task<string> ShowDialogueChoices(
    //     Dictionary<string, DialogueNode> choices)
    // {
    //    // Instantiate buttons inside choicesContainer, assign button texts,
    //    // and return a Task<string> that completes when a button is clicked.
    // }

    // In our DialogueSetup script above, we pass references:
    //   NameText = nameLabel (TMP_Text)
    //   DialogueText = dialogueLabel (TMP_Text)
    //   ChoicesContainer = choicesContainer (RectTransform)
    //   ChoiceButtonPrefab = choiceButtonPrefab (GameObject)

    // Example of a possible implementation (outline):
    // public static async Task<string> ShowDialogueChoices(Dictionary<string, DialogueNode> choices)
    // {
    //     var tcs = new TaskCompletionSource<string>();
    //     foreach (var kv in choices)
    //     {
    //         string choiceText = kv.Key;
    //         GameObject btnObj = Instantiate(choiceButtonPrefab, choicesContainer);
    //         btnObj.GetComponentInChildren<TMP_Text>().text = choiceText;
    //         btnObj.GetComponent<Button>().onClick.AddListener(() =>
    //         {
    //             tcs.SetResult(choiceText);
    //         });
    //     }
    //     return await tcs.Task;
    // }
}
```

* **Note**: `DialogueHelper` itself does not implement choice‐button creation. It relies on you to provide `UIHelper.ShowDialogueChoices(...)` that returns a `Task<string>` representing the chosen button’s text.

---

### 8.3 Starting a Dialogue and Handling Choices

```csharp
public class DialogueController : MonoBehaviour
{
    [SerializeField] private TMP_Text _nameText;
    [SerializeField] private TMP_Text _dialogueText;
    [SerializeField] private RectTransform _choicesContainer;
    [SerializeField] private GameObject _choiceButtonPrefab;

    private DialogueHelper.DialogueNode _dialogueRoot;

    private void Awake()
    {
        // Construct or load your dialogue tree (omitted for brevity)
        _dialogueRoot = /* build or fetch nodes */;
    }

    public async void BeginConversation()
    {
        var uiConfig = new DialogueHelper.DialogueUIConfig
        {
            NameText = _nameText,
            DialogueText = _dialogueText,
            ChoicesContainer = _choicesContainer,
            ChoiceButtonPrefab = _choiceButtonPrefab
        };

        // Subscribe to events
        DialogueHelper.OnDialogueNodeStart += node =>
        {
            if (!string.IsNullOrEmpty(node.CharacterName))
                _nameText.text = node.CharacterName;
        };
        DialogueHelper.OnCharacterSpeak += characterName =>
        {
            // e.g., play a “voice clip” or highlight character portrait
            Debug.Log($"{characterName} is speaking...");
        };

        await DialogueHelper.StartDialogue(_dialogueRoot, uiConfig);

        // Unsubscribe
        DialogueHelper.OnDialogueNodeStart -= /* callback */;
        DialogueHelper.OnCharacterSpeak -= /* callback */;
        Debug.Log("Conversation finished.");
    }
}
```

* **Flow**:

  * Before starting, subscribe to `OnDialogueNodeStart` so you can update the UI name label.
  * Call `StartDialogue`.
  * Each node’s speaker name automatically updates the UI.
  * After `StartDialogue` completes (dialogue ends), unsubscribe from events.

---

### 8.4 Saving & Restoring Dialogue Progress (e.g., on Level Reload)

```csharp
public class DialogueSaveManager : MonoBehaviour
{
    private Dictionary<string, DialogueHelper.DialogueNode> _nodeMap;

    private void Start()
    {
        // Suppose _dialogueRoot is already built
        _nodeMap = BuildNodeMap(_dialogueRoot);
    }

    public void SaveGame()
    {
        var state = DialogueHelper.SaveState();
        // Serialize state.CurrentNodeId and state.History (stack of strings) into your save file.
    }

    public void LoadGame()
    {
        var savedState = /* deserialize from save file into DialogueState */;
        DialogueHelper.RestoreState(savedState, _nodeMap);

        // Now you might call ProcessCurrentNode() if you want to replay or simply let the next StartDialogue pick up
    }

    private Dictionary<string, DialogueHelper.DialogueNode> BuildNodeMap(DialogueHelper.DialogueNode root)
    {
        var map = new Dictionary<string, DialogueHelper.DialogueNode>();
        Traverse(root, map);
        return map;

        void Traverse(DialogueHelper.DialogueNode node, Dictionary<string, DialogueHelper.DialogueNode> m)
        {
            if (!m.ContainsKey(node.Id)) m[node.Id] = node;
            if (node.Choices != null)
            {
                foreach (var kv in node.Choices)
                {
                    if (kv.Value != null)
                        Traverse(kv.Value, m);
                }
            }
        }
    }
}
```

* **Building `nodeMap`**: Recursively traverse your dialogue tree starting at `rootNode`, adding each node’s `Id → node` to the dictionary.
* **Saving**: Call `DialogueHelper.SaveState()`. Store `CurrentNodeId` and `History` stack of IDs.
* **Loading**: Call `DialogueHelper.RestoreState(savedState, nodeMap)`. Now `_dialogueHistory` and `_currentNode` are restored.

---

### 8.5 Going Back to a Previous Node

```csharp
public class DialogueBackButton : MonoBehaviour
{
    [SerializeField] private Button _backButton;

    private void Start()
    {
        _backButton.onClick.AddListener(async () => await DialogueHelper.GoBack());
    }
}
```

* Pressing the “Back” button triggers `DialogueHelper.GoBack()`:

  * Pops the last node off the stack, sets `_currentNode`, and calls `ProcessCurrentNode()`, which replays the line’s text.

---

### 8.6 Subscribing to Node Enter/Exit Callbacks

```csharp
public class QuestIntegration : MonoBehaviour
{
    private void OnEnable()
    {
        DialogueHelper.OnDialogueNodeComplete += HandleNodeComplete;
    }

    private void OnDisable()
    {
        DialogueHelper.OnDialogueNodeComplete -= HandleNodeComplete;
    }

    private void HandleNodeComplete(DialogueHelper.DialogueNode node)
    {
        if (node.Id == "quest_intro_node")
        {
            // Trigger quest acceptance
            QuestManager.AcceptQuest("FindTheHerb");
        }
    }
}
```

* When a node with `Id = "quest_intro_node"` finishes, `HandleNodeComplete` fires, letting you start a quest or trigger some gameplay effect.

---

## 9. Best Practices & Tips

1. **Always Provide Unique `Id` for Each Node**

   * Node IDs are critical for saving/restoring state. Ensure they are globally unique in your dialogue tree.

2. **Use `DialogueKey` for Localization**

   * If you support multiple languages, set `DialogueKey` to a unique key (e.g., `"npc_greeting_01"`) and store translations in a `LocalizationManager`.
   * If `LocalizationManager.GetLocalizedText(key)` returns non‐null, that localized string is used; otherwise, fallback to `node.Text`.

3. **Keep `Choices` Dictionary Keys Intuitive**

   * The dictionary key (e.g., `"Yes, tell me more."`) is what the player sees on the choice button. Avoid overly long keys—use concise text.

4. **Clear UI After Dialogue Ends**

   * `StartDialogue` does not automatically hide or clear UI elements (e.g., choice buttons). Ensure your `UIHelper.ShowDialogueChoices` cleans up the buttons after a choice is made or the dialogue ends.

5. **Subscription/Unsubscription of Events**

   * When you subscribe to `OnDialogueNodeStart` or `OnCharacterSpeak`, always unsubscribe in `OnDisable` or after dialogue completes to avoid memory leaks.

6. **Skip Animation Early**

   * During `AnimateText`, pressing **Space** finalizes the line instantly. You may want to also let **Mouse Click** skip—modify `if (_skipEnabled && Input.GetKeyDown(KeyCode.Space))` to check `Input.anyKeyDown` or a custom skip key.

7. **Handle Null `nextNode` (Ending Conversation)**

   * In `ProcessNode`, if the player chooses an option that maps to `null`, the `if (nextNode != null)` check fails, so the method simply returns and eventually unwinds to `StartDialogue`, completing the dialogue.

8. **UIHelper Dependencies**

   * Implement `UIHelper.ShowDialogueChoices(...)` so it returns a `Task<string>` completing when the player picks a choice. This is outside the scope of `DialogueHelper` but critical for choices.

9. **Avoid Long Pauses**

   * If you want to disable the punctuation pause, set `_defaultPunctuationDelay = 0f` or modify `IsPunctuation` logic.

10. **DialogueNode Event Hooks**

    * Although the class defines `UnityEvent OnNodeEnter` and `OnNodeExit` in `DialogueNode`, it does not currently invoke them. You can extend `ProcessNode` to call `node.OnNodeEnter.Invoke()` at the start and `node.OnNodeExit.Invoke()` at the end if desired.

---

## 10. Summary of Public API

| Method Signature                                                                                  | Description                                                                                                                                                                                 |
| ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `public static async Task StartDialogue(DialogueNode rootNode, DialogueUIConfig uiConfig = null)` | Begin a conversation at `rootNode`, hook up UI components if `uiConfig` is provided, and process the entire dialogue tree asynchronously.                                                   |
| `public static DialogueState SaveState()`                                                         | Capture the current dialogue progress (`CurrentNodeId` and history of visited node IDs). Useful for saving game state.                                                                      |
| `public static void RestoreState(DialogueState state, Dictionary<string, DialogueNode> nodeMap)`  | Restore previously saved dialogue progress by looking up node IDs in `nodeMap` and rebuilding `_dialogueHistory` and `_currentNode`.                                                        |
| `public static void EndDialogue()`                                                                | Terminate the current dialogue, clear state flags and history.                                                                                                                              |
| `public static async Task GoBack()`                                                               | Return to the previous node (if any) by popping from `_dialogueHistory` and reprocessing that node’s text.                                                                                  |
| `public static event Action<DialogueNode> OnDialogueNodeStart`                                    | Event invoked when a `DialogueNode` begins processing—listeners can update UI or play sounds.                                                                                               |
| `public static event Action<DialogueNode> OnDialogueNodeComplete`                                 | Event invoked when a `DialogueNode` finishes processing—listeners can trigger quests, animations, etc.                                                                                      |
| `public static event Action<string> OnCharacterSpeak`                                             | Event invoked before each node’s text animates if `CharacterName` is non‐empty—listeners can update the speaker’s portrait or name label.                                                   |
| **Data Structures**                                                                               |                                                                                                                                                                                             |
| `public class DialogueNode`                                                                       | Represents a single node in the conversation: `Id`, `CharacterName`, `Text`, optional `DialogueKey` for localization, optional `CharacterDelay`, `Choices` (dictionary), and UI references. |
| `public class DialogueUIConfig`                                                                   | Holds UI references (`NameText`, `DialogueText`, `ChoicesContainer`, `ChoiceButtonPrefab`) used during `StartDialogue` to connect UI elements to nodes.                                     |
| `public class DialogueState`                                                                      | Simple struct containing `CurrentNodeId` and a `Stack<string> History` for saving/restoring dialogue progress.                                                                              |

**NOTE**: Internally, `ProcessNode`, `WaitForInput`, `AnimateText`, `TypeText`, and `IsPunctuation` handle the logic of text display, input handling, and branching. You generally only need to call `StartDialogue`, and optionally `SaveState` / `RestoreState` / `GoBack` as needed.

With these tools and examples, you have everything needed to:

1. Define branching dialogue trees in code or data.
2. Connect them to UI screens that display speaker names and dialogue text.
3. Animate text for dramatic effect.
4. Present multiple‐choice decisions to the player.
5. Track dialogue progress, save/load mid‐conversation, or allow backtracking.
6. Hook into events to trigger game logic at key moments in the conversation.

Enjoy building your interactive narratives!
