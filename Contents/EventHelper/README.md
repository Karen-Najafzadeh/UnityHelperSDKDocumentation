# UnityHelperSDK Event System

## Overview
The Event System in UnityHelperSDK provides a flexible, decoupled, and ScriptableObject-based event architecture for Unity projects. It enables you to define, categorize, and trigger events without tight coupling between components, and supports both code and inspector-driven workflows.

---

## Key Components

- **EventAsset**: ScriptableObject representing a single event definition. Can carry metadata, payload, and category.
- **CategoryAsset**: ScriptableObject for organizing events into categories for better management and UI.
- **EventDatabase**: ScriptableObject holding all event and category assets for easy management.
- **EventDispatcher**: MonoBehaviour singleton that manages event registration and dispatching at runtime.
- **EventListener**: MonoBehaviour for listening to EventAssets and invoking UnityEvents in response.
- **EventHelper**: Static utility for advanced event subscription, priority, and lifetime management (optional, for advanced users).

---

## Basic Usage

### 1. Define Events
- Create new `EventAsset` and `CategoryAsset` assets via the Unity Editor (right-click in Project window > Create > UnityHelperSDK > Event Asset/Category).
- Optionally, group events in an `EventDatabase` for easy reference.

### 2. Trigger Events (Code)
```csharp
using UnityHelperSDK.Events;

// Reference your EventAsset (assign in inspector or load via Resources)
public EventAsset myEvent;

// Trigger the event (e.g. when a button is clicked)
public void OnButtonClicked() {
    EventDispatcher.Raise(myEvent);
}
```

### 3. Listen for Events (Code)
```csharp
using UnityHelperSDK.Events;

public EventAsset myEvent;

void OnEnable() {
    EventDispatcher.Register(myEvent, OnMyEvent);
}
void OnDisable() {
    EventDispatcher.Unregister(myEvent, OnMyEvent);
}
void OnMyEvent() {
    Debug.Log("Event received!");
    // Respond to event (e.g. show UI, play sound, etc)
}
```

### 4. Listen for Events (Inspector)
- Add the `EventListener` component to a GameObject.
- Assign the `EventAsset` to listen for.
- Assign a UnityEvent response in the inspector.

---

## More Examples

### Example: UI Button triggers an event, another script listens
```csharp
// ButtonTrigger.cs
using UnityEngine;
using UnityHelperSDK.Events;
public class ButtonTrigger : MonoBehaviour {
    public EventAsset buttonClickedEvent;
    public void OnButtonClick() {
        EventDispatcher.Raise(buttonClickedEvent);
    }
}

// ListenerScript.cs
using UnityEngine;
using UnityHelperSDK.Events;
public class ListenerScript : MonoBehaviour {
    public EventAsset buttonClickedEvent;
    void OnEnable() {
        EventDispatcher.Register(buttonClickedEvent, OnButtonClicked);
    }
    void OnDisable() {
        EventDispatcher.Unregister(buttonClickedEvent, OnButtonClicked);
    }
    void OnButtonClicked() {
        Debug.Log("Button was clicked!");
    }
}
```

### Example: Analytics Event
```csharp
// In your gameplay code
public EventAsset levelCompletedEvent;
void OnLevelComplete() {
    EventDispatcher.Raise(levelCompletedEvent);
    // Your analytics system can listen for this event
}
```

### Example: Debugging Event Flow
- Add Debug.Log statements in your event listeners.
- Use the Unity Console to verify event order and timing.
- Use categories and tags to filter and organize events in the editor.

---

## Advanced Usage

- Use `EventHelper` for scoped, prioritized, or coroutine-based event handling.
- Use `EventDatabase` to manage and query all events and categories in your project.
- Use `CategoryAsset` to organize events for editor tools or analytics.

---

## Best Practices
- Use ScriptableObject events for decoupling systems (UI, gameplay, audio, etc).
- Use categories for analytics, filtering, and editor tooling.
- Use `EventListener` for quick inspector-based event wiring.
- Use `EventHelper` for advanced scenarios (lifetime, priority, async).

---

## Example: Triggering a Tutorial Event
```csharp
// Suppose you have a TutorialStarted EventAsset
EventDispatcher.Raise(tutorialStartedEventAsset);
```

## Example: Listening for a Tutorial Event
```csharp
EventDispatcher.Register(tutorialStartedEventAsset, OnTutorialStarted);
void OnTutorialStarted() {
    // Start tutorial UI, etc
}
```

---

## Extending
- Add custom payloads to events via the `EventAsset` data fields.
- Extend `EventListener` for custom UnityEvent signatures.
- Integrate with analytics or UI systems via event categories.

---

## Files
- `EventAsset.cs`, `CategoryAsset.cs`, `EventDatabase.cs`, `EventDispatcher.cs`, `EventListener.cs`, `EventHelper.cs`, `EventTypes.cs`, `EventsRepository.cs`

---

## Support
For more advanced usage, see the code comments and the `EventHelper` API.
