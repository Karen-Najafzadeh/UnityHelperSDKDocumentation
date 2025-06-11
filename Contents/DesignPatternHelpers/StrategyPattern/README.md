**Generic Strategy Pattern in Unity**

This document explains the `IStrategy<TContext, TResult>` interface and `StrategyContext<TContext, TResult>` class, along with **extensive** real-world Unity examples illustrating flexible behavior injection at runtime.

---

## Table of Contents

1. [Overview](#overview)
2. [API Reference](#api-reference)

   * `IStrategy<TContext, TResult>`
   * `StrategyContext<TContext, TResult>`
3. [Example Scenarios](#scenarios)

   1. Movement Strategies (Walk, Run, Fly)
   2. Camera Follow Strategies (Smooth, Immediate, Orbital)
   3. Damage Calculation (Physical vs. Magical)
   4. Enemy AI Decision (Aggressive vs. Defensive)
   5. Pathfinding (A\* vs. Dijkstra)
   6. UI Layout (Grid vs. List)
   7. Audio Mixing (Stereo vs. Mono)
   8. Score Evaluation (Time-Based vs. Accuracy-Based)
   9. Animation Blending (Linear vs. Cubic)
4. [Best Practices & Edge Cases](#best-practices)
5. [Appendix: Full Code Listing](#appendix)

---

## 1. Overview<a name="overview"></a>

The **Strategy Pattern** enables selecting an algorithm’s behavior at runtime by encapsulating each algorithm in a class implementing a common interface. In Unity, it’s ideal for interchangeable gameplay mechanics.

* **`IStrategy<TContext, TResult>`** defines a single method `Execute`.
* **`StrategyContext<TContext, TResult>`** holds and executes the current strategy.

This decouples the context (e.g., player, camera, AI) from the algorithms, allowing dynamic swapping without conditional logic.

---

## 2. API Reference<a name="api-reference"></a>

### `IStrategy<TContext, TResult>`

```csharp
public interface IStrategy<TContext, TResult>
{
    TResult Execute(TContext context);
}
```

* **TContext**: Input data type (e.g., `Transform`, `Enemy`, `List<Item>`).
* **TResult**: Output type (e.g., `Vector3`, `float`, `bool`).

---

### `StrategyContext<TContext, TResult>`

```csharp
public class StrategyContext<TContext, TResult>
{
    private IStrategy<TContext, TResult> _strategy;

    public StrategyContext(IStrategy<TContext, TResult> strategy)
    {
        _strategy = strategy;
    }

    public void SetStrategy(IStrategy<TContext, TResult> strategy)
    {
        _strategy = strategy;
    }

    public TResult ExecuteStrategy(TContext context)
    {
        if (_strategy == null)
        {
            Debug.LogError("No strategy set");
            return default;
        }
        return _strategy.Execute(context);
    }
}
```

* **Constructor**: Inject initial strategy.
* **SetStrategy**: Swap algorithm at runtime.
* **ExecuteStrategy**: Runs the current strategy, logs error if none set.

---

## 3. Example Scenarios<a name="scenarios"></a>

### 3.1 Movement Strategies (Walk, Run, Fly)

```csharp
public class WalkStrategy : IStrategy<Transform, Vector3> {
    public Vector3 Execute(Transform ctx) => ctx.position + ctx.forward * 2f * Time.deltaTime;
}
public class RunStrategy : IStrategy<Transform, Vector3> {
    public Vector3 Execute(Transform ctx) => ctx.position + ctx.forward * 5f * Time.deltaTime;
}
public class FlyStrategy : IStrategy<Transform, Vector3> {
    public Vector3 Execute(Transform ctx) => ctx.position + (ctx.forward + Vector3.up) * 3f * Time.deltaTime;
}

// Usage in PlayerController
var movementContext = new StrategyContext<Transform, Vector3>(new WalkStrategy());
void Update() {
    if (Input.GetKey(KeyCode.LeftShift))
        movementContext.SetStrategy(new RunStrategy());
    if (Input.GetKey(KeyCode.Space))
        movementContext.SetStrategy(new FlyStrategy());
    transform.position = movementContext.ExecuteStrategy(transform);
}
```

### 3.2 Camera Follow Strategies (Smooth, Immediate, Orbital)

```csharp
public class ImmediateFollow : IStrategy<Camera, Vector3> {
    public Vector3 Execute(Camera cam) => target.position;
}
public class SmoothFollow : IStrategy<Camera, Vector3> {
    public Vector3 Execute(Camera cam) => Vector3.Lerp(cam.transform.position, target.position, 0.1f);
}
public class OrbitalFollow : IStrategy<Camera, Vector3> {
    public Vector3 Execute(Camera cam) => target.position + Quaternion.Euler(0, Time.time * 20, 0) * offset;
}

// Usage
var camContext = new StrategyContext<Camera, Vector3>(new SmoothFollow());
void LateUpdate() {
    cam.transform.position = camContext.ExecuteStrategy(cam);
}
```

### 3.3 Damage Calculation (Physical vs. Magical)

```csharp
public class PhysicalDamage : IStrategy<DamageContext, float> {
    public float Execute(DamageContext ctx) => ctx.AttackPower - ctx.Defense;
}
public class MagicalDamage : IStrategy<DamageContext, float> {
    public float Execute(DamageContext ctx) => ctx.MagicPower * 1.2f;
}

// Usage
var damageCtx = new StrategyContext<DamageContext, float>(new PhysicalDamage());
float dmg = damageCtx.ExecuteStrategy(new DamageContext { AttackPower=50, Defense=20 });
```

### 3.4 Enemy AI Decision (Aggressive vs. Defensive)

```csharp
public class AggressiveAI : IStrategy<EnemyAI, Action> {
    public Action Execute(EnemyAI ctx) => ctx.ChasePlayer;
}
public class DefensiveAI : IStrategy<EnemyAI, Action> {
    public Action Execute(EnemyAI ctx) => ctx.FindCover;
}

// Usage
enemy.ContextStrategy = new StrategyContext<EnemyAI, Action>(new AggressiveAI());
enemy.CurrentBehavior = enemy.ContextStrategy.ExecuteStrategy(enemy);
```

### 3.5 Pathfinding (A\* vs. Dijkstra)

```csharp
public class AStarPath : IStrategy<PathRequest, List<Vector3>> { /* A* impl */ }
public class DijkstraPath : IStrategy<PathRequest, List<Vector3>> { /* Dijkstra impl */ }

// Usage
var pathCtx = new StrategyContext<PathRequest, List<Vector3>>(new AStarPath());
var route = pathCtx.ExecuteStrategy(request);
```

### 3.6 UI Layout (Grid vs. List)

```csharp
public class GridLayoutStrategy : IStrategy<UIContainer, bool> { /* arrange children in grid */ }
public class ListLayoutStrategy : IStrategy<UIContainer, bool> { /* arrange vertically */ }

// Usage
var layoutCtx = new StrategyContext<UIContainer, bool>(new GridLayoutStrategy());
layoutCtx.ExecuteStrategy(inventoryPanel);
```

### 3.7 Audio Mixing (Stereo vs. Mono)

```csharp
public class StereoMix : IStrategy<AudioSource, AudioClip> {
    public AudioClip Execute(AudioSource src) { src.spatialBlend = 0; return src.clip; }
}
public class MonoMix : IStrategy<AudioSource, AudioClip> {
    public AudioClip Execute(AudioSource src) { src.spatialBlend = 1; return src.clip; }
}

// Usage
audioCtx.SetStrategy(new MonoMix());
audioCtx.ExecuteStrategy(audioSource);
```

### 3.8 Score Evaluation (Time-Based vs. Accuracy-Based)

```csharp
public class TimeScore : IStrategy<GameStats, int> { public int Execute(GameStats ctx) => (int)(1000 - ctx.TimeTaken); }
public class AccuracyScore : IStrategy<GameStats, int> { public int Execute(GameStats ctx) => (int)(ctx.Hits / (float)ctx.Attempts * 100); }

// Usage
var scoreCtx = new StrategyContext<GameStats, int>(new TimeScore());
int score = scoreCtx.ExecuteStrategy(stats);
```

### 3.9 Animation Blending (Linear vs. Cubic)

```csharp
public class LinearBlend : IStrategy<AnimationBlendContext, AnimationClip> { /* linear interpolation */ }
public class CubicBlend : IStrategy<AnimationBlendContext, AnimationClip> { /* cubic interpolation */ }

// Usage
var blendCtx = new StrategyContext<AnimationBlendContext, AnimationClip>(new CubicBlend());
```

---

## 4. Best Practices & Edge Cases<a name="best-practices"></a>

* **Null Strategy**: Always check for `_strategy != null`.
* **Stateless vs Stateful**: Strategies should be stateless or reset when reused.
* **Context Integrity**: Ensure `TContext` carries all required data.
* **Performance**: Avoid heavy allocations in `Execute`; cache results if needed.
* **Swap Frequency**: Minimize frequent `SetStrategy` calls in tight loops.

---

## 5. Appendix: Full Code Listing<a name="appendix"></a>

```csharp
public interface IStrategy<TContext, TResult>
{
    TResult Execute(TContext context);
}

public class StrategyContext<TContext, TResult>
{
    private IStrategy<TContext, TResult> _strategy;

    public StrategyContext(IStrategy<TContext, TResult> strategy) { _strategy = strategy; }
    public void SetStrategy(IStrategy<TContext, TResult> strategy) { _strategy = strategy; }
    public TResult ExecuteStrategy(TContext context)
    {
        if (_strategy == null)
        {
            Debug.LogError("No strategy set");
            return default;
        }
        return _strategy.Execute(context);
    }
}
```

These examples demonstrate how the Strategy pattern can be leveraged across diverse Unity systems to inject flexible, interchangeable algorithms.