Here’s a comprehensive guide to the UnityHelperSDK tutorial system, covering its architecture, data formats, core classes, and how to define and run your own tutorials.

---

## Overview

The UnityHelperSDK tutorial system is a **data‑driven**, **event‑based** engine for orchestrating multi‑step tutorials with branching, conditional logic, persistence, and analytics. You:

1. **Define** tutorials (and their categories) in JSON/ScriptableObjects.
2. **Load** definitions at runtime via `TutorialRepository`.
3. **Register** sequences with `TutorialHelper`.
4. **Start** tutorials by ID, letting the engine step through each `TutorialStep` as conditions are met.
5. **Track** progress in `PlayerPrefs` (so each tutorial can “only show once” if desired).
6. **Emit** events (`TutorialEvents`) for UI, logic hooks, and analytics.

All communication is decoupled via an `EventHelper` pub/sub layer.

---

## 1. Data Structures

### 1.1. TutorialConditionData

```csharp
[Serializable]
public class TutorialConditionData
{
    public string            EventId;        // identifier for the condition callback
    public TutorialConditionType ConditionType;// Start, Step, or Custom
    public string[]          Parameters;     // optional args (thresholds, keys, etc.)
}
```

### 1.2. TutorialConditionType

```csharp
public enum TutorialConditionType
{
    Start,   // gate tutorial start
    Step,    // gate progression within a step
    Custom   // user‑defined logic
}
```

### 1.3. TutorialData & Related

Defined inside `TutorialRepository`:

```csharp
[Serializable] public class TutorialData {
  public string                     Id;
  public string                     CategoryId;
  public string                     Title, Description;
  public bool                       OnlyShowOnce;
  public int                        RequiredLevel;
  public List<string>               Dependencies;
  public List<TutorialConditionData> StartConditions;
  public List<TutorialStepData>     Steps;
}

[Serializable] public class TutorialStepData {
  public string                     Id, DialogueKey;
  public GameObject                 TargetObject;
  public List<TutorialConditionData> Conditions;
  public TutorialConditionData      CompletionCondition;
}

[Serializable] public class TutorialCategoryData {
  public string Id, Name, Description;
  public int    Order;
}
```

---

## 2. Event Types (`TutorialEvents`)

All tutorial logic and UI hook into these structs via `EventHelper.Subscribe` / `Trigger`:

| Event                            | Data                                                             |
| -------------------------------- | ---------------------------------------------------------------- |
| **TutorialStartConditionEvent**  | TutorialId, PlayerLevel → sets `HasMetConditions`                |
| **TutorialStepConditionEvent**   | TutorialId, StepId, TargetObject → sets `HasMetConditions`       |
| **TutorialStartedEvent**         | TutorialId, CategoryId, RequiredLevel                            |
| **TutorialCompletedEvent**       | TutorialId, CategoryId, TimeSpent, WasSkipped                    |
| **TutorialStepStartedEvent**     | TutorialId, StepId, DialogueKey, TargetObject                    |
| **TutorialStepCompletedEvent**   | TutorialId, StepId, TimeSpent, WasSkipped                        |
| **CustomTutorialConditionEvent** | ConditionId, Parameters → sets `HasMetCondition`                 |
| **TutorialAnalyticsEvent**       | TutorialId, EventType, StepId, Duration, Success, AdditionalData |

Use these to update your UI (highlights, arrows, dialogue pop‑ups) or tie into game logic/telemetry.

---

## 3. Core Runtime

### 3.1. TutorialHelper

A static engine that:

* Holds all registered `TutorialSequence` instances.
* Manages the active tutorial, current step index, and looping through steps.
* Exposes:

  ```csharp
  Task Initialize();                    // subscribe to core events
  void    RegisterTutorial(sequence);  // called by Repository
  Task    StartTutorial(tutorialId);   // kicks off the flow
  ```
* Internally:

  * Checks start conditions: `sequence.CheckStartConditions()`
  * Recursively calls `ProcessNextStep()`, waiting for each step’s conditions and completion.
  * Fires analytics and start/complete events.

### 3.2. TutorialSequence

Encapsulates one tutorial flow:

* **Meta**: `Id`, `CategoryId`, `OnlyShowOnce`, `RequiredLevel`.
* **Start conditions**: list of callbacks invoked on a `TutorialStartConditionEvent`.
* **Steps**: `List<TutorialStep>`.
* **Lifecycle**:

  * `CheckStartConditions()`
  * `Start()`: timestamp + analytics + `Trigger(TutorialStartedEvent)`
  * `Complete()`: analytics + `Trigger(TutorialCompletedEvent)` + notify `TutorialRepository`.

### 3.3. TutorialStep

Represents a single step within a sequence:

* **Properties**: `Id`, `DialogueKey`, `Target` GameObject.
* **Condition callbacks** (list) → invoked against `TutorialStepConditionEvent`.
* **Completion callback** → invoked similarly; when met, fires `TutorialStepCompletedEvent`.
* **Cleanup**: unsubscribes all handlers once the step is done.

---

## 4. Data Loading & Persistence

