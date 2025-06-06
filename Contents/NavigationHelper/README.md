Below is a comprehensive reference for **NavigationHelper**, a static utility built on Unity’s NavMesh API. It provides common navigation‐related tasks such as pathfinding, smooth agent movement, spatial queries, dynamic obstacle handling, and simple path visualization.

---

## Table of Contents

1. [Overview](#overview)
2. [Path Finding](#path-finding)

   * [FindPath](#findpath)
   * [MoveAlongPathAsync](#movealongpathasync)
3. [Spatial Queries](#spatial-queries)

   * [GetNearestPoint](#getnearestpoint)
   * [IsOnNavMesh](#isonnavmesh)
   * [FindAreasInRadius](#findareasinradius)
4. [Dynamic Obstacles](#dynamic-obstacles)

   * [AddDynamicObstacle](#adddynamicobstacle)
   * [SetObstacleCarving](#setobstaclecarving)
   * [UpdateNavMeshArea](#updatenavmesharea)
5. [Helper Methods](#helper-methods)

   * [GetPathCorners](#getpathcorners)
   * [GetPathLength](#getpathlength)
   * [VisualizePath](#visualizepath)
6. [Real‐World Usage Examples](#real-world-usage-examples)

---

## Overview

**NavigationHelper** centralizes common NavMesh‐based tasks:

* **Path Finding**

  * Calculate a path between two world positions.
  * Move a `GameObject` (with or without a `NavMeshAgent`) smoothly along a NavMesh path.

* **Spatial Queries**

  * Project an arbitrary point onto the NavMesh.
  * Check if a point lies on the NavMesh.
  * Sample points around a circle to find nearby NavMesh locations.

* **Dynamic Obstacles**

  * Add and configure `NavMeshObstacle` components with carving.
  * Trigger a local NavMesh rebuild in a specified region (`Bounds`).

* **Helper Methods**

  * Extract path corners and compute total length.
  * Draw debug lines to visualize a path in the Scene or Game view.

All methods are static; no manual initialization is required. Simply call the helper methods from MonoBehaviours or other scripts in your game logic.

---

## Path Finding

### FindPath

```csharp
/// <summary>
/// Find a NavMesh path between two points.
/// </summary>
/// <param name="start">World‐space starting position.</param>
/// <param name="end">World‐space destination position.</param>
/// <param name="path">Output NavMeshPath object to store the result.</param>
/// <returns>True if a complete path was found; otherwise false.</returns>
public static bool FindPath(Vector3 start, Vector3 end, out NavMeshPath path)
```

**Description**
Queries Unity’s NavMesh system for a path from `start` to `end`. The resulting `NavMeshPath` is returned in the `out` parameter. Under the hood, this calls:

```csharp
path = new NavMeshPath();
bool success = NavMesh.CalculatePath(start, end, NavMesh.AllAreas, path);
```

* `NavMesh.AllAreas` includes all walkable areas—modify if you only want specific NavMesh layers.
* If no valid path exists or either point is invalid, `CalculatePath` returns `false`.

**Common Use Case**

* Checking whether an AI character can reach a target before issuing movement commands.
* Storing returned `NavMeshPath` for visualization or advanced logic.

**Example Usage**

```csharp
void CheckReachability(Vector3 origin, Vector3 target)
{
    if (NavigationHelper.FindPath(origin, target, out NavMeshPath path))
    {
        Debug.Log("Path found! Corner count: " + path.corners.Length);
    }
    else
    {
        Debug.LogWarning("No path could be calculated.");
    }
}
```

---

### MoveAlongPathAsync

```csharp
/// <summary>
/// Asynchronously move a GameObject (with a NavMeshAgent) toward a destination.
/// </summary>
/// <param name="agent">The GameObject to move (must or will receive a NavMeshAgent).</param>
/// <param name="destination">World‐space destination to navigate to.</param>
/// <param name="speed">
/// Optional override for the agent’s movement speed. If ≤ 0, the agent’s existing speed is unchanged.
/// </param>
/// <param name="stoppingDistance">
/// Optional override for the agent’s stopping distance. If ≤ 0, the agent’s existing stoppingDistance is unchanged.
/// </param>
/// <param name="onProgress">
/// Optional callback invoked each frame with a value [0…1] indicating navigation progress.
/// 0 = just started; 1 = arrived (remainingDistance ≤ stoppingDistance).
/// </param>
/// <returns>
/// A Task that completes when the agent has reached (or is within stoppingDistance of) the destination.
/// </returns>
public static async Task MoveAlongPathAsync(
    GameObject agent,
    Vector3 destination,
    float speed = -1,
    float stoppingDistance = -1,
    Action<float> onProgress = null)
```

**Description**
Ensures the specified `GameObject` has a `NavMeshAgent`. If it doesn’t, one is added at runtime. Then:

1. Optionally override the agent’s `speed` and `stoppingDistance`.

2. Call `navAgent.SetDestination(destination)` to begin path‐following.

3. Enter a `while` loop that yields control (`await Task.Yield()`) each frame until:

   * `navAgent.pathStatus == NavMeshPathStatus.PathComplete`, **and**
   * `navAgent.remainingDistance <= navAgent.stoppingDistance`.

4. Every iteration, if `onProgress` is provided, calculate a normalized progress value:

   ```
   float totalLength = GetPathLength(navAgent.path);
   float progress = 1f - (navAgent.remainingDistance / totalLength);
   onProgress?.Invoke(progress);
   ```

   * When the agent is at the start, `remainingDistance ≈ totalLength` → `progress ≈ 0`.
   * When the agent arrives, `remainingDistance ≤ stoppingDistance` → `progress ≈ 1`.

5. Once complete, return from the `Task`, allowing callers to `await` this method.

**Important Notes**

* If no valid path exists, the agent’s `pathStatus` becomes `Invalid` or `Partial`. This loop will exit only once `remainingDistance <= stoppingDistance`. In practice, always call `FindPath` beforehand or check `navAgent.pathStatus` after calling `SetDestination`.
* Because this uses `await Task.Yield()` each frame (instead of a coroutine), it does not block the main thread. If your project does not target .NET 4.x (where `async`/`Task` is fully supported), fall back to a coroutine approach.

**Real‐World Example**

```csharp
public class EnemyController : MonoBehaviour
{
    [SerializeField] private Transform player;
    private async void StartPursuit()
    {
        GameObject enemy = this.gameObject;

        // Show movement progress in a UI slider
        await NavigationHelper.MoveAlongPathAsync(
            enemy,
            player.position,
            speed: 3.5f,
            stoppingDistance: 0.2f,
            onProgress: (progress) =>
            {
                // e.g. update a progress bar:
                UIManager.Instance.SetProgressBar(progress);
            });
        
        Debug.Log("Enemy has arrived near the player!");
        // Perform attack or next behavior...
    }
}
```

---

## Spatial Queries

### GetNearestPoint

```csharp
/// <summary>
/// Returns the nearest position on the NavMesh to the given world‐space point.
/// </summary>
/// <param name="position">World‐space point to project onto the NavMesh.</param>
/// <returns>
/// If NavMesh sampling succeeds (within 100 units), returns the nearest valid NavMesh point; 
/// otherwise returns the original input position.
/// </returns>
public static Vector3 GetNearestPoint(Vector3 position)
```

**Description**
Internally calls:

```csharp
if (NavMesh.SamplePosition(position, out NavMeshHit hit, 100f, NavMesh.AllAreas))
    return hit.position;
else
    return position;
```

* `100f` is the maximum search radius. Adjust if your NavMesh geometry is very sparse or widely separated.
* If no NavMesh is found within that radius, you get back the original `position`; caller must decide how to handle that (e.g., abort movement).

**Example Usage**

```csharp
void TeleportToNavMesh(Vector3 arbitraryPoint)
{
    Vector3 safePoint = NavigationHelper.GetNearestPoint(arbitraryPoint);
    player.transform.position = safePoint;
    Debug.Log($"Teleported to nearest NavMesh at {safePoint}");
}
```

---

### IsOnNavMesh

```csharp
/// <summary>
/// Returns whether a given point lies on (or very near) the NavMesh.
/// </summary>
/// <param name="position">World‐space point to test.</param>
/// <param name="tolerance">
/// Maximum allowed distance off‐mesh. Default is 0.1f.
/// </param>
/// <returns>True if the NavMesh contains a sample within tolerance; otherwise false.</returns>
public static bool IsOnNavMesh(Vector3 position, float tolerance = 0.1f)
```

**Description**
Calls:

```csharp
return NavMesh.SamplePosition(position, out NavMeshHit hit, tolerance, NavMesh.AllAreas);
```

* If the NavMesh has a vertex/triangle within `tolerance` units of `position`, returns `true`.
* Helpful for quickly validating whether a point is reachable or valid for agent placement.

**Example Usage**

```csharp
void CheckSpawnPosition(Vector3 spawnPos)
{
    if (NavigationHelper.IsOnNavMesh(spawnPos))
        Debug.Log("Safe to spawn enemy here.");
    else
        Debug.LogWarning("Spawn position is off‐mesh! Choose another location.");
}
```

---

### FindAreasInRadius

```csharp
/// <summary>
/// Sample a circular arrangement of points around `center` to detect
/// which NavMesh triangles/areas lie within `radius`.
/// </summary>
/// <param name="center">World‐space center of the search circle.</param>
/// <param name="radius">Radius (in world units) from center to sample NavMesh points.</param>
/// <returns>
/// An array of `NavMeshHit` for each sample that actually found a NavMesh position within `radius`.
/// </returns>
public static NavMeshHit[] FindAreasInRadius(Vector3 center, float radius)
```

**Description**

1. Creates an empty `List<NavMeshHit>`.
2. Chooses `SampleCount = 8` equally spaced angles (0°, 45°, 90°, …).
3. For each angle `i`, compute a direction vector:

   ```
   angle = i * (2π / SampleCount)
   direction = (cos(angle), 0, sin(angle))
   pointToSample = center + direction * radius
   ```
4. Calls `NavMesh.SamplePosition(pointToSample, out hit, radius, NavMesh.AllAreas)`.
5. If sampling succeeds, add `hit` to the list.
6. Return the final `hits.ToArray()`.

**Use Cases**

* Quickly query “which NavMesh areas are approximately within X units of this point?”
* Implement simple avoidance checks—e.g., “Are there any nodes on the NavMesh at the edges of a circle around me?”
* Can be extended to more sample points if you need finer coverage.

**Example Usage**

```csharp
void HighlightNearbyNavAreas(Vector3 position)
{
    float searchRadius = 5f;
    NavMeshHit[] hits = NavigationHelper.FindAreasInRadius(position, searchRadius);

    foreach (var hit in hits)
    {
        // E.g., place a debug marker or highlight a mesh at hit.position
        Debug.DrawLine(position, hit.position, Color.green, 2f);
    }

    Debug.Log($"Found {hits.Length} NavMesh samples around {position}");
}
```

---

## Dynamic Obstacles

### AddDynamicObstacle

```csharp
/// <summary>
/// Attaches a NavMeshObstacle component to `obj` so it will carve into the NavMesh at runtime.
/// </summary>
/// <param name="obj">GameObject on which to add the obstacle (should have a Collider or represent an obstacle region).</param>
/// <param name="size">World‐space size of the obstacle’s bounding box (x, y, z).</param>
/// <returns>The newly created NavMeshObstacle.</returns>
public static NavMeshObstacle AddDynamicObstacle(GameObject obj, Vector3 size)
```

**Description**

1. `obj.AddComponent<NavMeshObstacle>()`.
2. Sets:

   * `obstacle.carving = true` → This makes the obstacle “carve” a hole in the NavMesh at runtime.
   * `obstacle.size = size` → Defines the box size of the obstacle region. If your object is not box‐shaped, approximate with a bounding box.

**Important**

* Carving is more expensive at runtime, especially if you have many moving obstacles. Use sparingly.
* By default `carveOnlyStationary == false`, meaning even if the obstacle moves, carving will update. If you only want carving when the obstacle stops, set `carveOnlyStationary = true`.

**Example Usage**

```csharp
public class DoorObstacle : MonoBehaviour
{
    private NavMeshObstacle _obstacle;

    private void Start()
    {
        // Suppose this door is 1x2x0.2 units in size
        _obstacle = NavigationHelper.AddDynamicObstacle(gameObject, new Vector3(1f, 2f, 0.2f));
    }

    public void ToggleDoor(bool open)
    {
        // Disable carving when open, enable carving when closed
        NavigationHelper.SetObstacleCarving(_obstacle, !open);
    }
}
```

---

### SetObstacleCarving

```csharp
/// <summary>
/// Enable or disable carving for an existing NavMeshObstacle.
/// </summary>
/// <param name="obstacle">NavMeshObstacle component previously added.</param>
/// <param name="enabled">
/// If true, carving is enabled (navmesh cuts an opening around the obstacle).
/// If false, carving is disabled (NavMesh is unmodified).
/// </param>
public static void SetObstacleCarving(NavMeshObstacle obstacle, bool enabled)
```

**Description**

* Simply toggles `obstacle.carving = enabled`.
* If enabling, also forces `obstacle.carveOnlyStationary = false` (so carving updates even if the obstacle moves).

**Example Usage**

```csharp
void OnPitfallTriggered(GameObject trap)
{
    // Suppose trap has a NavMeshObstacle attached
    var obstacle = trap.GetComponent<NavMeshObstacle>();
    if (obstacle != null)
    {
        // Temporarily disable carving so AI can path through the trap region
        NavigationHelper.SetObstacleCarving(obstacle, false);
    }
}
```

---

### UpdateNavMeshArea

```csharp
/// <summary>
/// Perform a runtime NavMesh rebuild in a specified bounding volume.
/// </summary>
/// <param name="bounds">Axis‐aligned bounding box in world space where the NavMesh should be updated.</param>
public static void UpdateNavMeshArea(Bounds bounds)
```

**Description**

1. Creates a new `NavMeshData` instance.
2. Gathers an (empty) `List<NavMeshBuildSource>`. In many cases, you’d collect actual sources (e.g., `NavMeshBuilder.CollectSources(...)`), but here it’s left empty, so rebuilding will effectively remove any prior baked data in `bounds`.
3. Calls:

   ```csharp
   var instance = NavMesh.AddNavMeshData(data);
   NavMeshBuilder.UpdateNavMeshData(
       data,
       NavMesh.GetSettingsByID(0),
       sources,
       bounds
   );
   ```

   * `NavMesh.GetSettingsByID(0)` returns the default NavMesh build settings (for agents of radius \~0.5m, etc.).
   * `sources` is an empty list—if you want dynamic runtime NavMesh building, you typically fill `sources` with objects’ meshes or colliders in that volume.

**Important Caveats**

* As shown, this method does not collect any actual geometry, so the rebuilt NavMesh in `bounds` may be empty. Usually you’d do:

  ```csharp
  var sources = new List<NavMeshBuildSource>();
  NavMeshBuilder.CollectSources(bounds, LayerMask.GetMask("Walkable"), NavMeshCollectGeometry.RenderMeshes, 0, new List<NavMeshBuildMarkup>(), sources);
  ```
* Then pass `sources` to `UpdateNavMeshData`.

**Example Usage**

```csharp
void RebuildLocalNavMeshArea(Vector3 point, float radius)
{
    Bounds localBounds = new Bounds(point, new Vector3(radius * 2, 2f, radius * 2));
    
    // This will (currently) produce an empty NavMesh in that area. For a real rebuild,
    // you'd call NavMeshBuilder.CollectSources(...) to populate 'sources'.
    NavigationHelper.UpdateNavMeshArea(localBounds);

    Debug.Log($"Requested NavMesh update within {localBounds}.");
}
```

---

## Helper Methods

### GetPathCorners

```csharp
/// <summary>
/// Returns the corner points (waypoints) of a computed NavMeshPath.
/// </summary>
/// <param name="path">A NavMeshPath returned by FindPath or obtained from a NavMeshAgent.</param>
/// <returns>
/// An array of Vector3 representing each corner along the path. If `path` is null,
/// returns an empty array.
/// </returns>
public static Vector3[] GetPathCorners(NavMeshPath path)
```

**Description**

* If `path == null`, returns `new Vector3[0]`.
* Otherwise, returns `path.corners`, which is the sequence of world‐space points where the path changes direction.

**Example Usage**

```csharp
void DrawPathCorners(NavMeshPath path)
{
    Vector3[] corners = NavigationHelper.GetPathCorners(path);
    for (int i = 0; i < corners.Length; i++)
    {
        Debug.DrawLine(corners[i], corners[i] + Vector3.up * 0.5f, Color.cyan, 2f);
    }
}
```

---

### GetPathLength

```csharp
/// <summary>
/// Calculate the total length (in world units) of a NavMeshPath by summing distances between corners.
/// </summary>
/// <param name="path">NavMeshPath to measure. If null or has fewer than 2 corners, returns 0.</param>
/// <returns>Total length along the path’s corners.</returns>
public static float GetPathLength(NavMeshPath path)
```

**Description**

* If `path == null` or `path.corners.Length < 2`, immediately returns `0f`.
* Else:

  ```csharp
  float length = 0f;
  for (int i = 0; i < path.corners.Length - 1; i++)
      length += Vector3.Distance(path.corners[i], path.corners[i + 1]);
  return length;
  ```

**Example Usage**

```csharp
void LogPathInfo(Vector3 start, Vector3 end)
{
    if (NavigationHelper.FindPath(start, end, out NavMeshPath path))
    {
        float length = NavigationHelper.GetPathLength(path);
        Debug.Log($"Calculated path length from A to B: {length:F2} meters.");
    }
    else
    {
        Debug.Log("No path could be found for length measurement.");
    }
}
```

---

### VisualizePath

```csharp
/// <summary>
/// Draw lines between each pair of consecutive corners in a NavMeshPath, 
/// using Debug.DrawLine. The lines persist for `_pathUpdateInterval` seconds.
/// </summary>
/// <param name="path">NavMeshPath whose corners will be connected.</param>
public static void VisualizePath(NavMeshPath path)
```

**Description**

1. If `_showDebugPath == false` or `path == null`, do nothing.
2. Otherwise, loop over `path.corners[i]` to `path.corners[i+1]` and call:

   ```csharp
   Debug.DrawLine(path.corners[i], path.corners[i + 1], _pathColor, _pathUpdateInterval);
   ```
3. Uses static fields `_pathColor` (defaults to `Color.yellow`) and `_pathUpdateInterval` (defaults to `0.5f` seconds).

**How to Enable**

* Before calling `VisualizePath`, set:

  ```csharp
  // Show debug path
  typeof(NavigationHelper)
      .GetField("_showDebugPath", BindingFlags.Static | BindingFlags.NonPublic)
      .SetValue(null, true);
  ```

  Or modify the source to expose a public setter. By default `_showDebugPath` is `false`.

**Example Usage**

```csharp
void DrawNavPath(Vector3 start, Vector3 end)
{
    if (NavigationHelper.FindPath(start, end, out NavMeshPath path))
    {
        // Temporarily enable debug drawing
        // (In real code, you’d expose a public method to toggle this)
        NavigationHelperVisualToggle(true);
        NavigationHelper.VisualizePath(path);
    }
}

// Helper method to reflectively set the private flag (for demonstration):
void NavigationHelperVisualToggle(bool enabled)
{
    var field = typeof(NavigationHelper).GetField("_showDebugPath", 
                   System.Reflection.BindingFlags.Static | System.Reflection.BindingFlags.NonPublic);
    field.SetValue(null, enabled);
}
```

---

## Real‐World Usage Examples

Below are a few end‐to‐end scenarios that illustrate how to combine the above methods in practice.

### Example 1: AI Patrol Between Waypoints

```csharp
public class PatrolAI : MonoBehaviour
{
    [SerializeField] private Transform[] waypoints;
    private int _currentIndex = 0;

    private async void Start()
    {
        // Ensure there are at least two waypoints
        if (waypoints.Length < 2) return;
        
        // Loop indefinitely
        while (true)
        {
            Vector3 target = waypoints[_currentIndex].position;
            await NavigationHelper.MoveAlongPathAsync(
                gameObject,
                target,
                speed: 2.5f,
                stoppingDistance: 0.1f,
                onProgress: (progress) =>
                {
                    // Optional: update a UI indicator above the AI showing progress
                    float pct = Mathf.Clamp01(progress);
                    // Example: HealthBarUI.Instance.UpdateProgressBar(this, pct);
                });

            // Advance to next waypoint
            _currentIndex = (_currentIndex + 1) % waypoints.Length;
        }
    }
}
```

**Explanation**

* The AI cycles through a list of `Transform` waypoints.
* Each iteration, it calls `MoveAlongPathAsync`. Because this returns a `Task`, `await`ing it ensures the AI will not proceed to the next waypoint until it arrives.
* You can pass `onProgress` to display a progress bar in the UI or drive any other logic tied to movement completion.

---

### Example 2: Dynamic Obstacle Toggle in a Door

```csharp
public class DoorController : MonoBehaviour
{
    private NavMeshObstacle _doorObstacle;

    private void Awake()
    {
        // Suppose this door has a BoxCollider with size (1, 2, 0.2)
        _doorObstacle = NavigationHelper.AddDynamicObstacle(gameObject, new Vector3(1f, 2f, 0.2f));
        // Initially closed → carving on:
        NavigationHelper.SetObstacleCarving(_doorObstacle, true);
    }

    public void OpenDoor()
    {
        // Disable carving so agents can pass through
        NavigationHelper.SetObstacleCarving(_doorObstacle, false);
        // Play open animation...
    }

    public void CloseDoor()
    {
        // Re‐enable carving so agents avoid this region
        NavigationHelper.SetObstacleCarving(_doorObstacle, true);
        // Play close animation...
    }
}
```

**Explanation**

* When the door is “closed,” it carves out a hole in the NavMesh, preventing agents from walking through.
* When “open,” carving is disabled, so the NavMesh no longer treats the door as an obstacle.

---

### Example 3: Teleport Player Safely onto NavMesh

```csharp
public class TeleportSystem : MonoBehaviour
{
    public async void TeleportPlayerTo(Vector3 rawPosition)
    {
        // First, sample nearest NavMesh
        Vector3 navPos = NavigationHelper.GetNearestPoint(rawPosition);
        
        // Then check if that navPos is truly on NavMesh
        if (!NavigationHelper.IsOnNavMesh(navPos))
        {
            Debug.LogWarning("Could not find any NavMesh within tolerance. Aborting teleport.");
            return;
        }

        // If valid, move the player immediately
        player.transform.position = navPos;
        player.transform.rotation = Quaternion.identity;
        
        // Optionally, figure out a path to check if we can stand there:
        if (NavigationHelper.FindPath(player.transform.position, navPos, out NavMeshPath path))
        {
            float length = NavigationHelper.GetPathLength(path);
            Debug.Log($"Teleported to NavMesh at {navPos}, path length: {length}");
        }
        else
        {
            Debug.Log("Teleported despite no valid path (maybe NavMesh changed).");
        }
    }
}
```

**Explanation**

* Ensures `rawPosition` is projected onto a valid NavMesh location.
* If within `tolerance`, updates the player’s `transform.position` directly.
* Optionally logs the path length in case you need to debug connectivity.

---

### Example 4: Visual Debugging of a Computed Path

```csharp
public class PathDebugger : MonoBehaviour
{
    [SerializeField] private Transform startPoint;
    [SerializeField] private Transform endPoint;

    private void Update()
    {
        // Every frame, compute and visualize a path from start to end
        if (NavigationHelper.FindPath(startPoint.position, endPoint.position, out NavMeshPath path))
        {
            // Temporarily enable drawing flag (via reflection or by exposing a public setter)
            NavigationHelperVisualToggle(true);
            NavigationHelper.VisualizePath(path);
        }
    }

    // Reflection helper for demonstration (in production, expose a public toggle instead)
    private void NavigationHelperVisualToggle(bool enabled)
    {
        var field = typeof(NavigationHelper)
            .GetField("_showDebugPath", System.Reflection.BindingFlags.Static | System.Reflection.BindingFlags.NonPublic);
        field.SetValue(null, enabled);
    }
}
```

**Explanation**

* In `Update()`, we recalculate the path each frame (costly—only do this sparingly in production!).
* We then call `VisualizePath(path)` to draw yellow lines between each pair of corners for `_pathUpdateInterval` seconds (0.5s by default).

---

## Configuration and Customization

* **Default Movement Parameters**

  * `_defaultStoppingDistance = 0.1f`
  * `_defaultMovementSpeed = 5f`
  * `_pathUpdateInterval = 0.5f`
    Change these by editing the static fields in `NavigationHelper` if you want different defaults.

* **Debug Path Appearance**

  * `_showDebugPath` (private `bool`): toggle on/off.
  * `_pathColor` (private `Color`): default `Color.yellow`.
  * `_pathWidth`: used if you implement a line‐renderer style drawing—currently unused since `Debug.DrawLine` ignores width.

To expose these fields publicly, simply add public setters or methods in `NavigationHelper`:

```csharp
public static void EnablePathVisualization(bool enable)
{
    _showDebugPath = enable;
}
public static void SetPathColor(Color color)
{
    _pathColor = color;
}
public static void SetPathUpdateInterval(float interval)
{
    _pathUpdateInterval = interval;
}
```

---

## Summary

**NavigationHelper** is designed to be a “one‐stop shop” for:

* Quickly computing NavMesh paths (`FindPath`)
* Smoothly moving agents (`MoveAlongPathAsync`)
* Querying whether arbitrary points are on the NavMesh (`IsOnNavMesh`, `GetNearestPoint`)
* Finding nearby NavMesh samples (`FindAreasInRadius`)
* Dynamically carving obstacles (`AddDynamicObstacle`, `SetObstacleCarving`)
* (Basic) runtime NavMesh rebuild inside a bounding box (`UpdateNavMeshArea`)
* Accessing and visualizing path segments (`GetPathCorners`, `GetPathLength`, `VisualizePath`)

Use these helpers to:

1. Confirm reachability (via `FindPath`),
2. Teleport or reposition characters safely onto the NavMesh,
3. Add/remove runtime obstacles that agents avoid,
4. Drive asynchronous movement logic with progress callbacks, and
5. Draw debug‐only lines for designers to fine‐tune NavMesh geometry.
