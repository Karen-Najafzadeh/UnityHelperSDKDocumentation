Here's a comprehensive explanation of the `TimeHelper` class with real-world examples and best practices:

---

### **TimeHelper Overview**
A versatile time management system for Unity handling timers, cooldowns, time scaling, and scheduled actions.

---

### **1. Timer Management**
**Features**: Start/stop timers, check remaining time  
**Example**: Ability cooldown visualization
```csharp
// Start 3-second timer for dash ability
TimeHelper.StartTimer("dash_cooldown", 3f, () => {
    dashButton.interactable = true;
    cooldownText.Hide();
});

// Update UI
void Update() {
    if (TimeHelper.IsTimerRunning("dash_cooldown")) {
        cooldownText.text = TimeHelper.FormatTime(
            TimeHelper.GetRemainingTime("dash_cooldown")
        );
    }
}
```

---

### **2. Cooldown System**
**Features**: Track skill/item cooldowns  
**Example**: Player spell casting
```csharp
void CastFireball() {
    if (TimeHelper.IsCooldownComplete("fireball")) {
        // Cast spell
        TimeHelper.StartCooldown("fireball", 5f);
    }
}

// Show cooldown progress
float remaining = TimeHelper.GetCooldownRemaining("fireball");
cooldownOverlay.fillAmount = remaining / 5f;
```

---

### **3. Time Scale Control**
**Features**: Slow-motion, pause/resume  
**Example**: Bullet time effect
```csharp
// Trigger 2-second slow motion at 0.2x speed
public async void ActivateBulletTime() {
    await TimeHelper.SlowMotion(0.2f, 2f);
}

// Pause game during menus
void OpenPauseMenu() {
    TimeHelper.PauseGame();
    ShowMenu();
}

void ClosePauseMenu() {
    TimeHelper.ResumeGame();
    HideMenu();
}
```

---

### **4. Scheduling System**
**Features**: Delayed actions, time-scale awareness  
**Example**: Delayed enemy spawns
```csharp
// Spawn enemies 2 seconds after level start (ignores pause)
TimeHelper.ScheduleAction(() => {
    SpawnEnemyWave();
}, 2f, ignoreTimeScale: true);

// Must call in MonoBehaviour Update!
void Update() {
    TimeHelper.UpdateScheduledActions();
}
```

---

### **5. Utility Methods**
**Features**: Time formatting, timestamps  
**Example**: Game session timer
```csharp
float sessionTime;

void Update() {
    sessionTime += Time.deltaTime;
    timerText.text = TimeHelper.FormatTime(sessionTime);
}

// Save timestamp
long saveTime = TimeHelper.GetTimestamp();
```

---

### **Critical Implementation Notes**
1. **Timer Updates**  
   Add this component to your scene:
   ```csharp
   public class TimeUpdater : MonoBehaviour {
       void Update() {
           foreach (var timer in TimeHelper.GetAllTimers()) {
               timer.Update();
           }
           TimeHelper.UpdateScheduledActions();
       }
   }
   ```

2. **Timer Class Fix**  
   Add this method to access timers:
   ```csharp
   public static IEnumerable<Timer> GetAllTimers() {
       return _timers.Values;
   }
   ```

---

### **Best Practices**
1. **Cooldown vs Timer**  
   - Use **cooldowns** for simple readiness checks  
   - Use **timers** for complex duration tracking with callbacks

2. **Time Scale Awareness**  
   ```csharp
   // For UI animations during pause
   TimeHelper.ScheduleAction(() => {
       // Animate pause menu
   }, 1f, ignoreTimeScale: true);
   ```

3. **Error Prevention**
   ```csharp
   void StartTimerSafe(string id) {
       if (!TimeHelper.IsTimerRunning(id)) {
           TimeHelper.StartTimer(id, ...);
       }
   }
   ```

---

### **Advanced Features**
**Custom Time Scales**  
```csharp
// Slow only gameplay, not UI
void SetGameplayTimeScale(float scale) {
    TimeHelper.SetTimeScale(scale);
    uiCanvas.timeScale = 1f; // Keep UI normal speed
}
```

**Precision Scheduling**  
```csharp
// Frame-perfect animation event
TimeHelper.ScheduleAction(() => {
    character.PlaySpecialAttack();
}, 3.217f);
```

---

This enhanced time management system provides robust temporal control for Unity projects. Proper implementation requires the update component and careful consideration of time scale impacts