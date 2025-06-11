**Generic Observer Pattern with Unity Integration**

This document provides a comprehensive guide to the `ObserverPattern<T>` class an observer/event system abstraction built atop `UnityHelperSDK.EventHelper`—complete with **extensive** usage examples.

---

## Table of Contents

1. [Overview](#overview)
2. [API Reference](#api-reference)

   * Constructor
   * Subscribe
   * Unsubscribe
   * Notify
3. [Example Scenarios](#example-scenarios)

   1. Health System (int)
   2. Position Tracking (Vector3)
   3. Score Updates with Priority and Coroutine
   4. Custom Event Struct (DamageInfo)
   5. UI Binding with Automatic Cleanup
   6. Grouped Events via Tag
   7. Delayed Reaction via Task/Coroutine
4. [Best Practices & Edge Cases](#best-practices)
5. [Appendix: Full Code Listing](#full-code)

---

## 1. Overview

`ObserverPattern<T>` wraps UnityHelperSDK's `EventHelper`, offering:

* **Thread-safe** event subscription.
* **Priority** ordering.
* **Coroutine** vs. synchronous callbacks.
* **Automatic cleanup** via `EventHandlerComponent` on GameObjects.
* **Tag** grouping to isolate event channels.

Generics require `T : struct` so events are value types, efficient and copy-safe.

---

## 2. API Reference

### Constructor

```csharp
public ObserverPattern(MonoBehaviour coroutineRunner = null, object tag = null)
```

* **coroutineRunner**: MonoBehaviour used to start coroutine observers.
* **tag**: Any object to namespace subscriptions (multiple independent channels).

---

### Subscribe

```csharp
public void Subscribe(
    Action<T> observer,
    EventPriority priority = EventPriority.Normal,
    bool isCoroutine = false,
    GameObject gameObject = null
)
```

* **observer**: Callback receiving `T`.
* **priority**: `Low`, `Normal`, `High`, etc.
* **isCoroutine**: Wrap as coroutine via `coroutineRunner`.
* **gameObject**: Attach `EventHandlerComponent` for auto-unsubscribe on destroy.

Throws `ArgumentNullException` if `observer` null.

---

### Unsubscribe

```csharp
public void Unsubscribe(Action<T> observer)
```

* Removes subscription. Throws if `observer` null.

---

### Notify

```csharp
public void Notify(T data)
```

* Triggers all observers (in priority order) with `data`.

---

## 3. Example Scenarios

### 3.1 Health System (int)

```csharp
// Setup
public class HealthSystem : MonoBehaviour {
    private ObserverPattern<int> _onHealthChanged;
    private int _health;

    private void Awake() {
        _onHealthChanged = new ObserverPattern<int>(this);
    }

    public void SetHealth(int value) {
        _health = value;
        _onHealthChanged.Notify(_health);
    }
    public void AddObserver(Action<int> obs) {
        _onHealthChanged.Subscribe(obs);
    }
}

// Usage
ihealthSystem.AddObserver(h => Debug.Log($"Health: {h}"));
healthSystem.SetHealth(80);
```

---

### 3.2 Position Tracking (Vector3)

```csharp
// Track player movement
var onPositionChanged = new ObserverPattern<Vector3>(this, tag: "Movement");

void UpdatePosition(Vector3 newPos) {
    onPositionChanged.Notify(newPos);
}

// Subscribe elsewhere
onPositionChanged.Subscribe(pos => {
    transform.position = pos;
});
```

---

### 3.3 Score Updates with Priority and Coroutine

```csharp
var onScore = new ObserverPattern<int>(this);

// High-priority to update UI first
onScore.Subscribe(score => UpdateHUD(score), priority: EventPriority.High);

// Coroutine-based analytics logging
tonScore.Subscribe(
    async score => await AnalyticsService.LogScore(score),
    priority: EventPriority.Low,
    isCoroutine: true
);

// Trigger
onScore.Notify(1000);
```

---

### 3.4 Custom Event Struct (DamageInfo)

```csharp
public struct DamageInfo { public int amount; public GameObject source; }

var onDamage = new ObserverPattern<DamageInfo>(this);

// Subscribe
onDamage.Subscribe(info => {
    health -= info.amount;
    Debug.Log($"Hit by {info.source.name}: {info.amount}");
});

// Notify
onDamage.Notify(new DamageInfo { amount = 25, source = enemyGO });
```

---

### 3.5 UI Binding with Automatic Cleanup

```csharp
public class HealthBar : MonoBehaviour {
    private ObserverPattern<int> _hpEvents;
    public HealthSystem healthSystem;

    private void Start() {
        _hpEvents = healthSystem.GetObserver();
        _hpEvents.Subscribe(_ => UpdateBar(_),
            gameObject: this.gameObject);
    }

    private void OnDestroy() {
        // No manual unsubscribe needed
    }
}
```

---

### 3.6 Grouped Events via Tag

```csharp
var eventsA = new ObserverPattern<int>(tag: "GroupA");
var eventsB = new ObserverPattern<int>(tag: "GroupB");

eventsA.Subscribe(i => Debug.Log($"A: {i}"));
eventsB.Subscribe(i => Debug.Log($"B: {i}"));

eventsA.Notify(1); // only A listeners
eventsB.Notify(2); // only B listeners
```

---

### 3.7 Delayed Reaction via Task/Coroutine

```csharp
// Subscribe as coroutine
onPositionChanged.Subscribe(
    async pos => {
        await Task.Delay(500);
        Debug.Log($"Delayed pos: {pos}");
    },
    isCoroutine: true
);
```

---

## 4. Best Practices & Edge Cases

* **Null checks**: `observer != null` enforced.
* **MonoBehaviour runner**: Required for `isCoroutine=true` observers.
* **Tag isolation**: Use tags to separate channels.
* **Cleanup**: Provide `gameObject` to auto-unsubscribe.
* **Performance**: Avoid heavy work in high-frequency events.

---

## 5. Appendix: Full Code Listing

```csharp
public class ObserverPattern<T> where T : struct {
    private readonly MonoBehaviour _coroutineRunner;
    private readonly object _tag;

    public ObserverPattern(MonoBehaviour coroutineRunner = null, object tag = null) {
        _coroutineRunner = coroutineRunner;
        _tag = tag;
    }

    public void Subscribe(Action<T> observer, EventPriority priority = EventPriority.Normal,
        bool isCoroutine = false, GameObject gameObject = null) { /* ... */ }

    public void Unsubscribe(Action<T> observer) { /* ... */ }

    public void Notify(T data) { /* ... */ }
}
```
