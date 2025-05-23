Here's a detailed walkthrough of your `FirebaseHelper` class with real-world examples for each component:

---

### **1. Initialization**
#### `InitializeAsync()`
**Purpose**: Sets up Firebase and Firestore  
**Example**: Initialize at game start
```csharp
async void Start() {
    bool success = await FirebaseHelper.InitializeAsync();
    if (success) {
        Debug.Log("Firebase ready!");
        // Now safe to use other FirebaseHelper methods
    }
}
```

---

### **2. Document Operations**
#### `SetDocumentAsync<T>()`
**Purpose**: Create/update documents with automatic serialization  
**Example**: Save player profile
```csharp
public async Task SavePlayerProfile(string userId) {
    PlayerProfile profile = new PlayerProfile {
        Name = "Warrior",
        Level = 25,
        LastLogin = DateTime.UtcNow
    };
    
    bool success = await FirebaseHelper.SetDocumentAsync(
        "players", 
        userId, 
        profile
    );
}
```

#### `GetDocumentAsync<T>()`
**Purpose**: Retrieve and deserialize documents  
**Example**: Load game settings
```csharp
public async Task LoadGraphicsSettings() {
    GraphicsSettings settings = await FirebaseHelper.GetDocumentAsync<GraphicsSettings>(
        "config", 
        "graphics"
    );
    
    if (settings != null) {
        QualitySettings.SetQualityLevel(settings.QualityPreset);
    }
}
```

#### `UpdateDocumentAsync()`
**Purpose**: Partial document updates  
**Example**: Update player currency
```csharp
public async Task AddCoins(string playerId, int amount) {
    var updates = new Dictionary<string, object> {
        {"currency.coins", FieldValue.Increment(amount)},
        {"updatedAt", FieldValue.ServerTimestamp}
    };
    
    await FirebaseHelper.UpdateDocumentAsync(
        "players",
        playerId,
        updates
    );
}
```

#### `DeleteDocumentAsync()`
**Purpose**: Remove documents  
**Example**: Delete obsolete save file
```csharp
public async Task DeleteOldSave(string saveSlotId) {
    await FirebaseHelper.DeleteDocumentAsync(
        "saves",
        saveSlotId
    );
}
```

---

### **3. Query Operations**
#### `QueryDocumentsAsync<T>()`
**Purpose**: Complex document queries  
**Example**: Find high-level players
```csharp
public async Task<List<PlayerProfile>> GetTopPlayers() {
    var conditions = new List<QueryCondition> {
        new QueryCondition {
            Field = "level",
            Operator = QueryOperator.GreaterThanOrEqualTo,
            Value = 50
        }
    };
    
    return await FirebaseHelper.QueryDocumentsAsync<PlayerProfile>(
        "players",
        conditions,
        limit: 10
    );
}
```

---

### **4. Real-time Listeners**
#### `ListenToDocument()`
**Purpose**: Real-time data updates  
**Example**: Live leaderboard updates
```csharp
void SetupLeaderboardListener(string leaderboardId) {
    FirebaseHelper.ListenToDocument<Leaderboard>(
        "leaderboards",
        leaderboardId,
        update => {
            // Refresh UI when data changes
            UpdateLeaderboardUI(update.Entries);
        },
        error => {
            Debug.LogError($"Leaderboard sync error: {error}");
        }
    );
}

// When leaving leaderboard screen
void OnDestroy() {
    FirebaseHelper.RemoveListener("leaderboards/top_scores");
}
```

---

### **5. Batch Operations**
#### `ExecuteBatchAsync()`
**Purpose**: Atomic write operations  
**Example**: Reset daily challenges
```csharp
public async Task ResetDailyChallenges() {
    var operations = new List<BatchOperation> {
        new BatchOperation {
            Collection = "challenges",
            DocumentId = "daily",
            Type = BatchOperationType.Set,
            Data = new Dictionary<string, object> {
                {"resetAt", DateTime.UtcNow},
                {"entries", new List<Challenge>()}
            }
        },
        new BatchOperation {
            Collection = "metadata",
            DocumentId = "counters",
            Type = BatchOperationType.Update,
            Data = new Dictionary<string, object> {
                {"dailyResets", FieldValue.Increment(1)}
            }
        }
    };
    
    await FirebaseHelper.ExecuteBatchAsync(operations);
}
```

