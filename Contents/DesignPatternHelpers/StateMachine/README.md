**Generic State Machine in Unity**

This document explains the `StateMachine<TState, TContext>` class and `IState<TContext>` interface, providing **extensive** real-world Unity example scenarios illustrating each method.

---

## Table of Contents

1. [Overview](#overview)
2. [API Reference](#api-reference)

   * Constructor
   * AddState
   * ChangeState
   * Update
   * CurrentState
3. [Real-World Scenarios](#scenarios)

   1. Enemy AI: Patrol → Chase → Attack
   2. Player State: Idle → Running → Jumping → Falling
   3. UI Flow: MainMenu → Options → Gameplay → Pause → GameOver
   4. Boss Phases: Phase1 → Phase2 → Enraged
   5. Network Connection: Disconnected → Connecting → Connected → Error
   6. Dialogue System: Waiting → Typing → Choice → Complete
   7. Power-Up Lifecycle: Inactive → Charging → Active → Cooldown
4. [Best Practices & Edge Cases](#best-practices)
5. [Appendix: Full Code Listing](#appendix)

---

## 1. Overview

The **State Machine** pattern encapsulates behaviors into distinct state classes implementing `IState<TContext>`. `StateMachine<TState, TContext>`:

* Manages a **dictionary** of named states.
* Handles **enter**, **update**, and **exit** callbacks.
* Allows **generic** context (e.g., MonoBehaviour, Manager, data struct).

This decouples state logic from host classes, improving maintainability and testability.

---

## 2. API Reference<a name="api-reference"></a>

### Constructor

```csharp
public StateMachine(TContext context)
```

* **context**: Passed to all state callbacks (e.g., the host MonoBehaviour).

### AddState

```csharp
public void AddState<T>(T state) where T : TState
```

* Registers a state instance under its concrete type.
* **Duplicate** registration logs a warning.

**Example**:

```csharp
stateMachine.AddState(new PatrolState());
stateMachine.AddState(new ChaseState());
```

### ChangeState

```csharp
public void ChangeState<T>() where T : TState
```

* Transitions from current to target state type.
* Invokes `OnExit` on previous, then `OnEnter` on new.
* **Missing** state logs an error.

**Example**:

```csharp
stateMachine.ChangeState<ChaseState>();
```

### Update

```csharp
public void Update()
```

* Calls `OnUpdate` on the **current** state each frame or tick.
* No-op if no current state.

**Example**:

```csharp
void Update() {
    stateMachine.Update();
}
```

### CurrentState

```csharp
public TState CurrentState { get; }
```

* Exposes the active state instance for queries or debugging.

**Example**:

```csharp
Debug.Log(stateMachine.CurrentState.GetType().Name);
```

---

## 3. Real-World Scenarios<a name="scenarios"></a>

### 3.1 Enemy AI: Patrol → Chase → Attack

```csharp
// Context
public class EnemyAI : MonoBehaviour {
    public StateMachine<IState<EnemyAI>, EnemyAI> fsm;
    public Transform[] waypoints;
    void Awake() {
        fsm = new StateMachine<IState<EnemyAI>, EnemyAI>(this);
        fsm.AddState(new PatrolState());
        fsm.AddState(new ChaseState());
        fsm.AddState(new AttackState());
        fsm.ChangeState<PatrolState>();
    }
    void Update() {
        fsm.Update();
    }
}

// PatrolState
public class PatrolState : IState<EnemyAI> {
    private int idx;
    public void OnEnter(EnemyAI ctx) { idx = 0; }
    public void OnUpdate(EnemyAI ctx) {
        var target = ctx.waypoints[idx];
        ctx.transform.position = Vector3.MoveTowards(ctx.transform.position, target.position, Time.deltaTime * 2);
        if (Vector3.Distance(ctx.transform.position, target.position) < 0.1f) idx = (idx + 1) % ctx.waypoints.Length;
        if (Vector3.Distance(ctx.transform.position, ctx.player.position) < 5f)
            ctx.fsm.ChangeState<ChaseState>();
    }
    public void OnExit(EnemyAI ctx) { }
}
```

*(ChaseState and AttackState similarly handle transitions.)*

---

### 3.2 Player State: Idle → Running → Jumping → Falling

```csharp
public class PlayerController : MonoBehaviour {
    StateMachine<IState<PlayerController>, PlayerController> fsm;
    void Awake() {
        fsm = new StateMachine<IState<PlayerController>, PlayerController>(this);
        fsm.AddState(new IdleState());
        fsm.AddState(new RunningState());
        fsm.AddState(new JumpingState());
        fsm.AddState(new FallingState());
        fsm.ChangeState<IdleState>();
    }
    void Update() {
        fsm.Update();
    }
}
```

*(States detect input or velocity to trigger `ChangeState<...>();`.)*

---

### 3.3 UI Flow: MainMenu → Options → Gameplay → Pause → GameOver

```csharp
public class UIManager : MonoBehaviour {
    StateMachine<IState<UIManager>, UIManager> fsm;
    void Start() {
        fsm = new StateMachine<IState<UIManager>, UIManager>(this);
        fsm.AddState(new MainMenuState());
        fsm.AddState(new OptionsState());
        fsm.AddState(new GameplayState());
        fsm.AddState(new PauseState());
        fsm.AddState(new GameOverState());
        fsm.ChangeState<MainMenuState>();
    }
    void Update() => fsm.Update();
}
```

*(Each state shows/hides UI panels and listens for button callbacks.)*

---

### 3.4 Boss Phases: Phase1 → Phase2 → Enraged

```csharp
public class BossAI : MonoBehaviour {
    public StateMachine<IState<BossAI>, BossAI> fsm;
    public float health;
    void Awake() {
        fsm = new StateMachine<IState<BossAI>, BossAI>(this);
        fsm.AddState(new Phase1State());
        fsm.AddState(new Phase2State());
        fsm.AddState(new EnragedState());
        fsm.ChangeState<Phase1State>();
    }
    void Update() {
        fsm.Update();
        if (health < 50 && fsm.CurrentState is Phase1State)
            fsm.ChangeState<Phase2State>();
        if (health < 20 && !(fsm.CurrentState is EnragedState))
            fsm.ChangeState<EnragedState>();
    }
}
```

---

### 3.5 Network Connection: Disconnected → Connecting → Connected → Error

```csharp
public class NetworkManager : MonoBehaviour {
    StateMachine<IState<NetworkManager>, NetworkManager> fsm;
    void Start() {
        fsm = new StateMachine<IState<NetworkManager>, NetworkManager>(this);
        fsm.AddState(new DisconnectedState());
        fsm.AddState(new ConnectingState());
        fsm.AddState(new ConnectedState());
        fsm.AddState(new ErrorState());
        fsm.ChangeState<DisconnectedState>();
    }
    void Update() => fsm.Update();
}
```

*(States handle `OnEnter` to start requests, `OnUpdate` to poll, `OnExit` to cleanup.)*

---

### 3.6 Dialogue System: Waiting → Typing → Choice → Complete

```csharp
public struct DialogueContext { public string[] lines; public int index; }
public class DialogueManager : MonoBehaviour {
    StateMachine<IState<DialogueContext>, DialogueContext> fsm;
    void Start() {
        fsm = new StateMachine<IState<DialogueContext>, DialogueContext>(new DialogueContext());
        fsm.AddState(new WaitingState());
        fsm.AddState(new TypingState());
        fsm.AddState(new ChoiceState());
        fsm.AddState(new CompleteState());
        fsm.ChangeState<WaitingState>();
    }
    void Update() => fsm.Update();
}
```

---

### 3.7 Power-Up Lifecycle: Inactive → Charging → Active → Cooldown

```csharp
public class PowerUp : MonoBehaviour {
    StateMachine<IState<PowerUp>, PowerUp> fsm;
    void Awake() {
        fsm = new StateMachine<IState<PowerUp>, PowerUp>(this);
        fsm.AddState(new InactiveState());
        fsm.AddState(new ChargingState());
        fsm.AddState(new ActiveState());
        fsm.AddState(new CooldownState());
        fsm.ChangeState<InactiveState>();
    }
    void Update() => fsm.Update();
}
```

---

## 4. Best Practices & Edge Cases<a name="best-practices"></a>

* **State Reuse**: Use single instances per state if stateless, or recreate if internal data varies per context.
* **Order**: Call `AddState` before `ChangeState`.
* **Null Context**: Ensure `context` is valid; avoid passing `null`.
* **Concurrency**: Not thread-safe—invoke transitions on main thread.
* **CurrentState Checks**: Guard against `null` before accessing `OnUpdate` logic.

---

## 5. Appendix: Full Code Listing<a name="appendix"></a>

```csharp
public class StateMachine<TState, TContext> where TState : IState<TContext> {
    private Dictionary<Type, TState> _states = new Dictionary<Type, TState>();
    private TState _currentState;
    private TContext _context;
    public TState CurrentState => _currentState;

    public StateMachine(TContext context) { _context = context; }

    public void AddState<T>(T state) where T : TState { /* ... */ }
    public void ChangeState<T>() where T : TState { /* ... */ }
    public void Update() { /* ... */ }
}

public interface IState<TContext> {
    void OnEnter(TContext context);
    void OnUpdate(TContext context);
    void OnExit(TContext context);
}
```

These examples illustrate how to apply the generic state machine across common Unity systems.