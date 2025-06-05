## Overview

This documentation describes a two-script, thread-safe, type-safe event management system for Unity, provided under the `UnityHelperSDK` namespace. It consists of:

1. **EventHandlerComponent + EventHelper** (first script)

   * A MonoBehaviour component (`EventHandlerComponent`) that can be attached to any GameObject to manage subscribing/unsubscribing handlers automatically, plus a static event bus (`EventHelper`) that handles publishing, priority ordering, coroutine-based invocation, scoping, and chained events.

2. **Wrappers, Priorities & Chaining Support** (second script)

   * Data types (e.g., `EventPriority`), abstract base classes (`EventHandlerWrapperBase`, `IChainedEvent`), concrete wrappers (`EventHandlerWrapper<T>`), and helper classes for chaining and completion tracking (`UIEvent`, `EventCompletionTracker`) as well as an interface (`IEventHandler`) for automatic cleanup.

Below is a breakdown of each class, interface, and method, along with realistic usage examples (C# in Unity 2020.3+).

---

## 1. EventHandlerComponent (MonoBehaviour)

```csharp
namespace UnityHelperSDK
{
    /// <summary>
    /// Component that can be added to any GameObject to manage event subscriptions
    /// </summary>
    public class EventHandlerComponent : MonoBehaviour
    {
        private readonly List<(Type type, Delegate handler)> _handlers = new List<(Type, Delegate)>();

        public void AddHandler<T>(Action<T> handler) where T : struct
        {
            _handlers.Add((typeof(T), handler));
            EventHelper.Subscribe(handler);
        }

        private void OnDestroy()
        {
            foreach (var (type, handler) in _handlers)
            {
                var method = typeof(EventHelper).GetMethod(nameof(EventHelper.Unsubscribe));
                var genericMethod = method.MakeGenericMethod(type);
                genericMethod.Invoke(null, new object[] { handler });
            }
            _handlers.Clear();

            // If the GameObject has any IEventHandler components, notify them
            var eventHandlers = GetComponents<IEventHandler>();
            foreach (var handler in eventHandlers)
            {
                handler.OnEventCleanup();
            }
        }
    }
}
```

### 1.1 Responsibilities

* **Centralized Subscription**

  * Call `AddHandler<T>(Action<T>)` to subscribe a handler `void HandlerMethod(T data)` to event type `T`.
  * Internally, `AddHandler` appends `(typeof(T), handler)` to the `_handlers` list and then calls the static `EventHelper.Subscribe<T>(handler)` method.

* **Automatic Unsubscription on Destroy**

  * In `OnDestroy()`, it loops through all stored `(Type, Delegate)` pairs in `_handlers`, finds `EventHelper.Unsubscribe<T>(handler)` via reflection, and invokes it. This removes the handler so that after this GameObject is destroyed, no further invocation will occur.
  * Clears `_handlers` to free resources.

* **Optional Lifecycle Hook via `IEventHandler`**

  * After unsubscribing, it checks for any components on the same GameObject implementing `IEventHandler` and calls `OnEventCleanup()` on each. This allows other scripts to know “events have been cleaned up.”

### 1.2 Usage Example

#### Scenario: ScoreManager listening for a `PlayerScored` event

1. **Define your event data struct:**

   ```csharp
   // Global struct used as event payload
   public struct PlayerScored
   {
       public int PlayerId;
       public int Points;
       
       public PlayerScored(int playerId, int points)
       {
           PlayerId = playerId;
           Points = points;
       }
   }
   ```

2. **Attach `EventHandlerComponent` to a GameObject (e.g., “GameController”).**

   ```csharp
   public class ScoreManager : MonoBehaviour, IEventHandler
   {
       private EventHandlerComponent _eventComponent;

       private void Awake()
       {
           // Ensure EventHandlerComponent is present
           _eventComponent = gameObject.AddComponent<EventHandlerComponent>();
           // Subscribe to PlayerScored events
           _eventComponent.AddHandler<PlayerScored>(OnPlayerScored);
       }

       private void OnPlayerScored(PlayerScored data)
       {
           Debug.Log($"Player {data.PlayerId} scored {data.Points} points.");
           // Update UI, internal score tables, etc.
       }

       // IEventHandler hook: called automatically when this GameObject is destroyed
       public void OnEventCleanup()
       {
           Debug.Log("ScoreManager: all event listeners have been cleaned up.");
       }
   }
   ```

3. **Trigger the event somewhere else (e.g., when a goal is scored):**

   ```csharp
   public class GoalDetector : MonoBehaviour
   {
       private void OnTriggerEnter(Collider other)
       {
           if (other.CompareTag("Player"))
           {
               int playerId = other.GetComponent<Player>().Id;
               PlayerScored evt = new PlayerScored(playerId, 1);
               EventHelper.Trigger(evt);
           }
       }
   }
   ```

* When `GoalDetector` calls `EventHelper.Trigger(new PlayerScored(...))`, all active `Action<PlayerScored>` handlers (with `Priority` ordering) will be invoked—including `OnPlayerScored`.
* If `GameController` (and thus `ScoreManager`) is destroyed, its `EventHandlerComponent.OnDestroy()` unsubscribes from all events automatically and calls `ScoreManager.OnEventCleanup()`.

---

## 2. EventHelper (Static Bus)

```csharp
namespace UnityHelperSDK
{
    public static class EventHelper
    {
        private static readonly object _lock = new object();
        private static readonly Dictionary<Type, List<EventHandlerWrapperBase>> _eventHandlers =
            new Dictionary<Type, List<EventHandlerWrapperBase>>();
        private static readonly Dictionary<object, List<(Type type, EventHandlerWrapperBase handler)>> _scopedHandlers =
            new Dictionary<object, List<(Type, EventHandlerWrapperBase)>>();
        private static readonly Queue<IChainedEvent> _chainedEvents = new Queue<IChainedEvent>();
        private static readonly EventCompletionTracker _completionTracker = new EventCompletionTracker();

        #region Event Registration

        public static void Subscribe<T>(
            Action<T> handler,
            EventPriority priority = EventPriority.Normal,
            bool isCoroutine = false,
            MonoBehaviour coroutineRunner = null,
            object tag = null
        ) where T : struct
        {
            lock (_lock)
            {
                Type type = typeof(T);
                if (!_eventHandlers.TryGetValue(type, out List<EventHandlerWrapperBase> handlers))
                {
                    handlers = new List<EventHandlerWrapperBase>();
                    _eventHandlers[type] = handlers;
                }

                // Remove any stale (dead) wrappers
                handlers.RemoveAll(h => h == null || h.Handler == null || h.Handler.Target == null);

                // Wrap the handler with metadata
                var wrapper = new EventHandlerWrapper<T>(handler, priority, isCoroutine, coroutineRunner, tag);

                // Insert in list at the correct index to maintain ascending Priority
                int insertIndex = handlers.FindIndex(h => h.Priority > priority);
                if (insertIndex == -1)
                    handlers.Add(wrapper);
                else
                    handlers.Insert(insertIndex, wrapper);
            }
        }

        public static void SubscribeScoped<T>(
            MonoBehaviour scope,
            Action<T> handler,
            EventPriority priority = EventPriority.Normal,
            bool isCoroutine = false,
            object tag = null
        ) where T : struct
        {
            if (scope == null)
                throw new ArgumentNullException(nameof(scope));

            lock (_lock)
            {
                Type type = typeof(T);

                // Fetch or create the handler list for this scope
                if (!_scopedHandlers.TryGetValue(scope, out List<(Type, EventHandlerWrapperBase)> handlers))
                {
                    handlers = new List<(Type, EventHandlerWrapperBase)>();
                    _scopedHandlers[scope] = handlers;
                }

                // Create wrapper
                EventHandlerWrapperBase wrapper = new EventHandlerWrapper<T>(handler, priority, isCoroutine, scope, tag);
                handlers.Add((type, wrapper));

                // Also subscribe normally so that this handler will be invoked
                Subscribe(handler, priority, isCoroutine, scope, tag);
            }
        }

        public static void Trigger<T>(T eventData) where T : struct
        {
            List<EventHandlerWrapperBase> handlersCopy;
            lock (_lock)
            {
                Type type = typeof(T);
                if (!_eventHandlers.TryGetValue(type, out List<EventHandlerWrapperBase> handlersList))
                    return;

                // Copy list to avoid modification during iteration
                handlersCopy = new List<EventHandlerWrapperBase>(handlersList);
            }

            // Invoke each handler in priority order
            foreach (EventHandlerWrapperBase wrapper in handlersCopy)
            {
                if (wrapper == null || wrapper.Handler == null) 
                    continue;

                if (wrapper.Handler is Action<T> action)
                {
                    if (wrapper.IsCoroutine && wrapper.CoroutineRunner != null)
                    {
                        wrapper.CoroutineRunner.StartCoroutine(ExecuteCoroutineHandler(action, eventData));
                    }
                    else
                    {
                        try
                        {
                            action(eventData);
                        }
                        catch (Exception ex)
                        {
                            Debug.LogError($"Error executing event handler for {typeof(T).Name}: {ex}");
                        }
                    }
                }
            }
        }

        private static IEnumerator ExecuteCoroutineHandler<T>(Action<T> handler, T eventData) where T : struct
        {
            try
            {
                handler(eventData);
            }
            catch (Exception ex)
            {
                Debug.LogError($"Error executing coroutine event handler for {typeof(T).Name}: {ex}");
            }
            yield break;
        }

        public static void Unsubscribe<T>(Action<T> handler) where T : struct
        {
            lock (_lock)
            {
                Type type = typeof(T);
                if (!_eventHandlers.TryGetValue(type, out List<EventHandlerWrapperBase> handlers))
                    return;

                handlers.RemoveAll(h =>
                {
                    if (h?.Handler is Action<T> actionHandler)
                        return actionHandler.Equals(handler);
                    return false;
                });
            }
        }

        public static void UnsubscribeScope(object scope)
        {
            lock (_lock)
            {
                if (_scopedHandlers.TryGetValue(scope, out List<(Type, EventHandlerWrapperBase)> handlers))
                {
                    foreach (var (type, wrapper) in handlers)
                    {
                        if (wrapper?.Handler is Delegate del)
                        {
                            // Find the generic Unsubscribe<T>(Action<T>) method
                            var method = typeof(EventHelper).GetMethod(nameof(Unsubscribe));
                            var genericMethod = method.MakeGenericMethod(type);
                            genericMethod.Invoke(null, new object[] { del });
                        }
                    }
                    _scopedHandlers.Remove(scope);
                }
            }
        }

        public static void TriggerChainedEvent(IChainedEvent firstEvent)
        {
            lock (_lock)
            {
                _chainedEvents.Enqueue(firstEvent);
                _completionTracker.AddEvent(firstEvent);
            }
        }

        public static void ProcessEvents()
        {
            // Process queued chained events
            while (_chainedEvents.Count > 0)
            {
                IChainedEvent evt = _chainedEvents.Peek();

                if (!evt.IsComplete)
                {
                    // If it is a UIEvent and has a coroutine runner, start its coroutine
                    if (evt is UIEvent uiEvent && uiEvent.CoroutineRunner != null)
                    {
                        uiEvent.CoroutineRunner.StartCoroutine(evt.ProcessEvent());
                    }
                    break;
                }

                // Current event is complete, dequeue it
                _chainedEvents.Dequeue();

                // Enqueue next chained event if any
                if (evt.NextEvent != null)
                {
                    _chainedEvents.Enqueue(evt.NextEvent);
                    _completionTracker.AddEvent(evt.NextEvent);
                }
            }

            // Update the completion tracker to remove finished events
            _completionTracker.Update();
        }

        #endregion
    }
}
```

### 2.1 Internal Data Structures

* **`_lock`**: A private object used for thread-safe `lock(...)` around all modifications to dictionaries and queues.

* **`_eventHandlers`**:

  * Key = `Type` of event (e.g., `typeof(PlayerScored)`).
  * Value = `List<EventHandlerWrapperBase>`, where each wrapper holds an `Action<T>`, its `Priority`, whether it should run as a coroutine (`IsCoroutine`), a reference to a `MonoBehaviour` if needed (`CoroutineRunner`), and an optional `Tag`.

* **`_scopedHandlers`**:

  * Key = arbitrary `object` used as a “scope” (often a `MonoBehaviour` instance).
  * Value = `List<(Type, EventHandlerWrapperBase)>`, storing which handlers were added under that scope so they can be unsubscribed all at once via `UnsubscribeScope(scope)`.

* **`_chainedEvents`**: A queue of pending `IChainedEvent` instances. Used for sequencing coroutine-based events where each event’s `ProcessEvent()` must finish before the next begins.

* **`_completionTracker`**: An instance of `EventCompletionTracker` (see section 3) that keeps track of which chained events are still pending, and which have completed or failed.

### 2.2 Key Methods

#### 2.2.1 `Subscribe<T>(Action<T>, EventPriority, bool isCoroutine, MonoBehaviour, object)`

* **Purpose**: Register a callback to be invoked whenever an event of type `T` is triggered.

* **Parameters**:

  * `handler: Action<T>`
  * `priority: EventPriority` (defaults to `Normal`)
  * `isCoroutine: bool` (defaults to `false`)
  * `coroutineRunner: MonoBehaviour` (the object on which to `StartCoroutine` if `isCoroutine == true`)
  * `tag: object` (optional user data)

* **Behavior**:

  1. Lock `_lock` to ensure thread safety.
  2. Get or create `List<EventHandlerWrapperBase>` for `typeof(T)` in `_eventHandlers`.
  3. Prune any “dead” wrappers where `h == null` or `h.Handler.Target == null` (the target object has been garbage-collected or destroyed).
  4. Create a new `EventHandlerWrapper<T> wrapper = new EventHandlerWrapper<T>(handler, priority, isCoroutine, coroutineRunner, tag)`.
  5. Find the first index in `handlers` where `h.Priority > priority`. If none found (i.e. insertIndex == −1), do `handlers.Add(wrapper)`; else insert at that index.

     * This ensures callbacks with lower numeric value of `eventPriority` (e.g. `Critical = 0`) run before `High = 1`, etc.

* **Realistic Example**
  **Scenario**: You want an enemy AI to respond to a `PlayerSpotted` event immediately (with higher priority), then a logging system to record it (normal priority).

  ```csharp
  // Event data payload
  public struct PlayerSpotted
  {
      public Vector3 Position;
      public int PlayerId;
      public PlayerSpotted(int playerId, Vector3 position)
      {
          PlayerId = playerId;
          Position = position;
      }
  }

  public class EnemyAI : MonoBehaviour
  {
      private void OnEnable()
      {
          // Subscribe at High priority so AI responds before other systems
          EventHelper.Subscribe<PlayerSpotted>(OnPlayerSpotted, EventPriority.High);
      }

      private void OnDisable()
      {
          EventHelper.Unsubscribe<PlayerSpotted>(OnPlayerSpotted);
      }

      private void OnPlayerSpotted(PlayerSpotted data)
      {
          Debug.Log($"EnemyAI: Player {data.PlayerId} spotted at {data.Position}");
          // e.g., initiate chase
      }
  }

  public class AnalyticsLogger : MonoBehaviour
  {
      private void Awake()
      {
          // Subscribe at Normal priority for logging
          EventHelper.Subscribe<PlayerSpotted>(LogPlayerSpotted, EventPriority.Normal);
      }

      private void LogPlayerSpotted(PlayerSpotted data)
      {
          Debug.Log($"AnalyticsLogger: Player {data.PlayerId} at position {data.Position}");
          // e.g., send to analytics server
      }
  }

  // Somewhere in the scene, when a player is detected:
  // EventHelper.Trigger(new PlayerSpotted(playerId, transform.position));
  ```

  * Because `EnemyAI` registered with `EventPriority.High (1)` and `AnalyticsLogger` with `EventPriority.Normal (2)`, `OnPlayerSpotted` in `EnemyAI` runs first.

---

#### 2.2.2 `SubscribeScoped<T>(MonoBehaviour scope, Action<T> handler, EventPriority, bool, object)`

* **Purpose**: Same as `Subscribe<T>`, but also associates the subscription with a “scope” object (often the `MonoBehaviour` itself). Later, calling `UnsubscribeScope(scope)` will remove all handlers registered under that scope.

* **Parameters**:

  * `scope: MonoBehaviour` (cannot be null)
  * `handler: Action<T>`
  * `priority, isCoroutine, tag` (same as `Subscribe`)

* **Behavior**:

  1. If `scope` is `null`, throw `ArgumentNullException`.
  2. Lock `_lock`.
  3. In `_scopedHandlers`, retrieve or create a `List<(Type, EventHandlerWrapperBase)>` for that `scope`.
  4. Create `EventHandlerWrapperBase wrapper = new EventHandlerWrapper<T>(...)` and add `(typeof(T), wrapper)` to the list for that `scope`.
  5. Call the normal `Subscribe(handler, priority, isCoroutine, scope, tag)` so that event is actually added to `_eventHandlers`.

* **Realistic Example**
  **Scenario**: A UI panel subscribes to various events. When the panel is closed/destroyed, you want all those subscriptions removed automatically.

  ```csharp
  public class InventoryPanel : MonoBehaviour
  {
      private void OnEnable()
      {
          // SubscribeScoped ensures that when the panel is destroyed, 
          // all these subscriptions will be removed.
          EventHelper.SubscribeScoped<ItemAdded>(this, OnItemAdded, EventPriority.Normal);
          EventHelper.SubscribeScoped<ItemRemoved>(this, OnItemRemoved, EventPriority.Normal);
      }

      private void OnDisable()
      {
          // Remove all scoped subscriptions at once
          EventHelper.UnsubscribeScope(this);
      }

      private void OnItemAdded(ItemAdded data)
      {
          // Update inventory UI
      }

      private void OnItemRemoved(ItemRemoved data)
      {
          // Update inventory UI
      }
  }
  ```

  * If the `InventoryPanel` GameObject is disabled or destroyed, calling `EventHelper.UnsubscribeScope(this)` iterates through all `(Type, wrapper)` pairs for `this` and invokes `Unsubscribe<T>(handler)` for each, ensuring a clean removal.

---

#### 2.2.3 `Trigger<T>(T eventData)`

* **Purpose**: Broadcast an event payload `eventData` of type `T` to all subscribed handlers, in order of ascending `Priority` (i.e. `Critical (0)` first, then `High (1)`, etc.).

* **Behavior**:

  1. Lock `_lock`, look up the list of `EventHandlerWrapperBase` for `typeof(T)`. If none, return early.
  2. Make a shallow copy `handlersCopy = new List<>(handlers)` so that if any handler subscribes/unsubscribes during invocation, it doesn’t mutate the iteration.
  3. Unlock and iterate over `handlersCopy`. For each `wrapper`:

     * If `wrapper == null` or `wrapper.Handler == null`, skip.
     * Cast `wrapper.Handler as Action<T>`.
     * If `wrapper.IsCoroutine == true` and `wrapper.CoroutineRunner` is non-null, start `ExecuteCoroutineHandler(action, eventData)` on `CoroutineRunner`.
     * Otherwise, invoke `action(eventData)` inside a try/catch—if an exception occurs, log it via `Debug.LogError` but continue to the next handler.

* **Coroutine Handling**:

  * `ExecuteCoroutineHandler<T>(Action<T> handler, T eventData)` is a trivial coroutine that just calls `handler(eventData)` inside a `try/catch`, then `yield break`.
  * This allows subscribers to be automatically run in a coroutine context if they requested `isCoroutine = true` and provided a valid `CoroutineRunner` (usually `this` MonoBehaviour).

* **Realistic Example**
  **Scenario**: Some AI routines need to run a heavyweight pathfinding coroutine when a `PlayerMoved` event occurs. Other systems can listen normally.

  ```csharp
  public struct PlayerMoved
  {
      public int PlayerId;
      public Vector3 OldPosition;
      public Vector3 NewPosition;
  }

  public class PathfindingAI : MonoBehaviour
  {
      private void OnEnable()
      {
          // Subscribe with isCoroutine = true so that OnPlayerMoved can start 
          // a coroutine inside this MonoBehaviour context
          EventHelper.Subscribe<PlayerMoved>(
              OnPlayerMoved, 
              EventPriority.Normal, 
              isCoroutine: true, 
              coroutineRunner: this
          );
      }

      private IEnumerator OnPlayerMoved(PlayerMoved data)
      {
          // Simulate a long pathfinding calculation
          yield return StartCoroutine(ComputePath(data.OldPosition, data.NewPosition));
          // Once complete, do something with the path
      }

      private IEnumerator ComputePath(Vector3 start, Vector3 end)
      {
          // Placeholder for actual pathfinding logic
          yield return new WaitForSeconds(1f);
          Debug.Log("Path computed.");
      }

      private void OnDisable()
      {
          EventHelper.Unsubscribe<PlayerMoved>(OnPlayerMoved);
      }
  }

  public class DebugLogger : MonoBehaviour
  {
      private void Awake()
      {
          // Normal, synchronous subscription
          EventHelper.Subscribe<PlayerMoved>(data => {
              Debug.Log($"PlayerMoved: {data.PlayerId} from {data.OldPosition} to {data.NewPosition}");
          });
      }
  }

  // Somewhere in code when player moves:
  // EventHelper.Trigger(new PlayerMoved { PlayerId = 5, OldPosition = lastPos, NewPosition = currentPos });
  ```

  * `PathfindingAI.OnPlayerMoved` will run as a coroutine, perform a “ComputePath” coroutine for 1 second, then continue.
  * Meanwhile, `DebugLogger` will log synchronously before or after (depending on priority) the pathfinding logic begins.

---

#### 2.2.4 `Unsubscribe<T>(Action<T> handler)`

* **Purpose**: Remove all wrappers whose `Handler` exactly equals the provided `handler` delegate, so that it no longer receives events of type `T`.

* **Behavior**:

  1. Lock `_lock`.
  2. Look up `List<EventHandlerWrapperBase> handlers` for `typeof(T)`. If not found, return.
  3. `handlers.RemoveAll(h => (h.Handler as Action<T>)?.Equals(handler) == true);`
  4. Unlock.

* **Realistic Example**
  **Scenario**: At runtime, you want to disable an on-screen logging system so that it no longer logs any `DamageTaken` events.

  ```csharp
  public struct DamageTaken
  {
      public int EntityId;
      public int Amount;
  }

  public class DamageLogger : MonoBehaviour
  {
      private Action<DamageTaken> _handler;

      private void Awake()
      {
          _handler = data => Debug.Log($"Entity {data.EntityId} took {data.Amount} damage.");
          EventHelper.Subscribe<DamageTaken>(_handler, EventPriority.Low);
      }

      public void DisableLogging()
      {
          EventHelper.Unsubscribe<DamageTaken>(_handler);
      }
  }

  // At runtime, if DisableLogging() is called, future DamageTaken events
  // will not invoke the logging callback.
  ```

---

#### 2.2.5 `UnsubscribeScope(object scope)`

* **Purpose**: Given an arbitrary `scope` (for example, a `MonoBehaviour` or any key object), remove *all* event subscriptions that were registered via `SubscribeScoped<T>(scope, ...)`.

* **Behavior**:

  1. Lock `_lock`.
  2. If `_scopedHandlers.TryGetValue(scope, out handlers)`, iterate through each `(Type type, EventHandlerWrapperBase wrapper)` in that list:

     * Retrieve `wrapper.Handler` as a `Delegate` (`del`).
     * Use reflection to call `EventHelper.Unsubscribe<T>(del)`:

       1. `MethodInfo method = typeof(EventHelper).GetMethod(nameof(Unsubscribe));`
       2. `MethodInfo genericMethod = method.MakeGenericMethod(type);`
       3. `genericMethod.Invoke(null, new object[] { del });`
  3. Remove the entry for `scope` from `_scopedHandlers`.
  4. Unlock.

* **Realistic Example**
  **Scenario**: You have a `DialogueWindow` that subscribes to several “UIChoiceSelected” and “DialogueLineComplete” events. When the window is closed, you want to remove all its subscriptions in one call:

  ```csharp
  public class DialogueWindow : MonoBehaviour
  {
      private void OnEnable()
      {
          EventHelper.SubscribeScoped<UIChoiceSelected>(this, OnChoiceSelected, EventPriority.Normal);
          EventHelper.SubscribeScoped<DialogueLineComplete>(this, OnLineComplete, EventPriority.Normal);
      }

      private void OnDisable()
      {
          // Automatically remove both subscriptions
          EventHelper.UnsubscribeScope(this);
      }

      private void OnChoiceSelected(UIChoiceSelected data)
      {
          // Handle user choice
      }

      private void OnLineComplete(DialogueLineComplete data)
      {
          // Display next line
      }
  }
  ```

  * By calling `UnsubscribeScope(this)`, both `OnChoiceSelected` and `OnLineComplete` handlers are removed automatically.

---

#### 2.2.6 `TriggerChainedEvent(IChainedEvent firstEvent)`

* **Purpose**: Start a chain of coroutine-based events in sequence (e.g., UI pop-ups opening in a specific order).

* **Behavior**:

  1. Lock `_lock`.
  2. Enqueue the initial `IChainedEvent` instance into `_chainedEvents`.
  3. Call `_completionTracker.AddEvent(firstEvent)` so that the tracker knows to watch for its completion.
  4. Unlock.

* **Note**: To actually process the queue, you must call `EventHelper.ProcessEvents()` each frame (for example, from a `MonoBehaviour.Update()` method). Only once the current top `IChainedEvent`’s `IsComplete == true` will it be dequeued and its `NextEvent` (if any) enqueued.

* **Realistic Example**
  **Scenario**: You want to show two modal UI panels in sequence: first “LoadingScreen,” then “MainMenu.” Each panel has to finish its opening animation (a coroutine) before the next panel is shown.

  ```csharp
  // Define a concrete chained UI event
  public class OpeningPanelEvent : UIEvent
  {
      private readonly GameObject _panelPrefab;
      private GameObject _instance;

      public OpeningPanelEvent(GameObject panelPrefab, MonoBehaviour coroutineRunner) : base(coroutineRunner)
      {
          _panelPrefab = panelPrefab;
      }

      public override IEnumerator ProcessEvent()
      {
          try
          {
              // Instantiate the panel and wait for 1s to simulate opening animation
              _instance = GameObject.Instantiate(_panelPrefab);
              yield return new WaitForSeconds(1f);
              // After opening, mark complete
              Complete();
          }
          catch (Exception ex)
          {
              SetError(ex);
          }
      }
  }

  public class UIManager : MonoBehaviour
  {
      [SerializeField] private GameObject _loadingScreenPrefab;
      [SerializeField] private GameObject _mainMenuPrefab;

      private void Update()
      {
          // Must call ProcessEvents every frame
          EventHelper.ProcessEvents();
      }

      public void ShowUISequence()
      {
          // Create first event (loading), then chain MainMenu
          var loadingEvent = new OpeningPanelEvent(_loadingScreenPrefab, this);
          var mainMenuEvent = new OpeningPanelEvent(_mainMenuPrefab, this);
          loadingEvent.NextEvent = mainMenuEvent;

          // Finally, trigger the chain
          EventHelper.TriggerChainedEvent(loadingEvent);
      }
  }
  ```

  * When `ShowUISequence()` is called, `loadingEvent` is enqueued.
  * In `Update()`, `EventHelper.ProcessEvents()` sees `loadingEvent.IsComplete == false`, so it calls `StartCoroutine(loadingEvent.ProcessEvent())`.
  * After 1 second, `Complete()` is called inside `loadingEvent`, so next frame `ProcessEvents()` will see `IsComplete == true`, dequeue it, then enqueue `mainMenuEvent`.
  * Then it runs `mainMenuEvent.ProcessEvent()`—so the two screens appear in strict order.

---

#### 2.2.7 `ProcessEvents()`

* **Purpose**: Should be called each frame (e.g. in a central `MonoBehaviour.Update()`) to drive chained events from `_chainedEvents`.
* **Behavior**:

  1. While `_chainedEvents.Count > 0`, peek at the first `IChainedEvent evt`.
  2. If `!evt.IsComplete`, and `evt is UIEvent uiEvent` with `uiEvent.CoroutineRunner != null`, call `StartCoroutine(evt.ProcessEvent())`. Then break out of the loop (wait until next frame to check completion again).
  3. If `evt.IsComplete == true`, dequeue it; if `evt.NextEvent != null`, enqueue `evt.NextEvent` and call `_completionTracker.AddEvent(evt.NextEvent)`. Loop again to check the next queued event.
  4. After processing chaining, call `_completionTracker.Update()` to purge completed events and record errors if any.

---

## 3. Wrappers, Priorities & Chaining Support

This second script provides the supporting types and classes that make the first‐script’s `EventHelper` system work: enum for priorities, abstract base classes, wrappers, interfaces for chaining, a base `UIEvent`, and a completion tracker.

```csharp
namespace UnityHelperSDK
{
    /// <summary>
    /// Priority levels for event handlers
    /// </summary>
    public enum EventPriority
    {
        Critical = 0,
        High = 1,
        Normal = 2,
        Low = 3,
        Background = 4
    }

    /// <summary>
    /// Base class for event handler wrappers
    /// </summary>
    public abstract class EventHandlerWrapperBase
    {
        public EventPriority Priority { get; }
        public bool IsCoroutine { get; }
        public MonoBehaviour CoroutineRunner { get; }
        public object Tag { get; }
        public abstract Type EventType { get; }
        public abstract Delegate Handler { get; }

        protected EventHandlerWrapperBase(EventPriority priority, bool isCoroutine, MonoBehaviour coroutineRunner, object tag)
        {
            Priority = priority;
            IsCoroutine = isCoroutine;
            CoroutineRunner = coroutineRunner;
            Tag = tag;
        }
    }

    /// <summary>
    /// Wrapper for event handlers that includes priority and metadata
    /// </summary>
    public class EventHandlerWrapper<T> : EventHandlerWrapperBase where T : struct
    {
        private readonly Action<T> _handler;
        public override Type EventType => typeof(T);
        public override Delegate Handler => _handler;

        public EventHandlerWrapper(
            Action<T> handler,
            EventPriority priority = EventPriority.Normal,
            bool isCoroutine = false,
            MonoBehaviour coroutineRunner = null,
            object tag = null
        ) : base(priority, isCoroutine, coroutineRunner, tag)
        {
            _handler = handler;
        }
    }

    /// <summary>
    /// Interface for events that can be chained together
    /// </summary>
    public interface IChainedEvent
    {
        IEnumerator ProcessEvent();
        bool IsComplete { get; }
        bool HasError { get; }
        Exception Error { get; }
        IChainedEvent NextEvent { get; set; }
    }

    /// <summary>
    /// Base class for UI events that need to be processed in order
    /// </summary>
    public abstract class UIEvent : IChainedEvent
    {
        public MonoBehaviour CoroutineRunner { get; protected set; }
        public IChainedEvent NextEvent { get; set; }
        public bool IsComplete { get; protected set; }
        public bool HasError { get; protected set; }
        public Exception Error { get; protected set; }

        protected UIEvent(MonoBehaviour coroutineRunner = null)
        {
            CoroutineRunner = coroutineRunner;
        }

        public abstract IEnumerator ProcessEvent();

        protected void SetError(Exception ex)
        {
            HasError = true;
            Error = ex;
            IsComplete = true;
        }

        protected void Complete()
        {
            IsComplete = true;
        }
    }

    /// <summary>
    /// Event completion tracker for coroutine-based events
    /// </summary>
    public class EventCompletionTracker
    {
        private readonly List<IChainedEvent> _pendingEvents = new List<IChainedEvent>();
        private readonly HashSet<IChainedEvent> _completedEvents = new HashSet<IChainedEvent>();

        public bool IsComplete => _pendingEvents.Count == 0;
        public bool HasErrors => _completedEvents.Any(e => e.HasError);

        public void AddEvent(IChainedEvent evt)
        {
            _pendingEvents.Add(evt);
        }

        public void Update()
        {
            _pendingEvents.RemoveAll(evt =>
            {
                if (evt.IsComplete)
                {
                    _completedEvents.Add(evt);
                    return true;
                }
                return false;
            });
        }

        public void Clear()
        {
            _pendingEvents.Clear();
            _completedEvents.Clear();
        }

        public IEnumerable<Exception> GetErrors()
        {
            return _completedEvents.Where(e => e.HasError).Select(e => e.Error);
        }
    }

    /// <summary>
    /// Interface for objects that want automatic event cleanup
    /// </summary>
    public interface IEventHandler
    {
        void OnEventCleanup();
    }
}
```

### 3.1 `EventPriority` (enum)

Defines ascending integer values so that `0 = Critical`, `1 = High`, `2 = Normal`, `3 = Low`, `4 = Background`. The smaller the enum value, the higher the priority when inserting into `_eventHandlers[type]`.

* **Use Case**:

  * If you have two handlers for `DamageTaken`, one logs to the console (`Low`), and one updates the UI (`Critical`), you would register UI first:

    ```csharp
    EventHelper.Subscribe<DamageTaken>(UpdateUIOnDamage, EventPriority.Critical);
    EventHelper.Subscribe<DamageTaken>(LogDamageToConsole, EventPriority.Low);
    ```

  * Invoking `EventHelper.Trigger(new DamageTaken(...))` will call `UpdateUIOnDamage` before `LogDamageToConsole`.

---

### 3.2 `EventHandlerWrapperBase` & `EventHandlerWrapper<T>`

* **`EventHandlerWrapperBase`** (abstract) holds:

  * `EventPriority Priority`
  * `bool IsCoroutine`
  * `MonoBehaviour CoroutineRunner`
  * `object Tag` (arbitrary user data)
  * Abstract `Type EventType { get; }` (e.g., `typeof(T)`)
  * Abstract `Delegate Handler { get; }` (the actual `Action<T>` delegate)

* **`EventHandlerWrapper<T>`** extends `EventHandlerWrapperBase` where `T : struct`:

  * Stores `private readonly Action<T> _handler;`
  * Implements `public override Type EventType => typeof(T);`
  * Implements `public override Delegate Handler => _handler;`
  * Constructor takes `(Action<T> handler, EventPriority priority, bool isCoroutine, MonoBehaviour coroutineRunner, object tag)` and passes metadata to the base class.

* **What This Enables**:

  * By wrapping each `Action<T>` in a `EventHandlerWrapper<T>`, the system can store extra metadata (priority, whether it should be called as a coroutine, etc.) in `_eventHandlers[typeof(T)]`.
  * When retrieving `wrapper.Handler`, it can be cast back to `Action<T>` at invocation time.

---

### 3.3 `IChainedEvent` & `UIEvent`

#### 3.3.1 `IChainedEvent` (interface)

Defines a contract for coroutine-based events that can be chained in sequence:

```csharp
public interface IChainedEvent
{
    IEnumerator ProcessEvent();
    bool IsComplete { get; }
    bool HasError { get; }
    Exception Error { get; }
    IChainedEvent NextEvent { get; set; }
}
```

* **`ProcessEvent()`**:

  * Should be implemented as an `IEnumerator` coroutine. Inside it, perform all necessary asynchronous steps, then call `Complete()` or `SetError(ex)` to mark completion.
* **Properties**:

  * `bool IsComplete { get; }` — indicates whether this event has finished (successfully or with an error).
  * `bool HasError { get; }` — if an exception was caught during processing.
  * `Exception Error { get; }` — the exception object if `HasError == true`.
  * `IChainedEvent NextEvent { get; set; }` — optional link to the next event in the chain.

#### 3.3.2 `UIEvent` (abstract base)

A concrete partial implementation of `IChainedEvent` that provides:

* **Constructor**:

  ```csharp
  protected UIEvent(MonoBehaviour coroutineRunner = null)
  {
      CoroutineRunner = coroutineRunner;
  }
  ```

  Stores a `CoroutineRunner` on which you can call `StartCoroutine`.

* **Protected Methods**:

  * `void SetError(Exception ex)`: Mark `HasError = true; Error = ex; IsComplete = true;`
  * `void Complete()`: Mark `IsComplete = true;`

* **Fields/Properties**:

  * `public MonoBehaviour CoroutineRunner { get; protected set; }`
  * `public bool IsComplete { get; protected set; }`
  * `public bool HasError { get; protected set; }`
  * `public Exception Error { get; protected set; }`
  * `public IChainedEvent NextEvent { get; set; }`

* **Abstract**:

  ```csharp
  public abstract IEnumerator ProcessEvent();
  ```

  Derived classes implement this coroutine. They should wrap their internal logic in a `try/catch` and call `Complete()` on success or `SetError(ex)` on failure.

* **Realistic Example** (expanding on prior section):

  ```csharp
  public class ShowPopupEvent : UIEvent
  {
      private readonly GameObject _popupPrefab;

      public ShowPopupEvent(GameObject prefab, MonoBehaviour runner) : base(runner)
      {
          _popupPrefab = prefab;
      }

      public override IEnumerator ProcessEvent()
      {
          GameObject popup = null;
          try
          {
              popup = UnityEngine.Object.Instantiate(_popupPrefab);
              // Wait until the popup’s own ShowAnimation coroutine completes
              yield return popup.GetComponent<PopupController>().ShowAnimation();
              // Mark this event as complete
              Complete();
          }
          catch (Exception ex)
          {
              if (popup != null)
                  UnityEngine.Object.Destroy(popup);
              SetError(ex);
          }
      }
  }
  ```

---

### 3.4 `EventCompletionTracker`

```csharp
public class EventCompletionTracker
{
    private readonly List<IChainedEvent> _pendingEvents = new List<IChainedEvent>();
    private readonly HashSet<IChainedEvent> _completedEvents = new HashSet<IChainedEvent>();

    public bool IsComplete => _pendingEvents.Count == 0;
    public bool HasErrors => _completedEvents.Any(e => e.HasError);

    public void AddEvent(IChainedEvent evt)
    {
        _pendingEvents.Add(evt);
    }

    public void Update()
    {
        // Remove any events where IsComplete == true, add them to _completedEvents
        _pendingEvents.RemoveAll(evt =>
        {
            if (evt.IsComplete)
            {
                _completedEvents.Add(evt);
                return true;
            }
            return false;
        });
    }

    public void Clear()
    {
        _pendingEvents.Clear();
        _completedEvents.Clear();
    }

    public IEnumerable<Exception> GetErrors()
    {
        return _completedEvents.Where(e => e.HasError).Select(e => e.Error);
    }
}
```

* **`AddEvent(IChainedEvent evt)`**: Call whenever you enqueue a new chained event (both in `TriggerChainedEvent` and when adding `NextEvent` to the queue).

* **`Update()`**: Called each frame (from `EventHelper.ProcessEvents()`), iterates through `_pendingEvents`, and if `evt.IsComplete == true`, moves it to `_completedEvents` and removes from `_pendingEvents`.

* **`IsComplete`**: `true` if no more pending events remain.

* **`HasErrors` / `GetErrors()`**: Check if any completed event had `HasError == true`, and retrieve their exceptions.

* **Realistic Example**
  **Scenario**: A manager wants to know when a sequence of loading steps (each a chained event) finishes or if any error occurred:

  ```csharp
  public class LoadingSequenceManager : MonoBehaviour
  {
      private void Update()
      {
          EventHelper.ProcessEvents();

          if (_tracking != null && _tracking.IsComplete)
          {
              if (_tracking.HasErrors)
              {
                  foreach (var ex in _tracking.GetErrors())
                      Debug.LogError($"Error in chained event: {ex}");
              }
              else
              {
                  Debug.Log("Loading sequence completed successfully.");
              }
              // Clear tracker so we only log this once
              _tracking.Clear();
              _tracking = null;
          }
      }

      private EventCompletionTracker _tracking;

      public void StartLoadingSequence()
      {
          // Create multiple chained events, e.g., LoadResourcesEvent -> InitializeUIEvent -> PlayIntroMusicEvent
          var loadResources = new LoadResourcesEvent(this);
          var initUI = new InitializeUIEvent(this);
          var playMusic = new PlayIntroMusicEvent(this);

          loadResources.NextEvent = initUI;
          initUI.NextEvent = playMusic;

          EventHelper.TriggerChainedEvent(loadResources);
          _tracking = new EventCompletionTracker();
          _tracking.AddEvent(loadResources);
          _tracking.AddEvent(initUI);
          _tracking.AddEvent(playMusic);
      }
  }
  ```

  * After calling `TriggerChainedEvent(loadResources)`, all three events are tracked in `_tracking`.
  * In `Update()`, once all three report `IsComplete == true`, `HasErrors` and `GetErrors()` can be used to respond accordingly.

---

### 3.5 `IEventHandler` (Interface)

```csharp
public interface IEventHandler
{
    void OnEventCleanup();
}
```

* **Purpose**:

  * If a component implements `IEventHandler`, and is on the same GameObject as an `EventHandlerComponent`, then when `EventHandlerComponent.OnDestroy()` is called (i.e., the GameObject is destroyed), after unsubscribing all event handlers, it invokes `OnEventCleanup()` on each `IEventHandler` on that GameObject.
* **Realistic Example**
  Continuing the **ScoreManager** example from section 1.2:

  ```csharp
  public class ScoreManager : MonoBehaviour, IEventHandler
  {
      private EventHandlerComponent _eventComponent;

      private void Awake()
      {
          _eventComponent = gameObject.AddComponent<EventHandlerComponent>();
          _eventComponent.AddHandler<PlayerScored>(OnPlayerScored);
      }

      private void OnPlayerScored(PlayerScored data)
      {
          // Update internal state/UI
      }

      public void OnEventCleanup()
      {
          // This method is invoked automatically by EventHandlerComponent.OnDestroy()
          Debug.Log("ScoreManager: Cleaned up event subscriptions.");
      }
  }
  ```

---

## 4. Putting It All Together: Typical Workflow

1. **Attach `EventHandlerComponent` to any GameObject that needs automatic subscribe/unsubscribe.**

   * Then call `AddHandler<T>(Action<T>)` on that component to register event callbacks.
   * Alternatively, you can call the static `EventHelper.Subscribe<T>(...)` directly if you prefer manual unsubscription.

2. **Triggering Events:**

   * Anywhere in code, call `EventHelper.Trigger(new MyEventType { ... })`.
   * All registered handlers for `MyEventType` (including coroutines, in priority order) will be invoked.

3. **Unsubscribing:**

   * If you used `AddHandler<T>` on an `EventHandlerComponent`, you don’t need to manually unsubscribe—`OnDestroy()` does it for you.
   * Otherwise, call `EventHelper.Unsubscribe<T>(myHandler)` to remove a specific handler.
   * If you used a “scoped” subscription (`SubscribeScoped<T>(scope, ...)`), you can remove all handlers registered under that `scope` by calling `EventHelper.UnsubscribeScope(scope)`.

4. **Chained Events & Processing:**

   * When you want a series of coroutine-based events to execute in strict order (e.g., UI modals opening sequentially), construct each as a `UIEvent` (or other `IChainedEvent`) and set their `NextEvent` fields accordingly.
   * Call `EventHelper.TriggerChainedEvent(firstEvent)` to begin the chain.
   * In a central `Update()` (e.g., in a “Manager” GameObject), call `EventHelper.ProcessEvents()` every frame so that the event bus can start each event’s coroutine and detect completion.

5. **Completion Tracking:**

   * If you need to know when an entire chain is done (or if any event in it failed), use an `EventCompletionTracker`. Add every event in the chain to it via `AddEvent(...)`, then in `Update()`, call `tracker.Update()`. Once `tracker.IsComplete == true`, handle success or failure via `tracker.HasErrors` / `tracker.GetErrors()`.

---

## 5. Summary of Key Methods & Real-World Snippets

### 5.1 Event Registration & Triggering

| Method                                                                                                                                                                | Purpose                                                                                                                                                                    | Example Use Case                                                                                                                                                                                                                                                  |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`EventHelper.Subscribe<T>(Action<T> handler, EventPriority priority = Normal, bool isCoroutine = false, MonoBehaviour coroutineRunner = null, object tag = null)`** | Register an `Action<T>` callback for event type `T`, with priority ordering. If `isCoroutine` is true, run handler via `StartCoroutine` on the provided `coroutineRunner`. | *AI system: `EventHelper.Subscribe<PlayerSpotted>(OnPlayerSpotted, EventPriority.High)` to respond immediately when player is detected.*<br>*Analytics system: `EventHelper.Subscribe<PlayerSpotted>(LogSpotted, EventPriority.Low)` to log after primary logic.* |
| **`EventHelper.SubscribeScoped<T>(MonoBehaviour scope, Action<T> handler, EventPriority priority = Normal, bool isCoroutine = false, object tag = null)`**            | Subscribe `handler` under a `scope` so you can call `EventHelper.UnsubscribeScope(scope)` later to remove all related subscriptions.                                       | *UI panel: in `OnEnable()`, call `UserPanel.SubscribeScoped<UserLoggedIn>(this, OnUserLogin)`. In `OnDisable()`, call `EventHelper.UnsubscribeScope(this)`.*                                                                                                      |
| **`EventHelper.Trigger<T>(T eventData)`**                                                                                                                             | Broadcast event to all registered handlers (in ascending `Priority`). If handler was registered with `isCoroutine = true`, invoked via coroutine.                          | *When a player picks up an item: `EventHelper.Trigger(new ItemPickedUp { ItemId = 12, PlayerId = 3 });` triggers all subscribed `Action<ItemPickedUp>`.*                                                                                                          |
| **`EventHelper.Unsubscribe<T>(Action<T> handler)`**                                                                                                                   | Remove a previously registered handler so that it no longer receives event notifications.                                                                                  | *Pause menu logic: `EventHelper.Unsubscribe<GamePaused>(OnGamePaused)`. Handler `OnGamePaused` no longer invoked.*                                                                                                                                                |
| **`EventHelper.UnsubscribeScope(object scope)`**                                                                                                                      | Remove **all** event handlers that were registered via `SubscribeScoped` under `scope`.                                                                                    | *When closing a level, call `EventHelper.UnsubscribeScope(myLevelManager)` to clear all its UI and gameplay event listeners.*                                                                                                                                     |

### 5.2 Coroutine-Based & Chained Events

| Method                                                          | Purpose                                                                                                                                           | Example Use Case                                                                                                                                               |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`EventHelper.TriggerChainedEvent(IChainedEvent firstEvent)`** | Begin a chain of coroutine-based events. Each event’s `NextEvent` will be queued only after the previous one completes.                           | *Show `ShowPopupEvent(A)`, then `ShowPopupEvent(B)`, then `ShowPopupEvent(C)` in order—each waits for the prior popup’s “Show animation” coroutine to finish.* |
| **`EventHelper.ProcessEvents()`**                               | Called each frame to “drive” chained events. Starts `ProcessEvent()` coroutine on the head event if it is not complete, or dequeues if completed. | *In a central `MonoBehaviour.Update()`, call `EventHelper.ProcessEvents()` so that chained UI events run in order over multiple frames.*                       |

### 5.3 Wrappers & Metadata

| Class / Interface               | Purpose                                                                                                                                             | Example Use Case                                                                                                                                                                            |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`EventHandlerWrapper<T>`**    | Encapsulates an `Action<T>` plus `Priority`, `IsCoroutine`, `CoroutineRunner`, and `Tag` metadata.                                                  | *When subscribing an event handler that should run as a coroutine on a specific MonoBehaviour, code does `new EventHandlerWrapper<T>(handler, EventPriority.Normal, true, this, someTag)`.* |
| **`EventPriority`** (enum)      | Defines ordering for handlers: `Critical = 0`, `High = 1`, `Normal = 2`, `Low = 3`, `Background = 4`.                                               | *If a “DamageTaken” event has multiple listeners (UI update vs. status effect), you can ensure UI update runs at `Critical` priority, then status effect logic at `Low`.*                   |
| **`EventCompletionTracker`**    | Tracks a list of `IChainedEvent` instances, cleans them up once they report `IsComplete == true`, and records any errors.                           | *While loading assets in sequence, the manager adds each load step event to the tracker. Once all are complete, it knows loading is done.*                                                  |
| **`IChainedEvent`** (interface) | Defines a coroutine-based event with `ProcessEvent()`, `IsComplete`, `HasError`, `Error`, and `NextEvent`.                                          | *Any custom UI or resource-loading event can implement `IChainedEvent` so it can be chained in a queue.*                                                                                    |
| **`UIEvent`** (abstract)        | Base implementation of `IChainedEvent` that provides `CoroutineRunner`, `Complete()`, and `SetError()`. Subclasses implement `ProcessEvent()`.      | *To show a series of tutorial popups, each popup event extends `UIEvent` and uses `Complete()` once it finishes its fade-in/fade-out.*                                                      |
| **`IEventHandler`** (interface) | Implemented by any MonoBehaviour that wants a final notification `OnEventCleanup()` when its `EventHandlerComponent` is destroyed and unsubscribes. | *A manager script that logs “cleaned up events” or needs to free additional resources when event subscriptions are removed can implement this interface.*                                   |

---

## 6. Additional Real-World Usage Patterns

### 6.1 Deferred Initialization of Subscribers

Sometimes you want a system to subscribe only once a certain resource is ready. For example, a `Minimap` script should subscribe to `PlayerMoved` only after the player’s `Transform` is assigned:

```csharp
public class Minimap : MonoBehaviour, IEventHandler
{
    private EventHandlerComponent _eventComponent;
    private Transform _playerTransform;

    private void Start()
    {
        // Defer subscription until player is found
        StartCoroutine(WaitForPlayerThenSubscribe());
    }

    private IEnumerator WaitForPlayerThenSubscribe()
    {
        while (_playerTransform == null)
        {
            var playerGO = GameObject.FindWithTag("Player");
            if (playerGO != null)
                _playerTransform = playerGO.transform;
            yield return null;
        }

        _eventComponent = gameObject.AddComponent<EventHandlerComponent>();
        _eventComponent.AddHandler<PlayerMoved>(OnPlayerMoved);
    }

    private void OnPlayerMoved(PlayerMoved data)
    {
        // Update the minimap blip for this player
    }

    public void OnEventCleanup()
    {
        // Called after unsubscribing
    }
}
```

* The `Minimap` waits until it can locate the player in the scene. Once found, it adds `EventHandlerComponent` (if not already present) and subscribes to `PlayerMoved`.
* If the `Minimap` GameObject is destroyed before the player appears, no subscriptions occur.

---

### 6.2 Tagging & Filtering by Tag

Because `EventHandlerWrapperBase` has a `Tag` property, you can optionally use it to filter handlers at runtime. For example, you could create a custom `TriggerWithTag<T>(T eventData, object senderTag)` method that only invokes handlers if `wrapper.Tag == senderTag` (not implemented by default, but possible). This can be useful if multiple “streams” of the same event type exist, differentiated by tag.

#### Illustrative Example (Pseudocode)

```csharp
public static void TriggerWithTag<T>(T eventData, object senderTag) where T : struct
{
    List<EventHandlerWrapperBase> handlersCopy;
    lock(_lock)
    {
        if (!_eventHandlers.TryGetValue(typeof(T), out var handlersList))
            return;
        handlersCopy = new List<EventHandlerWrapperBase>(handlersList);
    }

    foreach (var wrapper in handlersCopy)
    {
        if (wrapper.Tag != null && !wrapper.Tag.Equals(senderTag))
            continue; // Skip if tags don’t match
        // ...invoke as usual...
    }
}

// Usage:
EventHelper.Subscribe<PlayerScored>(OnPlayer1Scored, EventPriority.Normal, false, this, tag: "Player1");
EventHelper.Subscribe<PlayerScored>(OnPlayer2Scored, EventPriority.Normal, false, this, tag: "Player2");

// Then:
EventHelper.TriggerWithTag(new PlayerScored(1, 5), senderTag: "Player1");
// Only OnPlayer1Scored is invoked; OnPlayer2Scored is skipped.
```

---

### 6.3 Combining Scoped & Unscoped Subscriptions

You may choose to mix scoped and manual (unscoped) subscriptions:

* **Scoped** subscriptions automatically remove themselves when you call `UnsubscribeScope(scope)`.
* **Manual (unscoped)** subscriptions remain until you explicitly call `Unsubscribe<T>(handler)`.

#### Example

```csharp
public class MixedSubscriptions : MonoBehaviour
{
    private Action<EnemyDefeated> _logHandler;

    private void Awake()
    {
        // Manual subscription: must explicitly call Unsubscribe later
        _logHandler = data => Debug.Log($"Enemy {data.EnemyId} defeated for {data.PlayerId}");
        EventHelper.Subscribe<EnemyDefeated>(_logHandler, EventPriority.Low);

        // Scoped subscription: will be removed when UnsubscribeScope(this) is called
        EventHelper.SubscribeScoped<EnemyDefeated>(this, OnEnemyDefeated, EventPriority.Normal);
    }

    private void OnEnemyDefeated(EnemyDefeated data)
    {
        // Award XP to player
    }

    private void OnDestroy()
    {
        // Only the scoped subscription is removed automatically
        EventHelper.UnsubscribeScope(this);

        // If you also want to remove the manual subscription:
        EventHelper.Unsubscribe<EnemyDefeated>(_logHandler);
    }
}
```

---

## 7. Best Practices & Tips

1. **Always Unsubscribe or Use Scope**

   * Failure to call `Unsubscribe<T>` (or never calling `UnsubscribeScope(scope)`) can lead to memory leaks or “null reference” calls when handlers point to destroyed objects.
   * The easiest pattern is to attach an `EventHandlerComponent` and call `AddHandler<T>(...)` (which auto‐unsubscribes in `OnDestroy()`).

2. **Choose Appropriate Priority**

   * Use `EventPriority.Critical` (0) only for truly urgent logic (e.g., input blocking, cancellation).
   * Use `High` (1) for systems that must respond before most listeners (e.g., AI reactions).
   * Use `Normal` (2) for default business logic.
   * Use `Low` (3) or `Background` (4) for logging, analytics, or non-critical tasks.

3. **Use Coroutine Subscriptions Sparingly**

   * If a handler performs lengthy work, mark `isCoroutine = true` and supply a `coroutineRunner` (usually `this` MonoBehaviour). The method signature must be `IEnumerator HandlerName(T data)`.
   * Remember that all coroutine handlers are started *in parallel* (one coroutine per handler), but **only** if the handler was registered with `isCoroutine == true` and `CoroutineRunner` is non-null.

4. **Always Call `ProcessEvents()` Each Frame**

   * If you intend to use any chained events (`TriggerChainedEvent()`), you must call `EventHelper.ProcessEvents()` in an `Update()` loop (ideally on a central manager GameObject). Otherwise, queued `IChainedEvent`s will never begin.

5. **Check for Errors in Chained Events**

   * After calling `TriggerChainedEvent`, you can track completion with an `EventCompletionTracker`. Use `tracker.HasErrors` or `tracker.GetErrors()` to respond if one of your `UIEvent` subclasses threw an exception.

6. **Remove Dead Handlers Periodically**

   * `Subscribe<T>` automatically prunes stale wrappers whenever you subscribe. If handlers are destroyed at random times (without unsubscribing), you can also manually call a “cleanup” method (not provided out of the box) to prune dead wrappers:

     ```csharp
     lock (_lock)
     {
         foreach (var kvp in _eventHandlers)
         {
             kvp.Value.RemoveAll(h => h == null || h.Handler == null || h.Handler.Target == null);
         }
     }
     ```
   * This helps keep memory usage low if many objects are created/destroyed but fail to call `Unsubscribe<T>` before destruction.

---

## 8. Summary

The two scripts (EventHandlerComponent + EventHelper and the wrapper/priority/chaining support) combine to form a powerful, flexible event bus designed for Unity:

* **EventBus (EventHelper)**

  * **Subscribe / Unsubscribe** with priority ordering and optional coroutine invocation.
  * **Scoped Subscriptions** so you can batch-unsubscribe all handlers belonging to a specific “scope” object.
  * **Trigger** events by publishing a struct payload; handlers run in ascending priority order.
  * **Coroutine Support**: if you register `isCoroutine = true`, your handler is invoked inside `StartCoroutine(...)`.
  * **Chained Events** (IChainedEvent/UIEvent) enable sequential, coroutine-based workflows like UI pop-ups or staged loading.
  * **Thread Safety**: all mutating operations on `_eventHandlers` and `_scopedHandlers` are wrapped in `lock (_lock)`.

* **EventHandlerComponent**

  * MonoBehaviour that attaches to GameObjects and manages “AddHandler<T>() → EventHelper.Subscribe<T>()” plus automatic unsubscription in `OnDestroy()`.
  * If other components implement `IEventHandler`, calls their `OnEventCleanup()` right after unsubscribing.

* **Supporting Types**

  * **`EventPriority`** allows you to control invocation order.
  * **`EventHandlerWrapperBase` / `EventHandlerWrapper<T>`** store both the `Action<T>` delegate and metadata (priority, coroutine‐flag, runner, tag).
  * **`IChainedEvent`** / **`UIEvent`** + **`EventCompletionTracker`** let you queue and monitor asynchronous sequences of coroutine events.

By following the patterns shown in the examples—such as always unsubscribing or using the scoped pattern, picking appropriate priority levels, and calling `ProcessEvents()` for chained events—you can build robust, maintainable event-driven systems in Unity without worrying about leaks, race conditions, or ordering problems.

---

## 9. Appendix: Full Example Project Outline

Below is an outline combining many of these features into a single “mini‐project” that ties everything together. This is for illustrative purposes only.

### 9.1 Event Definitions

```csharp
public struct PlayerScored
{
    public int PlayerId;
    public int ScoreDelta;
}

public struct LevelLoaded
{
    public string LevelName;
}

public struct UISequenceComplete { }
```

### 9.2 Event Publisher

```csharp
public class GameManager : MonoBehaviour
{
    [SerializeField] private GameObject _mainMenuPrefab;
    [SerializeField] private GameObject _hudPrefab;
    [SerializeField] private GameObject _gameplayUI;

    private void Awake()
    {
        // 1. Trigger a chained UI sequence: Show main menu, then HUD once level is loaded
        var showMainMenu = new ShowUIPanelEvent(_mainMenuPrefab, this);
        var showHUD = new ShowUIPanelEvent(_hudPrefab, this);
        showMainMenu.NextEvent = showHUD;

        EventHelper.TriggerChainedEvent(showMainMenu);
    }

    public void OnLevelLoaded(string levelName)
    {
        // Notify listeners that the level load is complete
        EventHelper.Trigger(new LevelLoaded { LevelName = levelName });
    }

    public void ScorePlayer(int playerId, int delta)
    {
        EventHelper.Trigger(new PlayerScored { PlayerId = playerId, ScoreDelta = delta });
    }

    private void Update()
    {
        // Drive chained UI events
        EventHelper.ProcessEvents();
    }
}
```

### 9.3 UI Panel Opening Event

```csharp
public class ShowUIPanelEvent : UIEvent
{
    private readonly GameObject _panelPrefab;
    private GameObject _panelInstance;

    public ShowUIPanelEvent(GameObject panelPrefab, MonoBehaviour runner) : base(runner)
    {
        _panelPrefab = panelPrefab;
    }

    public override IEnumerator ProcessEvent()
    {
        try
        {
            // Instantiate and wait 0.5s to simulate fade-in
            _panelInstance = GameObject.Instantiate(_panelPrefab);
            yield return new WaitForSeconds(0.5f);
            Complete();
        }
        catch (Exception ex)
        {
            if (_panelInstance != null)
                GameObject.Destroy(_panelInstance);
            SetError(ex);
        }
    }
}
```

### 9.4 HUD Manager (Subscriber)

```csharp
public class HUDManager : MonoBehaviour, IEventHandler
{
    private EventHandlerComponent _eventComponent;

    private void Awake()
    {
        _eventComponent = gameObject.AddComponent<EventHandlerComponent>();

        // Subscribe to LevelLoaded so we can initialize HUD
        _eventComponent.AddHandler<LevelLoaded>(OnLevelLoaded);
        // Subscribe to PlayerScored to update score display
        _eventComponent.AddHandler<PlayerScored>(OnPlayerScored);
    }

    private void OnLevelLoaded(LevelLoaded data)
    {
        Debug.Log($"HUDManager: Level '{data.LevelName}' loaded, showing HUD");
        // e.g., enable HUD UI
    }

    private void OnPlayerScored(PlayerScored data)
    {
        Debug.Log($"HUDManager: Displaying new score for Player {data.PlayerId} (+{data.ScoreDelta})");
        // Update on-screen score text
    }

    public void OnEventCleanup()
    {
        Debug.Log("HUDManager: EventHandlerComponent is being destroyed, cleaning up.");
    }
}
```

### 9.5 Score Display (Scoped Subscriber)

```csharp
public class ScoreDisplay : MonoBehaviour
{
    private void OnEnable()
    {
        // SubscribeScoped so that if ScoreDisplay is disabled, 
        // all subscriptions are automatically removed
        EventHelper.SubscribeScoped<PlayerScored>(this, UpdateScore, EventPriority.Normal);
    }

    private void OnDisable()
    {
        // Automatically remove subscriptions added with SubscribeScoped
        EventHelper.UnsubscribeScope(this);
    }

    private void UpdateScore(PlayerScored data)
    {
        // Update only this player’s portion of the UI
    }
}
```

### 9.6 Example of Triggering

```csharp
public class Player : MonoBehaviour
{
    public int Id { get; private set; } = 1;
    private int _score = 0;

    private void Start()
    {
        Id = Random.Range(1, 1000); // example ID
    }

    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            // Simulate scoring
            _score += 10;
            EventHelper.Trigger(new PlayerScored { PlayerId = this.Id, ScoreDelta = 10 });
        }
    }
}
```

---

## 10. Conclusion

This unified event management library:

* **Decouples** producers and consumers of events via strongly typed payload structs (`T : struct`).
* **Maintains order** of handler invocation via `EventPriority`.
* **Supports coroutine handlers** that run on a provided `MonoBehaviour`.
* **Enables scoped subscriptions** so large UI systems or gameplay systems can automatically unregister all their handlers when destroyed.
* **Provides chained event sequences** via `IChainedEvent` and `UIEvent`, processed through `ProcessEvents()`.
* **Tracks completion and errors** of chained events via `EventCompletionTracker`.
* **Offers a simple “add component, call AddHandler” pattern** through `EventHandlerComponent` with automatic cleanup in `OnDestroy()`.

By following the patterns and examples above, you can build complex, robust event-driven architectures in Unity while avoiding common pitfalls such as memory leaks, race conditions, or unpredictable invocation order. Feel free to extend or customize—for example, adding tag-based filtering, dynamic unsubscription filtering, or additional metadata to wrappers—while retaining the core thread-safe, priority-ordered foundation.

Happy coding!