All definitions live in JSON under `Resources/Tutorials/`:

* **tutorial\_definitions.json** → `Dictionary<string, TutorialData>`
* **tutorial\_categories.json** → `Dictionary<string, TutorialCategoryData>`

`TutorialRepository` on **Awake**:

1. **Load definitions** (`JsonHelper.Deserialize` from TextAssets).
2. **Create sequences** via `CreateTutorialSequence(...)`.
3. **Load progress** (`PlayerPrefs["CompletedTutorials"]`) into a `HashSet<string>`.

When a tutorial completes, `CompleteTutorial(id)`:

* Adds to `CompletedTutorials`, `SaveProgress()`, triggers analytics, and fires `OnTutorialCompleted` event.

---

## 5. Using the System

### 5.1. Setup

1. **Add** a `TutorialRepository` to your startup scene (or rely on its singleton‑creation).
2. **Ensure** `TutorialHelper.Initialize()` is called on game launch (e.g., in an async awakening manager).

### 5.2. Defining a Tutorial (JSON Example)

```jsonc
{
  "move-character": {
    "Id":           "move-character",
    "CategoryId":   "movement",
    "Title":        "Moving Around",
    "Description":  "Learn how to move your character.",
    "OnlyShowOnce": true,
    "RequiredLevel":1,
    "Dependencies": [],
    "StartConditions":[
      { "EventId":"game-loaded", "ConditionType":"Start", "Parameters": [] }
    ],
    "Steps":[
      {
        "Id":"step-1",
        "DialogueKey":"movement_intro",
        "TargetObject":"PlayerCharacter",
        "Conditions":[
          { "EventId":"input-detected", "ConditionType":"Step", "Parameters":["Horizontal"] }
        ],
        "CompletionCondition":
          { "EventId":"distance-moved", "ConditionType":"Custom", "Parameters":["5"] }
      }
    ]
  }
}
```

> **Tip**: For `TargetObject`, you can store the name or lookup via `GameObject.Find`.

### 5.3. Starting a Tutorial

From your game logic:

```csharp
// Ensure repo is initialized
await TutorialRepository.Instance.StartTutorial("move-character");
```

* Checks category, required level, dependencies, and “only show once” logic.
* Triggers `TutorialStartedEvent` → begins the engine loop.

### 5.4. Custom Conditions

To define bespoke logic (e.g. “player health > 50%”):

1. **Subscribe** your own handler to `TutorialEvents.CustomTutorialConditionEvent`.
2. **Check** `evt.ConditionId` and `evt.Parameters`; set `evt.HasMetCondition` accordingly.

```csharp
EventHelper.Subscribe<TutorialEvents.CustomTutorialConditionEvent>(evt => {
  if (evt.ConditionId == "health-check") {
    float threshold = float.Parse(evt.Parameters[0].ToString());
    evt.HasMetCondition = Player.Instance.Health >= threshold;
  }
});
```

Then in JSON use `"ConditionType":"Custom"` with `"EventId":"health-check"`.

---

## 6. Listening & UI Integration

Hook UI elements into events:

```csharp
EventHelper.Subscribe<TutorialEvents.TutorialStepStartedEvent>(evt => {
  // Display dialogue for evt.DialogueKey
  DialogueUI.Show(evt.DialogueKey);
  // Highlight evt.TargetObject with highlightPrefab
  HighlightManager.Highlight(evt.TargetObject, highlightColor);
});
```

Similarly, on `TutorialStepCompletedEvent`, remove highlights or show “Next” buttons.

---

## 7. API Quick Reference

| Class                  | Key Methods/Props                                                                                     |
| ---------------------- | ----------------------------------------------------------------------------------------------------- |
| **TutorialHelper**     | `Initialize()`, `RegisterTutorial()`, `StartTutorial(id)`                                             |
| **TutorialRepository** | `StartTutorial(id)`, `CompleteTutorial(id)`, `IsTutorialCompleted(id)`, `GetTutorialsByCategory(cat)` |
| **TutorialSequence**   | `CheckStartConditions()`, `Start()`, `Complete()`                                                     |
| **TutorialStep**       | `AddCondition()`, `SetCompletionCondition()`, `CheckConditions()`, `CheckCompletionCondition()`       |
| **EventHelper**        | `Subscribe<T>(handler)`, `Unsubscribe(handler)`, `Trigger(evt)`                                       |

---

### That’s it!

With these building blocks, you can author rich, condition‑driven tutorials entirely via data definitions, plug in custom logic, and keep your game code decoupled from tutorial flow.

---

OK, Let's walk through **practical examples** of how to **integrate the `EventHelper` and `IChainedEvent` system into a Unity tutorial system**. These examples will reflect **real tutorial steps** such as showing messages, waiting for input, UI highlighting, and triggering gameplay actions.

---

## 🧩 Quick Setup Recap

Assume:

* You have a `TutorialManager` MonoBehaviour that runs tutorials.
* You're using `UIEvent`, `DialogueEvent`, `WaitForKeyEvent`, `HighlightButtonEvent`, etc. as we’ve defined before.
* You call `EventHelper.ProcessEvents()` in a `MonoBehaviour.Update()` method somewhere in your game (like `TutorialManager` or a central GameManager).

