Here's a comprehensive `README.md` for your `JsonHelper` class:

```markdown
# Unity JSON Helper Utility

A comprehensive JSON utility class for Unity and .NET projects featuring serialization, deserialization, file I/O, schema validation, diff/merge operations, HTTP requests, Firestore integration, and more.

## Features

- Core JSON serialization/deserialization
- File I/O operations (sync & async)
- JSON formatting (pretty print/minify)
- Schema validation
- JSON diff/merge operations
- Path-based access to nested data
- Data cleanup and conversion utilities
- Unity Editor integration
- PlayerPrefs helpers
- HTTP request utilities
- Firebase Firestore integration
- JSON Patch support
- Version migration system
- Custom converter registration
- GZIP compression
- Telemetry hooks

## Installation

1. Ensure you have Newtonsoft.Json (Json.NET) installed via Package Manager
2. For Firebase features, install Firebase SDK
3. Copy `JsonHelper.cs` into your project's `Scripts/Utilities` folder

## Usage Examples

### Core Serialization

```csharp
// Serialize an object
var playerData = new { name = "Hero", level = 5, items = new[] { "sword", "shield" } };
string json = JsonHelper.Serialize(playerData, prettyPrint: true);

// Deserialize to dictionary
var dict = JsonHelper.DeserializeToDictionary(json);
Debug.Log($"Player name: {dict["name"]}");

// Safe deserialization
if (JsonHelper.TryDeserialize(json, out var result, out var error))
{
    // Use result
}
else
{
    Debug.LogError($"Failed to parse: {error}");
}
```

### File Operations

```csharp
// Save to file
JsonHelper.SerializeToFile(playerData, "Assets/Data/player.json");

// Async save
await JsonHelper.SerializeToFileAsync(playerData, "Assets/Data/player_async.json");

// Load from file
var loadedData = JsonHelper.DeserializeFromFile("Assets/Data/player.json");

// Async load
var asyncData = await JsonHelper.DeserializeFromFileAsync("Assets/Data/player_async.json");
```

### Schema Validation

```csharp
string schema = @"{
    'type': 'object',
    'properties': {
        'name': {'type':'string'},
        'level': {'type':'integer'}
    },
    'required': ['name']
}";

try {
    JsonHelper.ValidateSchema(json, schema);
    Debug.Log("JSON is valid!");
}
catch (JsonSchemaException ex) {
    Debug.LogError($"Invalid JSON: {ex.Message}");
}
```

### Diff & Merge

```csharp
var original = JObject.Parse(@"{'name':'Hero','items':['sword']}");
var modified = JObject.Parse(@"{'name':'SuperHero','items':['sword','shield']}");

// Get differences
var diffs = JsonHelper.DiffJson(original, modified);
// Returns: ["/name", "/items"]

// Merge changes
JsonHelper.MergeJson(original, modified);
```

### Path-based Access

```csharp
var complexData = new Dictionary<string, object> {
    ["player"] = new Dictionary<string, object> {
        ["stats"] = new Dictionary<string, object> {
            ["health"] = 100,
            ["items"] = new List<object> { "sword", "potion" }
        }
    }
};

// Get nested value
var health = JsonHelper.GetByPath(complexData, "player/stats/health"); // 100
var item = JsonHelper.GetByPath(complexData, "player/stats/items/0"); // "sword"

// Set nested value
JsonHelper.SetByPath(complexData, "player/stats/health", 150);
```

### Data Conversion

```csharp
// Convert dictionary to strongly-typed object
public class Player {
    public string name;
    public int level;
}

var player = JsonHelper.ToObject<Player>(dict);

// Deep clone
var playerClone = JsonHelper.DeepClone(player);

// Flatten nested structure
var flat = JsonHelper.Flatten(complexData);
// Returns: {
//   "player.stats.health": 100,
//   "player.stats.items.0": "sword",
//   "player.stats.items.1": "potion"
// }
```

### Unity Integration

```csharp
// Save to PlayerPrefs
JsonHelper.SaveToPrefs("PlayerData", playerData);

// Load from PlayerPrefs
var prefsData = JsonHelper.LoadFromPrefs("PlayerData");

#if UNITY_EDITOR
// Create ScriptableObject from JSON
JsonHelper.CreateScriptableObjectFromJson<PlayerConfig>(json, "Assets/Data/PlayerConfig.asset");
#endif
```

### HTTP Requests

```csharp
// POST JSON to API
var response = await JsonHelper.SendJsonAsync(
    "https://api.example.com/players",
    new { name = "NewPlayer" },
    UnityWebRequest.kHttpVerbPOST,
    new Dictionary<string, string> { ["Authorization"] = "Bearer token" }
);

// GET JSON from API
var apiData = await JsonHelper.GetJsonAsync<PlayerData>("https://api.example.com/players/123");
```

### Firebase Integration

```csharp
// Initialize (call once at startup)
await JsonHelper.InitializeFirebaseAsync();

// Write data
await JsonHelper.WriteToFirestoreAsync("players", "player123", playerData);

// Read data
var firestoreData = await JsonHelper.ReadFromFirestoreAsync("players", "player123");

// Update single field
await JsonHelper.UpdateFirestoreFieldAsync("players", "player123", "level", 6);
```

### Advanced Features

```csharp
// JSON Patch
string patch = @"[
    { 'op': 'replace', 'path': '/name', 'value': 'UpdatedName' },
    { 'op': 'add', 'path': '/items/-', 'value': 'bow' }
]";
JsonHelper.ApplyJsonPatch(ref playerData, patch);

// Version Migration
JsonHelper.RegisterMigration(1, data => {
    data["version"] = 2;
    data["legacyField"] = "converted value";
    return data;
});

var migrated = JsonHelper.Migrate(oldData, 1, 2);

// Custom Converters
public class Vector3Converter : JsonConverter<Vector3> {
    // Implementation omitted
}
JsonHelper.RegisterConverter(new Vector3Converter());

// Compression
byte[] compressed = JsonHelper.CompressJson(largeJson);
string decompressed = JsonHelper.DecompressJson(compressed);
```

## Best Practices

1. **Error Handling**: Always wrap deserialization in try-catch blocks
2. **Async Operations**: Use async methods for file I/O and network requests
3. **Schema Validation**: Validate incoming JSON when working with external sources
4. **Firebase**: Initialize Firebase only once at application startup
5. **Memory**: Use compression for large JSON payloads
6. **Telemetry**: Use the event hooks for performance monitoring

## Dependencies

- Newtonsoft.Json (Json.NET) 13.0+
- Unity 2018.4+ (some features require newer versions)
- Firebase SDK (for Firestore features)
- Marvin.JsonPatch (for JSON Patch support)

```

The examples are organized by functionality and include both basic and advanced use cases. Each section demonstrates practical applications of the methods with realistic scenarios.

