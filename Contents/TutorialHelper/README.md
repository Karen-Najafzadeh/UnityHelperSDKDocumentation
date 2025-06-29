# UnityHelperSDK Tutorial System Documentation

## Overview

The UnityHelperSDK Tutorial System is a modular, extensible framework for creating, managing, and displaying interactive tutorials in Unity games. It supports complex, multi-step tutorials with branching logic, custom conditions, analytics, and seamless integration with both runtime and editor workflows.

This documentation explains the architecture, main components, and how to use the system in your own project.

---

## Architecture & Main Components

### 1. **ScriptableObject Data Model**
- **TutorialDefinition**: Stores all data for a single tutorial (ID, category, steps, conditions, etc.).
- **TutorialCategory**: Groups tutorials into categories for organization and filtering.
- **TutorialStepData**: Represents a single step in a tutorial, including its conditions and target objects.
- **TutorialConditionData**: Serializable data for conditions (event ID, type, parameters).

These ScriptableObjects are stored in `Assets/Resources/Tutorials` and can be created/edited via the Unity Editor.

### 2. **Runtime Classes**
- **TutorialRepository**: Singleton MonoBehaviour that loads, manages, and tracks all tutorials, categories, and progress. Handles persistence and acts as the main data source.
- **TutorialSequence**: Represents a sequence of steps for a tutorial, including start conditions and logic for activation/completion.
- **TutorialStep**: Represents a single step, with its own conditions and completion logic.
- **TutorialHelper**: Static class that manages the flow of tutorials, step progression, UI integration, and event handling.
- **TutorialEvents**: Defines all event structs used for communication between tutorial components (start, complete, step, analytics, etc.).

### 3. **Editor Tools**
- **TutorialEditorWindow**: Custom Unity Editor window for managing tutorials and categories visually.
- **TutorialTreeView**: UIElements-based tree view for browsing and selecting tutorials/categories in the editor.

---

## How the System Works

### 1. **Data Definition (ScriptableObjects)**
- Create `TutorialCategory` and `TutorialDefinition` assets in `Assets/Resources/Tutorials`.
- Each `TutorialDefinition` contains:
  - Unique ID, category, title, description
  - List of dependencies (other tutorials that must be completed first)
  - List of start conditions (when the tutorial should be available)
  - List of steps (`TutorialStepData`), each with its own conditions and completion logic

### 2. **Initialization**
- On game start, `TutorialRepository.Instance` is created (singleton pattern).
- It loads all categories and definitions from resources, builds runtime data structures, and loads player progress.
- `TutorialHelper.Initialize()` can be called to set up the tutorial system and register available tutorials.

### 3. **Starting a Tutorial**
- Tutorials can be started manually (e.g., `await TutorialHelper.StartTutorial("tutorial_id")`) or automatically if their start conditions are met.
- Start conditions are checked using event-driven logic (see `TutorialEvents.TutorialStartConditionEvent`).
- If a tutorial is set to `OnlyShowOnce`, it will not be shown again after completion.

### 4. **Step Progression**
- Each tutorial consists of ordered steps (`TutorialStep`).
- Each step can have multiple conditions (e.g., player actions, reaching a location, etc.).
- The system waits for all conditions to be met before progressing to the next step.
- Completion of a step can also be gated by custom logic or events.

### 5. **UI Integration**
- The system supports highlighting UI elements, showing dialogue, and displaying arrows or overlays to guide the player.
- UI references (canvas, prefabs) are managed by `TutorialHelper`.
- Animations and delays can be configured for a smooth user experience.

### 6. **Analytics & Events**
- All major tutorial events (start, complete, step, analytics) are triggered via `TutorialEvents`.
- You can subscribe to these events for custom analytics, logging, or additional game logic.

### 7. **Persistence**
- Player progress (completed tutorials, current step, etc.) is saved and loaded automatically by `TutorialRepository`.
- Supports JSON serialization for easy integration with cloud save or external systems.

### 8. **Editor Workflow**
- Use the `Tutorial Editor` window (menu: Unity Helper SDK/Tutorial Editor) to:
  - Create, edit, and organize tutorials and categories
  - Edit steps, conditions, and dependencies visually
  - Save changes directly to ScriptableObject assets

---

## How to Use the Tutorial System in Your Project

### 1. **Setup**
- Import all scripts and editor tools from the `HelperUtilities/TutorialUtilities` folders.
- Ensure Newtonsoft.Json is available in your project (for serialization).

### 2. **Create Categories and Tutorials**
- In the Unity Editor, open the `Tutorial Editor` window.
- Create new categories and tutorials as needed.
- Define steps, conditions, and assign target objects for each step.

### 3. **Initialize at Runtime**
- Call `TutorialHelper.Initialize()` at game startup (e.g., in a bootstrap script or main menu).
- Optionally, register additional tutorials or subscribe to tutorial events for custom logic.

### 4. **Start Tutorials**
- To start a tutorial manually:
  ```csharp
  await TutorialHelper.StartTutorial("tutorial_id");
  ```
- Tutorials can also start automatically if their start conditions are met (e.g., player reaches a certain level).

### 5. **Custom Conditions and Events**
- Define custom conditions by extending the event system or using `CustomTutorialCondition`.
- Subscribe to `TutorialEvents` for analytics, UI updates, or other game logic.

