**Generic State Pattern Base in Unity**

This document details the `IStatePattern<TContext>` interface and `StatePatternBase<TState, TContext>` abstract class, followed by **extensive** real-world Unity examples for each method.

---

## Table of Contents

1. [Overview](#overview)
2. [API Reference](#api-reference)

   * `IStatePattern<TContext>`
   * `StatePatternBase<TState, TContext>`
3. [Real-World Scenarios](#scenarios)

   1. Player Controller: Idle → Running → Jumping → Attacking
   2. Enemy AI: Patrol → Alert → Attack → Flee
   3. UI Window Interaction: Closed → Opening → Open → Closing
   4. Weapon Firing Modes: Single → Burst → Auto
   5. Dialogue Node Navigation: Start → Typing → AwaitChoice → End
   6. Traffic Light: Green → Yellow → Red
   7. Mini-Game Flow: Init → Playing → Paused → Completed
4. [Best Practices & Edge Cases](#best-practices)
5. [Full Code Listing](#appendix)

---

## 1. Overview<a name="overview"></a>

The **State Pattern** encapsulates behaviors and transitions into state objects implementing `IStatePattern<TContext>`. `StatePatternBase<TState, TContext>`:

* Holds the **current** state instance.
* Provides `ChangeState`, invoking `Exit` and `Enter` hooks.
* Calls `HandleInput` then `Update` in `Update()`.
* Guards against redundant transitions and logs errors.

This decouples context logic (e.g., `Player`, `EnemyAI`, UI managers) from state-specific code.

---

## 2. API Reference<a name="api-reference"></a>

### `IStatePattern<TContext>`

Defines the lifecycle methods for a state.

```csharp
public interface IStatePattern<TContext>
{
    void Enter(TContext context);
    void HandleInput(TContext context);
    void Update(TContext context);
    void Exit(TContext context);
}
```

* **Enter**: Called immediately when transitioning into this state.
* **HandleInput**: Process input or events before update logic.
* **Update**: Main per-frame logic.
* **Exit**: Cleanup when leaving state.

---

### `StatePatternBase<TState, TContext>`

Abstract base managing current state and transitions.

```csharp
public abstract class StatePatternBase<TState, TContext> where TState : IStatePattern<TContext>
{
    protected TState _currentState;
    protected TContext _context;

    public StatePatternBase(TContext context) { _context = context; }

    public void ChangeState(TState newState)
    {
        if (_currentState != null && _currentState.Equals(newState))
        {
            Debug.LogWarning("Attempting to change to the same state");
            return;
        }
        try
        {
            _currentState?.Exit(_context);
            _currentState = newState;
            _currentState?.Enter(_context);
        }
        catch (Exception e)
        {
            Debug.LogError($"Error changing state: {e}");
        }
    }

    public void Update()
    {
        _currentState?.HandleInput(_context);
        _currentState?.Update(_context);
    }

    public TState GetCurrentState() => _currentState;
}
```

* **ChangeState**: Safely exits old state and enters new. Prevents duplicate.
* **Update**: Drives `HandleInput` then `Update` on active state each frame.
* **GetCurrentState**: Exposes for debugging or conditional logic.

---

## 3. Real-World Scenarios<a name="scenarios"></a>

### 3.1 Player Controller: Idle → Running → Jumping → Attacking

```csharp
// Context
public class Player : MonoBehaviour
{
    public StatePatternBase<PlayerState, Player> StateManager;
    void Awake()
    {
        StateManager = new PlayerStateManager(this);
    }
    void Update() => StateManager.Update();
}

// Base state
public abstract class PlayerState : IStatePattern<Player>
{
    public virtual void Enter(Player ctx) {}
    public virtual void HandleInput(Player ctx) {}
    public virtual void Update(Player ctx) {}
    public virtual void Exit(Player ctx) {}
}

// Concrete states\public class IdleState : PlayerState
{
    public override void HandleInput(Player ctx)
    {
        if (Input.GetAxis("Horizontal") != 0) ctx.StateManager.ChangeState(new RunningState());
        else if (Input.GetButtonDown("Jump")) ctx.StateManager.ChangeState(new JumpingState());
        else if (Input.GetButtonDown("Fire1")) ctx.StateManager.ChangeState(new AttackingState());
    }
}
// RunningState, JumpingState, AttackingState implement movement and revert to Idle.

// Manager\public class PlayerStateManager : StatePatternBase<PlayerState, Player>
{
    public PlayerStateManager(Player ctx) : base(ctx)
    {
        ChangeState(new IdleState());
    }
}
```

---

### 3.2 Enemy AI: Patrol → Alert → Attack → Flee

```csharp
// Context
public class EnemyAI : MonoBehaviour
{
    public StatePatternBase<EnemyState, EnemyAI> StateManager;
    void Awake()
    {
        StateManager = new EnemyStateManager(this);
    }
    void Update() => StateManager.Update();
}

public abstract class EnemyState : IStatePattern<EnemyAI>
{
    public virtual void Enter(EnemyAI ctx) {}
    public virtual void HandleInput(EnemyAI ctx) {}
    public virtual void Update(EnemyAI ctx) {}
    public virtual void Exit(EnemyAI ctx) {}
}

public class PatrolState : EnemyState
{
    public override void Update(EnemyAI ctx)
    {
        // Move between waypoints
        if (ctx.SeesPlayer()) ctx.StateManager.ChangeState(new AlertState());
    }
}
// AlertState triggers AttackState; AttackState may transition to FleeState if health low.

public class EnemyStateManager : StatePatternBase<EnemyState, EnemyAI>
{
    public EnemyStateManager(EnemyAI ctx) : base(ctx)
    {
        ChangeState(new PatrolState());
    }
}
```

---

### 3.3 UI Window Interaction: Closed → Opening → Open → Closing

```csharp
// Context: UIWindow with CanvasGroup
public class UIWindow : MonoBehaviour
{
    public StatePatternBase<UIState, UIWindow> UIManager;
    void Awake() => UIManager = new UIWindowStateManager(this);
    void Update() => UIManager.Update();
}

public abstract class UIState : IStatePattern<UIWindow> { /* ... */ }

public class ClosedState : UIState
{
    public override void Enter(UIWindow ctx)
    {
        ctx.gameObject.SetActive(false);
    }
}
// OpeningState fades in, then ChangeState(new OpenState());
// ClosingState fades out, then ChangeState(new ClosedState());

public class UIWindowStateManager : StatePatternBase<UIState, UIWindow>
{
    public UIWindowStateManager(UIWindow ctx) : base(ctx)
    {
        ChangeState(new ClosedState());
    }
}
```

---

### 3.4 Weapon Firing Modes: Single → Burst → Auto

```csharp
public class Weapon : MonoBehaviour
{
    public StatePatternBase<FireModeState, Weapon> ModeManager;
    void Awake() => ModeManager = new WeaponModeManager(this);
    void Update()
    {
        ModeManager.Update();
        if (Input.GetKeyDown(KeyCode.V)) // change mode
            ModeManager.ChangeState(new BurstMode());
    }
}

public abstract class FireModeState : IStatePattern<Weapon> { /* ... */ }
public class SingleShotMode : FireModeState { /* on Fire1 press => shoot once */ }
public class BurstMode : FireModeState { /* shoot 3-round burst */ }
public class AutoMode : FireModeState { /* hold to shoot continuously */ }

public class WeaponModeManager : StatePatternBase<FireModeState, Weapon>
{
    public WeaponModeManager(Weapon ctx) : base(ctx)
    {
        ChangeState(new SingleShotMode());
    }
}
```

---

### 3.5 Dialogue Node Navigation: Start → Typing → AwaitChoice → End

```csharp
public struct DialogueContext { public string[] lines; public int lineIndex; }
public class Dialogue : MonoBehaviour
{
    public StatePatternBase<DialogueState, Dialogue> StateManager;
    public DialogueContext Context;
    void Start()
    {
        StateManager = new DialogueStateManager(this);
    }
    void Update() => StateManager.Update();
}

public abstract class DialogueState : IStatePattern<Dialogue> { }
public class StartState : DialogueState { /* prep UI then ChangeState(TypingState) */ }
public class TypingState : DialogueState { /* Type out line; on complete => AwaitChoice */ }
public class AwaitChoiceState : DialogueState { /* wait for input then next line or EndState */ }
public class EndState : DialogueState { /* cleanup */ }

public class DialogueStateManager : StatePatternBase<DialogueState, Dialogue>
{
    public DialogueStateManager(Dialogue ctx) : base(ctx)
    {
        ChangeState(new StartState());
    }
}
```

---

### 3.6 Traffic Light: Green → Yellow → Red

```csharp
public class TrafficLight : MonoBehaviour
{
    public StatePatternBase<LightState, TrafficLight> Manager;
    void Awake() => Manager = new TrafficLightStateManager(this);
    void Update() => Manager.Update();
}

public abstract class LightState : IStatePattern<TrafficLight> { }
public class GreenState : LightState
{ public override void Enter(TrafficLight ctx) => StartCoroutine(ChangeAfter(ctx, 5, new YellowState())); }
public class YellowState : LightState { /* 2s to RedState */ }
public class RedState : LightState { /* 5s to GreenState */ }

public class TrafficLightStateManager : StatePatternBase<LightState, TrafficLight>
{
    public TrafficLightStateManager(TrafficLight ctx) : base(ctx)
    {
        ChangeState(new GreenState());
    }
}
```

---

### 3.7 Mini-Game Flow: Init → Playing → Paused → Completed

```csharp
public class MiniGameManager : MonoBehaviour
{
    public StatePatternBase<MiniState, MiniGameManager> FSM;
    void Start() { FSM = new MiniGameStateManager(this); }
    void Update() => FSM.Update();
}

public abstract class MiniState : IStatePattern<MiniGameManager> { }
public class InitState : MiniState { /* load assets then PlayingState*/ }
public class PlayingState : MiniState { /* core loop; Esc => Paused */ }
public class PausedState : MiniState { /* Esc => back to Playing */ }
public class CompletedState : MiniState { /* show results */ }

public class MiniGameStateManager : StatePatternBase<MiniState, MiniGameManager>
{
    public MiniGameStateManager(MiniGameManager ctx) : base(ctx)
    {
        ChangeState(new InitState());
    }
}
```

---

## 4. Best Practices & Edge Cases<a name="best-practices"></a>

* **Equality**: Ensure state classes override `Equals` if comparing instances.
* **Null Checks**: `_currentState` may be `null` until first `ChangeState`.
* **Exception Safety**: `ChangeState` catches exceptions; log details.
* **Lightweight States**: Keep per-state data minimal or store in context.
* **Single Responsibility**: Each state focuses on that behavior only.

---

## 5. Appendix: Full Code Listing<a name="appendix"></a>

```csharp
public interface IStatePattern<TContext>
{
    void Enter(TContext context);
    void HandleInput(TContext context);
    void Update(TContext context);
    void Exit(TContext context);
}

public abstract class StatePatternBase<TState, TContext> where TState : IStatePattern<TContext>
{
    protected TState _currentState;
    protected TContext _context;

    public StatePatternBase(TContext context) { _context = context; }

    public void ChangeState(TState newState) { /* ... */ }
    public void Update() { /* ... */ }
    public TState GetCurrentState() => _currentState;
}
```

These examples demonstrate how to apply the `StatePatternBase` abstraction across common Unity systems, enabling clear, maintainable state-driven logic.