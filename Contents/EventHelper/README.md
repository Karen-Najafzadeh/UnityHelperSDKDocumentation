Here’s a complete guide to the UnityHelperSDK **event helper** system—covering its design, core classes, data flows, and how to integrate it into your Unity projects.

---

## Overview

The **EventHelper** subsystem provides a **type‑safe**, **thread‑safe**, **priority‑driven**, and **lifecycle‑aware** pub/sub framework for Unity, with support for:

* **Strong typing**: events are value‑type structs (no string keys).
* **Automatic cleanup**: handlers tied to objects or scopes are removed when those objects are destroyed.
* **Prioritized dispatch**: `EventPriority` controls ordering.
* **Coroutine support**: handlers can run as coroutines.
* **Chained events**: sequences of `IChainedEvent` processed in order.
* **Scoped subscriptions**: subscribe/unsubscribe all handlers for a given `MonoBehaviour`.

All subscriptions go through a single static `EventHelper` class; `EventHandlerComponent` adds Unity‑lifecycle convenience on MonoBehaviours.

---

## 1. Core Data Types

### 1.1. EventPriority

```csharp
public enum EventPriority
{
    Critical = 0,
    High     = 1,
    Normal   = 2,
    Low      = 3,
    Background = 4
}
```

Lower numeric values execute **earlier**.

---

### 1.2. Handler Wrappers

* **EventHandlerWrapperBase**

  * Holds metadata: `Priority`, `IsCoroutine`, `CoroutineRunner`, `Tag`.
  * Abstract members: `Type EventType`, `Delegate Handler`.

* **EventHandlerWrapper<T>**

  * Wraps an `Action<T>` delegate and exposes its `EventType` and `Handler`.

---

### 1.3. Chained Events

* **IChainedEvent**

  ```csharp
  interface IChainedEvent {
    IEnumerator ProcessEvent();
    bool IsComplete { get; }
    bool HasError    { get; }
    Exception Error  { get; }
    IChainedEvent NextEvent { get; set; }
  }
  ```
* **UIEvent**: base implementation for coroutine‑driven UI actions. You derive it and implement `ProcessEvent()`; it sets `IsComplete` when done.
* **EventCompletionTracker**: tracks pending `IChainedEvent`s, moves them to completed when `IsComplete` is true, captures any errors, and can clear or report them.

---

## 2. EventHelper Static API

### 2.1. Subscribe / Unsubscribe

```csharp
// Basic subscription:
EventHelper.Subscribe<MyEvent>(handler, priority, isCoroutine, coroutineRunner, tag);

// Scoped subscription:
EventHelper.SubscribeScoped<MyEvent>(myMonoBehaviour, handler, priority, isCoroutine, tag);

// Unsubscribe a specific handler:
EventHelper.Unsubscribe<MyEvent>(handler);

// Unsubscribe all for a MonoBehaviour scope:
EventHelper.UnsubscribeScope(myMonoBehaviour);
```

* **`T`** must be a `struct` (value‑type event data).
* `isCoroutine=true` runs the handler inside a Unity coroutine on `coroutineRunner`.
* `tag` is arbitrary metadata you can use for grouping or filtering.

### 2.2. Trigger

```csharp
EventHelper.Trigger(new MyEvent { /* fill fields */ });
```

* Takes a copy of the handler list under lock, then executes in **priority order**.
* Exceptions within handlers are caught and logged.

---

### 2.3. Chained Events & Processing

```csharp
// Enqueue a chain starting point:
EventHelper.TriggerChainedEvent(firstChainedEvent);

// In a MonoBehaviour.Update():
EventHelper.ProcessEvents();
```

* Chained: when one `IChainedEvent` completes, its `NextEvent` is enqueued.
* The `EventCompletionTracker` updates to remove completed events and tracks errors.

---

## 3. MonoBehaviour Integration

### 3.1. EventHandlerComponent

Attach to any `GameObject` to automatically:

* **Subscribe** handlers via `AddHandler<T>(handler)`.
* **Unsubscribe** all tracked handlers on `OnDestroy()`.
* **Notify** any `IEventHandler` components to run custom cleanup.

```csharp
public class MyListener : MonoBehaviour
{
    private EventHandlerComponent _ehc;

    void Awake() {
      _ehc = gameObject.AddComponent<EventHandlerComponent>();
      _ehc.AddHandler<MyEvent>(OnMyEvent);
    }

    void OnMyEvent(MyEvent e) {
      // handle…
    }
}
```

When the GameObject is destroyed, all its subscriptions are cleaned up automatically.

---

## 4. Putting It All Together

1. **Define your event type**:

   ```csharp
   public struct PlayerDamagedEvent
   {
     public int DamageAmount;
     public GameObject Target;
   }
   ```
2. **Subscribe** early (e.g. in `Awake`):

   ```csharp
   EventHelper.Subscribe<PlayerDamagedEvent>(OnDamaged, EventPriority.High, isCoroutine: false);
   ```

   Or within a scope:

   ```csharp
   _ehc.AddHandler<PlayerDamagedEvent>(OnDamaged);
   ```
3. **Trigger** when appropriate:

   ```csharp
   EventHelper.Trigger(new PlayerDamagedEvent { DamageAmount = 10, Target = playerObj });
   ```
4. **Process chained UI events** if you’re using them:

   ```csharp
   EventHelper.TriggerChainedEvent(myChainedUIEvent);
   // …in your Update:
   EventHelper.ProcessEvents();
   ```
5. **Unsubscribe** (if needed manually):

   ```csharp
   EventHelper.Unsubscribe<PlayerDamagedEvent>(OnDamaged);
   // or for all tied to a MonoBehaviour:
   EventHelper.UnsubscribeScope(this);
   ```

---

## 5. API Quick Reference

| Call                                     | Description                                |
| ---------------------------------------- | ------------------------------------------ |
| `Subscribe<T>(handler, priority,…)`      | Register an event handler                  |
| `SubscribeScoped<T>(scope, handler,…)`   | Register tied to `scope`, auto‑cleanup     |
| `Trigger<T>(eventData)`                  | Fire an event                              |
| `Unsubscribe<T>(handler)`                | Remove a specific handler                  |
| `UnsubscribeScope(object scope)`         | Remove *all* handlers for that `scope`     |
| `TriggerChainedEvent(IChainedEvent evt)` | Enqueue a coroutine‑style chained event    |
| `ProcessEvents()`                        | Advance chained events; call in `Update()` |

---

With this framework, you get a robust, maintainable event system—no magic strings, full Unity‑lifecycle integration, and fine control over execution order and scope. 

---

Here are two concrete examples of custom `IChainedEvent` implementations and how you’d wire them up and process them in your game.

---

## 1. A Simple UI Fade‑In → Delay → Fade‑Out Sequence

```csharp
using System.Collections;
using UnityEngine;
using UnityHelperSDK;

public class UIFadeEvent : UIEvent
{
    private readonly CanvasGroup _canvasGroup;
    private readonly float _duration;

    public UIFadeEvent(CanvasGroup canvasGroup, float duration, MonoBehaviour runner = null)
        : base(runner)
    {
        _canvasGroup = canvasGroup;
        _duration = duration;
    }

    public override IEnumerator ProcessEvent()
    {
        // Fade in
        float elapsed = 0f;
        while (elapsed < _duration)
        {
            _canvasGroup.alpha = Mathf.Lerp(0f, 1f, elapsed / _duration);
            elapsed += Time.deltaTime;
            yield return null;
        }
        _canvasGroup.alpha = 1f;

        Complete();
    }
}
```

