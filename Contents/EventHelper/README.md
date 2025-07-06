
# UnityHelperSDK Event System

A modern, designer-friendly, and scalable event system for Unity, built on ScriptableObject-based events and categories. This system enables robust event-driven workflows, advanced editor tooling, and flexible data payloads for both programmers and designers.

---

## Table of Contents
- [Overview](#overview)
- [Key Features](#key-features)
- [Event System Architecture](#event-system-architecture)
- [Getting Started](#getting-started)
- [Creating Events & Categories](#creating-events--categories)
- [Triggering Events](#triggering-events)
- [Listening to Events](#listening-to-events)
- [Passing Data with Events](#passing-data-with-events)
- [Advanced Usage](#advanced-usage)
- [Best Practices](#best-practices)
- [FAQ](#faq)

---

## Overview

This event system replaces traditional C# event patterns with asset-based, ScriptableObject-driven events and categories. It is optimized for both code and designer workflows, supporting advanced filtering, tagging, and metadata.

---

## Key Features
- **ScriptableObject-based Events**: Events are assets, not structs or enums.
- **Categories**: Organize events with rich metadata (color, icon, tags, description).
- **Designer-Friendly**: Create, edit, and manage events/categories in the Unity Editor.
- **Advanced Editor Tools**: Bulk operations, filtering, search, and details panel.
- **Flexible Data Payloads**: Events can carry arbitrary data fields.
- **Runtime & Editor Support**: Trigger and listen to events in both play mode and edit mode.

---

## Event System Architecture

- **EventAsset**: ScriptableObject representing a single event, with metadata and data fields.
- **CategoryAsset**: ScriptableObject representing a category for grouping events.
- **EventDatabase**: ScriptableObject holding all events and categories for easy management.
- **EventDispatcher**: Central static dispatcher for raising and listening to events.
- **EventListener**: MonoBehaviour for listening to events and invoking UnityEvents.
- **EventManagerWindow**: Editor window for managing events and categories.

---

## Getting Started

1. **Import the Event System**: Add the `EventsUtilities` folder to your project.
2. **Create an Event Database**: In the Unity Editor, go to `Tools > UnityHelperSDK > Event Manager` and click "Create Event Database" if one does not exist.
3. **Open the Event Manager**: Use the Event Manager window to create and manage events and categories.

---

## Creating Events & Categories

### Creating an Event
1. Open the Event Manager window (`Tools > UnityHelperSDK > Event Manager`).
2. In the **Events** tab, click **Add New Event**.
3. Fill in the event name, description, category, color, icon, and tags.
4. (Optional) Add data fields for custom payloads.

**Example 1: Simple Event**
- Name: `OnGameStarted`
- Description: "Fired when the game starts."
- Category: `System`

**Example 2: Event with Data**
- Name: `OnPlayerHealthChanged`
- Data Fields: `currentHealth` (float), `maxHealth` (float), `damage` (float)

### Creating a Category
1. In the **Categories** tab, click **Add New Category**.
2. Set the category name, color, icon, description, and tags.
3. Assign events to this category as needed.

**Example 1: System Category**
- Name: `System`
- Color: Blue
- Description: "System-level events."

**Example 2: Player Category**
- Name: `Player`
- Color: Green
- Description: "Player-related events."

---

## Triggering Events

You can trigger events from code using the static `EventDispatcher.Raise` or `EventHelper.Trigger` methods.

```csharp
// Example 1: Trigger a simple event
public EventAsset onGameStartedEvent;

void Start() {
    EventDispatcher.Raise(onGameStartedEvent);
}

// Example 2: Trigger an event with data
public EventAsset onPlayerHealthChangedEvent;

void TakeDamage(float damage) {
    onPlayerHealthChangedEvent.SetValue("damage", damage);
    onPlayerHealthChangedEvent.SetValue("currentHealth", currentHealth);
    onPlayerHealthChangedEvent.SetValue("maxHealth", maxHealth);
    EventDispatcher.Raise(onPlayerHealthChangedEvent);
}
```

---

## Listening to Events

You can listen to events in two main ways:

### 1. Using the `EventListener` MonoBehaviour
Attach the `EventListener` component to a GameObject and assign the `EventAsset` and a UnityEvent response.

**Example 1: Listen for a simple event**
- Add `EventListener` to a GameObject.
- Assign `onGameStartedEvent`.
- Add a UnityEvent response in the Inspector.

**Example 2: Listen for an event in code**
```csharp
public EventAsset onPlayerHealthChangedEvent;

void OnEnable() {
    EventDispatcher.Register(onPlayerHealthChangedEvent, OnHealthChanged);
}

void OnDisable() {
    EventDispatcher.Unregister(onPlayerHealthChangedEvent, OnHealthChanged);
}

void OnHealthChanged() {
    float damage = onPlayerHealthChangedEvent.GetValue<float>("damage");
    Debug.Log($"Player took {damage} damage!");
}
```

### 2. Using `EventHelper` for Advanced Scenarios
`EventHelper` supports advanced subscription patterns, priorities, and scoped handlers.

**Example 1: Subscribe with priority**
```csharp
EventHelper.Subscribe(onGameStartedEvent, MyHandler, EventPriority.High);
```

**Example 2: Scoped subscription (auto-cleanup)**
```csharp
EventHelper.SubscribeScoped(this, onGameStartedEvent, MyHandler);
```

---

## Passing Data with Events

Each `EventAsset` can carry arbitrary data fields. Use `SetValue` before triggering, and `GetValue` in listeners.

**Example 1: Sending data**
```csharp
onPlayerHealthChangedEvent.SetValue("damage", 10f);
EventDispatcher.Raise(onPlayerHealthChangedEvent);
```

**Example 2: Receiving data**
```csharp
void OnHealthChanged() {
    float damage = onPlayerHealthChangedEvent.GetValue<float>("damage");
    Debug.Log($"Damage: {damage}");
}
```

---

## Advanced Usage
- **Bulk Operations**: Use multi-select and bulk assign/delete in the Event Manager window.
- **Tag Filtering**: Filter events and categories by tags for fast navigation.
- **Category Assignment**: Assign or reassign categories to events in bulk.
- **Custom Data Types**: Store and retrieve complex data (Vector2, Vector3, Color, etc.) using the data fields.
- **Editor-Only Metadata**: Use icons, colors, and descriptions for better organization.

---

## Best Practices
- Use categories and tags to keep your event system organized.
- Prefer asset-based events for designer workflows and easy refactoring.
- Use data fields for passing contextual information, not for large payloads.
- Clean up event listeners in `OnDisable` or `OnDestroy` to avoid memory leaks.

---

## FAQ

**Q: Can I create events at runtime?**
A: Yes, but runtime-created events are not persistent. For persistent events, create them as assets in the editor.

**Q: How do I pass custom data with an event?**
A: Use `SetValue(key, value)` before triggering, and `GetValue<T>(key)` in your listener.

**Q: Can I listen to multiple events with one listener?**
A: Yes, add multiple `EventListener` components or register multiple events in code.

**Q: How do I organize a large number of events?**
A: Use categories, tags, and the Event Manager window's filtering/search features.