### 6. **Persistence and Progress**
- Progress is saved automatically, but you can call `TutorialRepository.Instance.SaveProgress()` to force a save.
- To reset progress (for testing), clear the saved data or use a debug command.

---

## Examples

### Example 1: Creating a Simple Tutorial

1. **Create a Category**
   - In the editor, create a new `TutorialCategory` asset (e.g., "Onboarding").

2. **Create a Tutorial**
   - Create a new `TutorialDefinition` asset.
   - Set its ID, category, title, and description.
   - Add start conditions (e.g., player level >= 1).
   - Add steps, each with a dialogue key, target object, and conditions (e.g., click a button).

3. **Initialize and Start**
   - In your game startup code:
     ```csharp
     TutorialHelper.Initialize();
     await TutorialHelper.StartTutorial("onboarding_tutorial");
     ```

### Example 2: Assigning Target Objects at Runtime

Sometimes, you may want to assign or override the target GameObject for a tutorial step at runtime (for example, if the UI is dynamically generated). You can do this by accessing the `TutorialStep` instance after the tutorial is loaded or before starting it:

```csharp
// After TutorialHelper.Initialize() and before starting the tutorial
var tutorial = TutorialRepository.Instance.ActiveSequences["onboarding_tutorial"];
var step = tutorial.Steps[0]; // e.g., first step
step.Target = GameObject.Find("DynamicButton"); // Assign the actual GameObject
```

If you need to assign targets dynamically for all steps, you can loop through them and set the `Target` property as needed.

> **Note:** If you are using ScriptableObjects for tutorial definitions, the `TargetObject` field in `TutorialStepData` can be left empty and set at runtime as above.

### Example 3: Creating Tutorials at Runtime

You can create and register new tutorials at runtime if your game requires dynamic tutorials (e.g., for user-generated content or A/B testing):

```csharp
// Create a new TutorialSequence
var sequence = new TutorialSequence(
    id: "dynamic_tutorial",
    onlyShowOnce: false,
    requiredLevel: 0,
    categoryId: "dynamic"
);

// Create steps
var step1 = new TutorialStep("step1", "Welcome!", GameObject.Find("StartButton"));
step1.AddCondition(evt => { evt.HasMetConditions = true; }); // Always true for demo
sequence.AddStep(step1);

// Register the tutorial
TutorialHelper.RegisterTutorial(sequence);

// Start it
await TutorialHelper.StartTutorial("dynamic_tutorial");
```

You can add as many steps and conditions as needed. This approach is useful for procedural or context-sensitive tutorials.

### Example 4: Subscribing to Tutorial Events

You can listen for tutorial events to trigger custom logic, analytics, or UI updates:

```csharp
// Subscribe to tutorial started event
EventHelper.Subscribe<TutorialEvents.TutorialStartedEvent>(evt => {
    Debug.Log($"Tutorial started: {evt.TutorialId}");
});

// Subscribe to step completed event
EventHelper.Subscribe<TutorialEvents.TutorialStepCompletedEvent>(evt => {
    Debug.Log($"Step completed: {evt.StepId}");
});
```

---

## FAQ & Advanced Usage

### Q: How do I make a step wait for a specific player action?
A: Add a condition to the step that sets `evt.HasMetConditions = true` when the action occurs. For example, subscribe to a button click and update the event:

```csharp
step.AddCondition(evt => {
    MyButton.onClick.AddListener(() => evt.HasMetConditions = true);
});
```

### Q: Can I change the dialogue or step content at runtime?
A: Yes. You can modify the `DialogueKey` or any other property of a `TutorialStep` before the step is shown.

### Q: How do I reset or replay a tutorial?
A: Remove the tutorial ID from `TutorialRepository.Instance.CompletedTutorials` and call `StartTutorial` again. For a full reset, clear all progress data.

### Q: Can I add steps to an existing tutorial at runtime?
A: Yes. Access the `TutorialSequence` and use `AddStep()` to append new steps before starting or while paused.

---

## Best Practices
- Use categories to organize tutorials for scalability.
- Keep step conditions simple and use custom conditions for complex logic.
- Use analytics events to track player progress and improve onboarding.
- Test tutorials thoroughly using the editor tools and runtime event logs.
- Assign target objects at runtime if your UI is dynamic or instantiated at runtime.
- For localization, use the `DialogueKey` to fetch localized strings.

---

## Summary Table

| Component                | Purpose                                                      |
|--------------------------|--------------------------------------------------------------|
| TutorialDefinition       | Stores all data for a single tutorial                        |
| TutorialCategory         | Groups tutorials for organization                            |
| TutorialStepData         | Represents a single step in a tutorial                       |
| TutorialConditionData    | Serializable data for conditions                             |
| TutorialRepository       | Loads, manages, and tracks all tutorials and progress        |
| TutorialSequence         | Represents a sequence of steps for a tutorial                |
| TutorialStep             | Represents a single step, with its own logic                 |
| TutorialHelper           | Manages flow, UI, and event handling                         |
| TutorialEvents           | Defines all event structs for communication                  |
| TutorialEditorWindow     | Editor window for managing tutorials and categories           |
| TutorialTreeView         | UIElements-based tree view for browsing tutorials/categories |

---

## References
- All scripts are located in `Assets/Scripts/HelperUtilities/TutorialUtilities` and `Assets/Editor/TutorialUtilities`.
- See the provided C# files for implementation details and extension points.

---

For further questions or advanced usage, refer to the code comments or contact the SDK maintainers.