```csharp
// In some MonoBehaviour that kicks off the sequence:
public class TutorialUIController : MonoBehaviour
{
    [SerializeField] private CanvasGroup _tooltipGroup;

    private void Start()
    {
        // Create fade‑in, wait‑for‑seconds, then fade‑out
        var fadeIn  = new UIFadeEvent(_tooltipGroup, 0.5f, this);
        var wait    = new WaitEvent(2f, this);          // defined below
        var fadeOut = new UIFadeEvent(_tooltipGroup, 0.5f, this);

        // Chain them
        fadeIn.NextEvent  = wait;
        wait.NextEvent    = fadeOut;

        // Enqueue the first
        EventHelper.TriggerChainedEvent(fadeIn);
    }

    private void Update()
    {
        EventHelper.ProcessEvents();
    }
}
```

```csharp
// A tiny “wait” event you can reuse
public class WaitEvent : UIEvent
{
    private readonly float _seconds;
    private float _startTime;

    public WaitEvent(float seconds, MonoBehaviour runner = null) 
        : base(runner)
    {
        _seconds = seconds;
    }

    public override IEnumerator ProcessEvent()
    {
        _startTime = Time.time;
        while (Time.time - _startTime < _seconds)
            yield return null;
        Complete();
    }
}
```

---

## 2. Dialogue → Highlight → Input‑Check Chain

Imagine you want to show a dialogue box, then highlight a UI button, then wait for the player to click it.

```csharp
// 2.1 DialogueEvent shows text, waits for the player to dismiss it
public class DialogueEvent : UIEvent
{
    private readonly string _dialogueKey;
    private readonly DialogueUI _ui;

    public DialogueEvent(string dialogueKey, DialogueUI ui, MonoBehaviour runner = null)
        : base(runner)
    {
        _dialogueKey = dialogueKey;
        _ui = ui;
    }

    public override IEnumerator ProcessEvent()
    {
        bool done = false;
        _ui.Show(_dialogueKey, () => done = true);
        while (!done)
            yield return null;
        Complete();
    }
}
```

```csharp
// 2.2 HighlightButtonEvent pulses a highlight on a button until clicked
public class HighlightButtonEvent : UIEvent
{
    private readonly Button _button;
    private bool _clicked;

    public HighlightButtonEvent(Button button, MonoBehaviour runner = null)
        : base(runner)
    {
        _button = button;
    }

    public override IEnumerator ProcessEvent()
    {
        // Subscribe click
        _button.onClick.AddListener(OnClicked);

        // Pulse effect
        while (!_clicked)
        {
            // e.g. animate scale or color here...
            yield return new WaitForSeconds(0.5f);
        }

        // Cleanup
        _button.onClick.RemoveListener(OnClicked);
        Complete();
    }

    private void OnClicked() => _clicked = true;
}
```

```csharp
// 2.3 Putting it all together
public class TutorialExample : MonoBehaviour
{
    [SerializeField] private DialogueUI _dialogueUI;
    [SerializeField] private Button   _nextButton;

    private void Start()
    {
        var dialogEvt  = new DialogueEvent("welcome_message", _dialogueUI, this);
        var highlight  = new HighlightButtonEvent(_nextButton, this);

        dialogEvt.NextEvent = highlight;
        EventHelper.TriggerChainedEvent(dialogEvt);
    }

    private void Update()
    {
        EventHelper.ProcessEvents();
    }
}
```

---

### How It Works

1. **Implement** `IChainedEvent` (or derive from `UIEvent`) and inside `ProcessEvent()`

   * Drive your UI/logic (tweens, waits, dialogues, input).
   * Call `Complete()` when done (or `SetError(ex)` on failure).

2. **Chain** them by setting `.NextEvent`.

3. **Trigger** the first with `EventHelper.TriggerChainedEvent(...)`.

4. **Process** in your main `Update()` via `EventHelper.ProcessEvents()`
   — this will automatically enqueue each `NextEvent` once its predecessor reports `IsComplete`.

With these patterns you can build arbitrarily complex, linear or branching sequences of UI and logic steps, all managed by the same event‑helper infrastructure. Let me know if you’d like more variations or troubleshooting tips!
