Here's a comprehensive guide to using the `SceneHelper` class with practical examples and best practices:

---

### **SceneHelper Overview**
A robust scene management system handling async loading, state persistence, and transitions.

---

### **1. Core Features**
- **Async Loading**: Progress tracking and loading screens
- **Scene State**: Save/Restore object states and positions
- **Transitions**: Built-in fade effects
- **Additive Scenes**: Manage multiple active scenes

---

### **2. Basic Usage**
#### **Loading a Scene**
```csharp
public async Task LoadMainLevel()
{
    // Show loading screen and load scene
    await SceneHelper.LoadSceneAsync("MainLevel", LoadSceneMode.Single);
    
    // Optional: Initialize level after load
    GameManager.InitializeLevel();
}
```

#### **Additive Scene Loading**
```csharp
public async Task LoadEnvironment()
{
    await SceneHelper.LoadSceneAsync("ForestEnvironment", LoadSceneMode.Additive);
    await SceneHelper.LoadSceneAsync("WeatherSystem", LoadSceneMode.Additive);
}
```

---

### **3. Scene Transitions**
#### **Fade Transition**
```csharp
public async Task TransitionToBossArena()
{
    // Start transition
    await SceneHelper.FadeTransitionAsync(1f, Color.red);
    
    // Load scene during transition
    await SceneHelper.LoadSceneAsync("BossArena", LoadSceneMode.Single);
    
    // Complete transition
    await SceneHelper.FadeTransitionAsync(1f, Color.red);
}
```

---

### **4. State Management**
#### **Saving State**
```csharp
public async Task ReturnToHub()
{
    // Save current scene state
    SceneHelper.SaveSceneState("DungeonLevel3");
    
    // Load hub scene
    await SceneHelper.LoadSceneAsync("HubWorld", LoadSceneMode.Single);
}
```

#### **Restoring State**
```csharp
public async Task ReloadDungeon()
{
    await SceneHelper.LoadSceneAsync("DungeonLevel3", LoadSceneMode.Additive);
    SceneHelper.RestoreSceneState("DungeonLevel3");
}
```

---

### **5. Advanced Usage**
#### **Progress Tracking**
```csharp
public async Task LoadHeavyScene()
{
    await SceneHelper.LoadSceneAsync("OpenWorld", 
        onProgress: p => Debug.Log($"Loading: {p:P0}"));
}
```

#### **Combined Transition & Loading**
```csharp
public async Task SeamlessSceneChange(string newScene)
{
    await SceneHelper.FadeTransitionAsync(0.5f);
    await SceneHelper.LoadSceneAsync(newScene);
    await SceneHelper.FadeTransitionAsync(0.5f);
}
```

---

### **Best Practices**
1. **Scene Groups**
   ```csharp
   public async Task LoadUIComponents()
   {
       await SceneHelper.LoadSceneAsync("MainUI", LoadSceneMode.Additive);
       await SceneHelper.LoadSceneAsync("DialogueSystem", LoadSceneMode.Additive);
   }
   ```

2. **State Management**
   ```csharp
   void OnSceneUnload(string sceneName)
   {
       SceneHelper.SaveSceneState(sceneName);
       // Save custom data
       SceneHelper.GetSceneState(sceneName).CustomData["ChestsOpened"] = openedChests;
   }
   ```

3. **Error Handling**
   ```csharp
   public async Task SafeSceneLoad(string sceneName)
   {
       try {
           await SceneHelper.LoadSceneAsync(sceneName);
       }
       catch (Exception ex) {
           Debug.LogError($"Load failed: {ex.Message}");
           await SceneHelper.LoadSceneAsync("FallbackScene");
       }
   }
   ```

---

### **Performance Considerations**
- **Preloading**: Load heavy scenes in background
  ```csharp
  async Task PreloadAssets()
  {
      await SceneHelper.LoadSceneAsync("CityAssets", LoadSceneMode.Additive);
      SceneManager.SetActiveScene(SceneManager.GetSceneByName("CityAssets"));
  }
  ```
  
- **Memory Management**
  ```csharp
  public async Task UnloadUnusedScenes()
  {
      foreach (var scene in SceneHelper.GetActiveScenes())
      {
          if (scene != "PersistentScene")
              await SceneHelper.UnloadSceneAsync(scene);
      }
  }
  ```

---

### **Troubleshooting**
**Common Issues**:
- **Missing Scenes**: Ensure scenes are in Build Settings
- **State Restoration**: Verify object names haven't changed
- **Transition Artifacts**: Use at least 0.5s fade duration

---

### **Example Workflow**
**Level Transition**:
1. Start fade out
2. Save current level state
3. Async load new level
4. Restore any persistent state
5. Fade in

```csharp
public async Task ChangeLevel(string newLevel)
{
    // Start transition
    await SceneHelper.FadeTransitionAsync(0.5f);
    
    // Save and unload current
    SceneHelper.SaveSceneState(currentLevel);
    await SceneHelper.UnloadSceneAsync(currentLevel);
    
    // Load new
    await SceneHelper.LoadSceneAsync(newLevel);
    
    // Restore persistent data
    if (SceneHelper.HasSceneState(newLevel))
        SceneHelper.RestoreSceneState(newLevel);
    
    // Finish transition
    await SceneHelper.FadeTransitionAsync(0.5f);
}
```

---

This scene management system provides essential tools for professional Unity projects. Key features include state persistence for complex level designs and smooth transitions for polished user experiences.