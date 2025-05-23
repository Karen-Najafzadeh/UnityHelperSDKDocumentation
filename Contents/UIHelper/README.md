Here's a comprehensive explanation of the `UIHelper` class with real-world examples and best practices:

---

### **UIHelper Overview**
A robust UI management system handling common UI patterns, animations, pooling, and responsive layouts while integrating with localization and pooling systems.

---

### **1. Core Features**
#### **UI Pool Management**
```csharp
// Register custom UI elements
UIHelper.RegisterUIPrefab("InventoryItem", inventoryItemPrefab);

// Get pooled element
var item = UIHelper.GetOrCreateUIElement("InventoryItem");
```

#### **Toast Notifications**
```csharp
// Show error message
await UIHelper.ShowToast("ERROR_CONNECTION_LOST", Color.red);

// Localized success message
await UIHelper.ShowToast("SUCCESS_SAVE_COMPLETE");
```

---

### **2. Popup System**
#### **Confirmation Dialog**
```csharp
int result = await UIHelper.ShowPopup(
    "CONFIRM_DELETE_TITLE",
    "CONFIRM_DELETE_MSG",
    "BUTTON_CANCEL", "BUTTON_DELETE"
);

if (result == 1) DeleteItem();
```

#### **Custom Popup**
```csharp
// Create custom popup prefab
UIHelper.RegisterUIPrefab("AchievementPopup", achievementPopupPrefab);

// Show custom popup
var popup = UIHelper.GetOrCreateUIElement("AchievementPopup");
popup.GetComponent<AchievementPopup>().Show(achievementData);
```

---

### **3. Loading Indicators**
```csharp
// Show during async operation
GameObject spinner = null;

async Task LoadGameData() {
    spinner = UIHelper.ShowLoadingSpinner();
    await LoadDataAsync();
    UIHelper.HideLoadingSpinner(spinner);
}
```

---

### **4. Dynamic UI Generation**
#### **Runtime UI Creation**
```csharp
// Create settings menu
var panel = new GameObject("SettingsPanel").AddComponent<Image>();
var title = UIHelper.CreateLabel("SETTINGS_TITLE", panel.transform);
var volumeSlider = UIHelper.CreateSlider("SETTINGS_VOLUME", panel.transform);
```

#### **Dynamic Buttons**
```csharp
foreach (var weapon in unlockedWeapons) {
    var btn = UIHelper.CreateButton(weapon.Name, () => SelectWeapon(weapon));
    btn.transform.SetParent(weaponList);
}
```

---

### **5. Responsive Layouts**
```csharp
// Create responsive panel
var panel = CreatePanel();
UIHelper.MakeResponsive(panel.GetComponent<RectTransform>(), 
    minWidth: 300f, 
    maxWidth: 800f);

// Update when screen changes
void OnScreenSizeChanged() {
    UIHelper.UpdateResponsiveLayout();
}
```

---

### **Best Practices**
1. **Separation of Concerns**
   ```csharp
   // Good practice
   public class ShopUI : MonoBehaviour {
       void Show() {
           UIHelper.ShowPopup("SHOP_TITLE", "SHOP_WELCOME");
       }
   }
   ```

2. **Pooling Strategy**
   ```csharp
   // Bad: Instantiate directly
   // Good: Use pool
   var item = UIHelper.GetOrCreateUIElement("InventoryItem");
   ```

3. **Animation Sequencing**
   ```csharp
   async Task ShowMainMenu() {
       await UIHelper.AnimateElementIn(logo);
       await UIHelper.AnimateElementIn(menuButtons);
       await UIHelper.AnimateElementIn(socialPanel);
   }
   ```

---

### **Performance Considerations**
1. **Pool Limits**
   ```csharp
   // Set reasonable pool sizes
   void Initialize() {
       UIHelper.PrewarmPool("Toast", 5);
       UIHelper.PrewarmPool("Popup", 3);
   }
   ```

2. **Animation Optimization**
   ```csharp
   // Cancel redundant animations
   void OnDestroy() {
       transform.DOKill();
   }
   ```

---

### **Integration Points**
1. **Localization**
   ```csharp
   // All text elements automatically localized
   buttonText.text = LocalizationManager.Get("BUTTON_CONTINUE");
   ```

2. **State Management**
   ```csharp
   void UpdateUI(GameState state) {
       healthBar.value = state.Health;
       UpdateScoreDisplay(state.Score);
   }
   ```

---

### **Advanced Features**
#### **Custom Transitions**
```csharp
// Custom popup animation
async Task AnimateCustomPopup(GameObject popup) {
    await popup.transform.DOPunchScale(Vector3.one * 0.2f, 0.3f);
    await popup.GetComponent<CanvasGroup>().DOFade(1f, 0.2f);
}
```

#### **Adaptive Layouts**
```csharp
// Phone vs tablet layouts
void UpdateLayout() {
    if (IsTablet) {
        ApplyTwoColumnLayout();
    } else {
        ApplySingleColumnLayout();
    }
}
```

---

### **Troubleshooting**
1. **Missing Canvas Reference**
   ```csharp
   // Always initialize first
   void Start() {
       UIHelper.Initialize(mainCanvas);
   }
   ```

2. **Animation Conflicts**
   ```csharp
   // Proper cleanup
   void OnDisable() {
       transform.DOKill();
       GetComponent<CanvasGroup>().DOKill();
   }
   ```

---

This UI system provides a comprehensive solution for modern Unity projects. Key features include efficient pooling for performance-critical UIs, built-in animations, and deep localization integration. The responsive layout system helps maintain consistent experiences across devices, while the component-based architecture enables easy extension.