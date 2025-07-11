# UnityHelperSDK Tutorial System

## Overview
The Tutorial System in UnityHelperSDK provides a modular, extensible, and testable framework for building in-game tutorials, onboarding flows, and contextual help. It is designed for flexibility, editor tooling, and integration with modern Unity workflows.

---

## Key Components

- **TutorialSequence**: ScriptableObject representing a sequence of tutorial steps, with branching, conditions, and metadata.
- **TutorialStep**: ScriptableObject representing a single tutorial instruction, with conditions, actions, and UI configuration.
- **TutorialManager**: MonoBehaviour façade that bridges Unity's lifecycle with the core tutorial logic. Forwards events and updates to the service layer.
- **ITutorialService**: Interface for the core tutorial logic, enabling testability and dependency injection.
- **TutorialService**: Pure C# implementation of the tutorial logic (progression, branching, state, etc).
- **IServiceLocator**: Interface for integrating with dependency injection containers or service locators.
- **TutorialManagerWithServiceLocator**: Example of integrating the tutorial system with a DI container.

---

## Basic Usage

### 1. Create Tutorial Steps and Sequences
- Create `TutorialStep` and `TutorialSequence` assets via the Unity Editor (right-click in Project window > Create > UnityHelperSDK > Tutorial Step/Sequence).
- Configure steps, conditions, actions, and branching in the inspector.
- Assign steps to sequences.

### 2. Add the TutorialManager to Your Scene
- Add the `TutorialManager` MonoBehaviour to a GameObject in your scene.
- Assign your `TutorialSequence` assets to the `AllSequences` list in the inspector.

### 3. Start a Tutorial Sequence (Code)
```csharp
using UnityHelperSDK.Tutorial;

// Reference the TutorialManager (singleton or via DI)
public TutorialSequence myTutorialSequence;

void Start() {
    TutorialManager.Instance.StartSequence(myTutorialSequence);
}
```

### 4. Advance, Skip, or Complete Steps (Code)
```csharp
// Advance to next step (e.g. after user action)
TutorialManager.Instance.AdvanceStep();

// Skip current step
TutorialManager.Instance.SkipStep();

// Complete the current sequence
TutorialManager.Instance.CompleteSequence();
```

### 5. Listen for Tutorial Events
```csharp
TutorialManager.Instance.OnStepStarted += step => Debug.Log($"Step started: {step.stepName}");
TutorialManager.Instance.OnSequenceCompleted += seq => Debug.Log($"Sequence complete: {seq.sequenceName}");
```

---

## More Examples

### Example: Full Tutorial Flow
```csharp
// 1. Create steps and sequence in the editor
// 2. Assign steps to the sequence
// 3. In your game manager:
public TutorialSequence introSequence;
void Start() {
    TutorialManager.Instance.StartSequence(introSequence);
}

// 4. In your UI or gameplay scripts, call:
TutorialManager.Instance.AdvanceStep(); // when the player completes a task
```

### Example: Triggering Steps from Events
```csharp
using UnityHelperSDK.Events;
public EventAsset stepCompleteEvent;
void OnEnable() {
    EventDispatcher.Register(stepCompleteEvent, TutorialManager.Instance.AdvanceStep);
}
void OnDisable() {
    EventDispatcher.Unregister(stepCompleteEvent, TutorialManager.Instance.AdvanceStep);
}
```

### Example: Resetting Tutorial Progress
```csharp
// (If you implement Save/Load in your TutorialService)
TutorialManager.Instance.CurrentSequence.ResetRuntimeState();
```

### Example: Custom UI Integration
```csharp
TutorialManager.Instance.OnStepStarted += ShowTutorialPanel;
TutorialManager.Instance.OnStepCompleted += HideTutorialPanel;

void ShowTutorialPanel(TutorialStep step) {
    // Show your custom UI for the step
}
void HideTutorialPanel(TutorialStep step) {
    // Hide or update your UI
}
```

---

## Advanced Usage

- Use `ITutorialService` and `TutorialService` for unit testing and custom logic.
- Integrate with a DI container using `TutorialManagerWithServiceLocator` and `IServiceLocator`.
- Use the event system to trigger tutorial progression from gameplay events:
```csharp
using UnityHelperSDK.Events;
EventDispatcher.Raise(myTutorialStep.triggerEvent);
```
- Customize UI rendering by subscribing to tutorial events and updating your own UI.

---

## Best Practices
- Use ScriptableObjects for all tutorial data for easy editing and reuse.
- Use the event system for decoupled progression triggers.
- Separate UI rendering from tutorial logic for flexibility.
- Use the service interface for testing and extensibility.

---

## Example: Integrating with the Event System
```csharp
// In a gameplay script, trigger a tutorial event
EventDispatcher.Raise(myTutorialStep.triggerEvent);

// In the tutorial system, listen for the event to advance the step
TutorialManager.Instance.AdvanceStep();
```

---

## Files
- `TutorialSequence.cs`, `TutorialStep.cs`, `TutorialManager.cs`, `ITutorialService.cs`, `TutorialService.cs`, `IServiceLocator.cs`, `TutorialManagerWithServiceLocator.cs`

---

## Support
For advanced usage, see the code comments and the service interface. Extend the system for analytics, localization, and custom UI as needed.