---

## 🧪 **Example 1: Simple Step-by-Step Tutorial**

### Step Flow:

1. Show dialogue "Welcome!"
2. Wait for player to press **Enter**
3. Highlight the **Jump** button
4. Wait until the player jumps
5. Show "Good job!"

```csharp
public void StartSimpleTutorial()
{
    var welcome = new DialogueEvent("Welcome!", dialogueUI, this);
    var waitEnter = new WaitForKeyEvent(KeyCode.Return, this);
    var highlightJump = new HighlightButtonEvent(jumpButton, this);
    var waitJump = new WaitUntilEvent(() => player.HasJumped, this);
    var done = new DialogueEvent("Good job!", dialogueUI, this);

    // Chain steps
    welcome.NextEvent = waitEnter;
    waitEnter.NextEvent = highlightJump;
    highlightJump.NextEvent = waitJump;
    waitJump.NextEvent = done;

    EventHelper.TriggerChainedEvent(welcome);
}
```

---

## 🔀 **Example 2: Branching Based on Player Choice**

Use `WaitUntilEvent` with conditional logic to branch tutorial paths.

```csharp
public void StartChoiceTutorial()
{
    var askChoice = new DialogueEvent("Choose your path: [A]ttack or [D]efend", dialogueUI, this);
    var waitInput = new WaitUntilEvent(() => Input.GetKeyDown(KeyCode.A) || Input.GetKeyDown(KeyCode.D), this);

    var attackStep = new DialogueEvent("You chose to attack!", dialogueUI, this);
    var defendStep = new DialogueEvent("You chose to defend!", dialogueUI, this);

    askChoice.NextEvent = waitInput;

    waitInput.NextEvent = Input.GetKeyDown(KeyCode.A) ? attackStep : defendStep;

    EventHelper.TriggerChainedEvent(askChoice);
}
```

> ✅ **Note:** If you want dynamic branching, you can override `.ProcessEvent()` in a custom event to pick `NextEvent` manually based on conditions at runtime.

---

## 📜 **Example 3: Modular Tutorials Using ScriptableObjects or Data**

Define tutorial steps as reusable assets, each step corresponding to a serialized `TutorialStep`:

```csharp
[CreateAssetMenu(menuName = "Tutorial/Step")]
public class TutorialStep : ScriptableObject
{
    public string message;
    public KeyCode keyToPress;
}
```

Then dynamically build the event chain:

```csharp
public void RunSteps(TutorialStep[] steps)
{
    IChainedEvent previous = null;

    foreach (var step in steps)
    {
        var dialogue = new DialogueEvent(step.message, dialogueUI, this);
        var wait = new WaitForKeyEvent(step.keyToPress, this);

        dialogue.NextEvent = wait;

        if (previous == null)
            EventHelper.TriggerChainedEvent(dialogue);
        else
            previous.NextEvent = dialogue;

        previous = wait;
    }
}
```

---

## 🎯 **Example 4: Triggering Events During Gameplay**

Imagine your tutorial requires the player to reach a specific area:

```csharp
public class WaitForZoneEnterEvent : UIEvent
{
    private readonly Transform _player;
    private readonly Collider _zone;

    public WaitForZoneEnterEvent(Transform player, Collider zone, MonoBehaviour runner = null)
        : base(runner)
    {
        _player = player;
        _zone = zone;
    }

    public override IEnumerator ProcessEvent()
    {
        while (!_zone.bounds.Contains(_player.position))
            yield return null;
        Complete();
    }
}
```

### Usage:

```csharp
var reachZone = new WaitForZoneEnterEvent(player.transform, tutorialZone, this);
var message = new DialogueEvent("You reached the safe zone!", dialogueUI, this);

reachZone.NextEvent = message;

EventHelper.TriggerChainedEvent(reachZone);
```

---

## 🧠 **Advanced: TutorialManager with EventHandlerComponent**

To manage auto-unsubscribing listeners, use `EventHandlerComponent`:

```csharp
public class TutorialManager : MonoBehaviour, IEventHandler
{
    private EventHandlerComponent _eventHandler;

    private void Awake()
    {
        _eventHandler = gameObject.AddComponent<EventHandlerComponent>();
    }

    private void OnEnable()
    {
        _eventHandler.AddHandler<TutorialStepCompleted>(OnTutorialStepCompleted);
    }

    private void OnTutorialStepCompleted(TutorialStepCompleted step)
    {
        Debug.Log($"Step complete: {step.name}");
    }

    public void OnEventCleanup()
    {
        Debug.Log("TutorialManager cleaned up");
    }
}
```

---

## Summary: What You Can Build With This

* ✅ Text-based guided onboarding
* ✅ Interactive tutorials with button highlights
* ✅ Conditional logic (choices, branching)
* ✅ Visual effects (camera, animation, etc.)
* ✅ Modular tutorials via ScriptableObjects
* ✅ Auto-cleanup to avoid leaks or stale listeners

---

Happy coding!