---

### **6. File Operations Integration**
#### `UploadFileWithMetadataAsync()`
**Purpose**: Store files with metadata  
**Example**: Player screenshot sharing
```csharp
public async Task UploadPlayerScreenshot(Texture2D screenshot) {
    byte[] bytes = screenshot.EncodeToPNG();
    
    // Implementation would need Firebase Storage:
    // var reference = FirebaseStorage.DefaultInstance
    //     .Reference.Child("screenshots/user123.png");
    // await reference.PutBytesAsync(bytes);
    
    var metadata = new Dictionary<string, object> {
        {"description", "Epic boss fight!"},
        {"location", "castle_dungeon"},
        {"likes", 0}
    };
    
    await FirebaseHelper.UploadFileWithMetadataAsync(
        "screenshots/user123.png",
        "screenshots",
        "user123_20230815",
        metadata
    );
}
```

---

### **7. Data Synchronization**
#### `SyncRemoteJsonWithFirestoreAsync()`
**Purpose**: Keep Firestore in sync with external APIs  
**Example**: Daily content updates
```csharp
public async Task SyncNewsFeed() {
    await FirebaseHelper.SyncRemoteJsonWithFirestoreAsync(
        "https://game-api.com/daily-news",
        "content",
        "daily_news"
    );
}
```

---

### **Key Components Explained**

1. **QueryCondition Class**  
   Builds complex filters for Firestore queries:
   ```csharp
   // Find players with >=1000 coins in Asia region
   new List<QueryCondition> {
       new QueryCondition { Field = "currency.coins", Operator = GreaterThanOrEqualTo, Value = 1000 },
       new QueryCondition { Field = "region", Operator = EqualTo, Value = "Asia" }
   }
   ```

2. **BatchOperation System**  
   Enables atomic writes across multiple documents:
   ```csharp
   // Simultaneously update player inventory and quest progress
   var ops = new List<BatchOperation> {
       new BatchOperation {
           Collection = "players", 
           DocumentId = "user123",
           Type = Update,
           Data = new Dictionary<string, object> { {"quests.active", "q789"} }
       },
       new BatchOperation {
           Collection = "inventories",
           DocumentId = "user123",
           Type = Update,
           Data = new Dictionary<string, object> { {"items.sword", 1} }
       }
   };
   ```

3. **Listener Management**  
   Handles real-time subscriptions:
   ```csharp
   // Multiplayer game session tracking
   void JoinMatch(string matchId) {
       FirebaseHelper.ListenToDocument<MatchState>(
           "matches",
           matchId,
           state => {
               UpdateGameState(state);
               SyncPlayerPositions(state.Participants);
           }
       );
   }

   void LeaveMatch() {
       FirebaseHelper.RemoveListener($"matches/{currentMatchId}");
   }
   ```

---

### **Best Practices**

1. **Error Handling**
   ```csharp
   try {
       await FirebaseHelper.SetDocumentAsync(...);
   }
   catch (FirebaseException fbEx) {
       Debug.LogError($"Firestore error: {fbEx.ErrorCode}");
   }
   catch (Exception ex) {
       Debug.LogError($"General error: {ex.Message}");
   }
   ```

2. **Performance Optimization**
   ```csharp
   // Use FieldMask for partial updates
   await UpdateDocumentAsync("players", "user123", new Dictionary<string, object> {
       {"health", 100},
       {"position.x", 120},
       {"position.y", 45}
   });
   ```

3. **Security Rules**  
   Always pair with proper Firestore security rules:
   ```javascript
   match /players/{userId} {
       allow read, write: if request.auth.uid == userId;
   }
   ```

4. **Batching**  
   Group related operations:
   ```csharp
   // Daily reset batch
   var batch = new List<BatchOperation> {
       ResetPlayerEnergyOp(),
       ClearDailyQuestsOp(),
       GenerateNewItemsOp()
   };
   await ExecuteBatchAsync(batch);
   ```

---

This comprehensive helper abstracts Firestore complexity while maintaining flexibility. Each method handles Firebase's asynchronous nature and provides Unity-specific error handling. The integration with your existing JSON system makes it particularly powerful for complex game data scenarios.