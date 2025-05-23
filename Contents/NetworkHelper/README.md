Here's a comprehensive guide to using the `NetworkRequestManager` class with practical examples and explanations:

---

### **NetworkRequestManager Overview**
A robust HTTP client for Unity featuring retries, concurrency control, and support for multiple data formats.

---

### **1. Core Features**
- **Automatic Retries**: 3 attempts with exponential backoff
- **Concurrency Control**: 4 simultaneous requests by default
- **Timeout Handling**: 10 seconds default
- **Authentication**: JWT/Bearer token support
- **File Transfers**: Download/upload with progress tracking
- **Media Support**: Direct texture/audio loading

---

### **2. Basic Usage**
#### **GET Request (JSON)**
```csharp
public async Task LoadPlayerProfile()
{
    try
    {
        var profile = await NetworkRequestManager.GetJsonAsync<PlayerProfile>(
            "https://api.game.com/players/123"
        );
        DisplayProfile(profile);
    }
    catch (Exception ex)
    {
        Debug.LogError($"Profile load failed: {ex.Message}");
    }
}
```

#### **POST Request (JSON)**
```csharp
public async Task SaveHighScore(int score)
{
    try
    {
        var result = await NetworkRequestManager.PostJsonAsync<SaveResult>(
            "https://api.game.com/scores",
            new { score = score, level = 5 }
        );
        ShowSuccess("Score saved!");
    }
    catch (Exception ex)
    {
        ShowError($"Save failed: {ex.Message}");
    }
}
```

---

### **3. Advanced Operations**
#### **File Download**
```csharp
public async Task DownloadMapPackage(string mapId)
{
    var progress = new Progress<float>(p => 
        loadingBar.value = p
    );
    
    await NetworkRequestManager.DownloadFileAsync(
        $"https://cdn.game.com/maps/{mapId}.zip",
        Path.Combine(Application.persistentDataPath, $"{mapId}.zip"),
        progress
    );
}
```

#### **Image Loading**
```csharp
public async Task<Texture2D> LoadAvatar(string userId)
{
    return await NetworkRequestManager.DownloadTextureAsync(
        $"https://avatars.game.com/{userId}.png"
    );
}
```

---

### **4. Error Handling & Retries**
```csharp
public async Task<bool> TryLoadShopItems()
{
    var (success, items) = await NetworkRequestManager.TryGetJsonAsync<ShopItem[]>(
        "https://api.game.com/shop"
    );
    
    if (success)
    {
        DisplayShop(items);
        return true;
    }
    
    ShowError("Failed to load shop");
    return false;
}
```

---

### **5. Custom Requests**
#### **PATCH Request**
```csharp
public async Task UpdatePlayerStatus(string newStatus)
{
    await NetworkRequestManager.PatchJsonAsync<PlayerUpdateResponse>(
        "https://api.game.com/players/me/status",
        new { status = newStatus }
    );
}
```

#### **Custom Headers**
```csharp
public async Task LoadRegionalContent()
{
    var headers = new Dictionary<string, string>
    {
        ["X-Region"] = GetPlayerRegion()
    };
    
    return await NetworkRequestManager.GetJsonAsync<RegionalContent>(
        "https://api.game.com/content",
        headers
    );
}
```

---

### **6. Configuration**
#### **Global Settings**
```csharp
void InitializeNetwork()
{
    // Set auth token
    NetworkRequestManager.AuthToken = playerSession.Token;
    
    // Configure retries and timeout
    NetworkRequestManager.SetMaxRetries(5);
    NetworkRequestManager.SetDefaultTimeout(TimeSpan.FromSeconds(15));
    
    // Add default headers
    NetworkRequestManager.SetDefaultHeader("X-Client-Version", Application.version);
}
```

#### **Concurrency Control**
```csharp
// Increase concurrent requests for heavy loading screens
NetworkRequestManager.SetMaxConcurrency(8);
```

---

### **Best Practices**
1. **Cancellation Support**
   ```csharp
   public async Task LoadFriendsList(CancellationToken ct)
   {
       return await NetworkRequestManager.GetJsonAsync<Friend[]>(
           "https://api.game.com/friends",
           cancellationToken: ct
       );
   }
   ```

2. **Progress Reporting**
   ```csharp
   var progress = new Progress<float>(p => 
       downloadProgress.text = $"{p:P0}"
   );
   ```

3. **Secure Token Handling**
   ```csharp
   void OnLogout()
   {
       NetworkRequestManager.AuthToken = null;
       NetworkRequestManager.ClearDefaultHeaders();
   }
   ```

---

### **Troubleshooting**
**Common Issues**:
- **Timeout Errors**: Increase default timeout
  ```csharp
  NetworkRequestManager.SetDefaultTimeout(TimeSpan.FromSeconds(30));
  ```
  
- **JSON Parsing Errors**: Verify DTO structures match API responses

- **Concurrency Limits**: Adjust based on device capability
  ```csharp
  // High-end devices
  NetworkRequestManager.SetMaxConcurrency(10);
  ```

---

### **Performance Tips**
- **Pool Requests**: Queue requests during loading screens
- **Cache Responses**: Implement local caching for static data
- **Batch Operations**: Combine related requests when possible

---

This network manager provides a comprehensive solution for Unity web communication needs. Key features like automatic retries and concurrency control make it robust for production use, while flexible configuration options adapt to various gameplay scenarios.