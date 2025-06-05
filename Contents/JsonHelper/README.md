Below is a complete breakdown of **JsonHelper**—a single static class that brings together a wide array of JSON‐related capabilities for Unity/.NET, including:

* Core serialization & deserialization
* Converting JSON to/from `Dictionary<string, object>` with nested parsing
* Safe “try‐deserialize” patterns
* Synchronous and asynchronous file I/O (read/write JSON files)
* Pretty‐print and minification utilities
* Schema validation hooks (commented out)
* Merging and diffing of JSON objects (`JObject`)
* Path‐based get/set on nested dictionaries
* Removing nulls and flattening deeply nested dictionaries
* Converting between `Dictionary<string, object>` and POCOs (plain C# objects)
* Deep‐clone via JSON roundtrip
* Unity‐specific helpers (ScriptableObject creation, PlayerPrefs storage)
* HTTP requests with JSON payloads using `UnityWebRequest`
* (Commented out) Firebase Firestore integration
* (Commented out) JSON patch (RFC6902) support stub
* Versioning/migration hooks for dictionary schemas
* Custom `JsonConverter` registration
* GZIP compression/decompression
* Simple telemetry events around serialization

Use **JsonHelper** whenever you need robust JSON manipulations in Unity: reading/writing JSON files, merging two JSON payloads, comparing them, persisting data to PlayerPrefs, performing HTTP calls to a JSON‐based REST API, or generating ScriptableObjects from JSON at edit time.

---

## Table of Contents

1. [Overview & Purpose](#overview--purpose)
2. [Core Serialize / Deserialize](#core-serialize--deserialize)

   * 2.1. `Serialize(object data, bool prettyPrint = false)`
   * 2.2. `DeserializeToDictionary(string json)` & Helpers
   * 2.3. `TryDeserialize(string json, out Dictionary<string, object> result, out string error)`
3. [File I/O](#file-io)

   * 3.1. `SerializeToFile(object data, string filePath, bool prettyPrint = false)`
   * 3.2. `DeserializeFromFile(string filePath)`
   * 3.3. `SerializeToFileAsync(...)` & `DeserializeFromFileAsync(...)`
4. [Formatting & Minification](#formatting--minification)

   * 4.1. `PrettyPrint(string json)`
   * 4.2. `Minify(string json)`
   * 4.3. Generic `Deserialize<T>(string json)`
5. [Schema Validation (Commented Out)](#schema-validation-commented-out)
6. [Merge & Diff](#merge--diff)

   * 6.1. `MergeJson(JObject dest, JObject src)`
   * 6.2. `DiffJson(JToken a, JToken b)`
7. [Path‐Based Access in Nested Dictionaries](#path-based-access-in-nested-dictionaries)

   * 7.1. `GetByPath(Dictionary<string, object> dict, string path)`
   * 7.2. `SetByPath(Dictionary<string, object> dict, string path, object value)`
8. [Cleanup & Conversion](#cleanup--conversion)

   * 8.1. `RemoveNulls(object obj)`
   * 8.2. `ToObject<T>(Dictionary<string, object> dict)`
   * 8.3. `DeepClone<T>(T obj)`
   * 8.4. `Flatten(Dictionary<string, object> source, string parentKey = "", char separator = '.')`
   * 8.5. `ContainsPath(Dictionary<string, object> dict, string path)`
9. [Unity Editor & PlayerPrefs Helpers](#unity-editor--playerprefs-helpers)

   * 9.1. `CreateScriptableObjectFromJson<T>(string json, string assetPath)` (Editor only)
   * 9.2. `SaveToPrefs(string key, object obj)`
   * 9.3. `LoadFromPrefs(string key)`
10. [HTTP Requests (UnityWebRequest)](#http-requests-unitywebrequest)

    * 10.1. `SendJsonAsync(string url, object payload, string method = POST, Dictionary<string,string> headers = null)`
    * 10.2. `GetJsonAsync<T>(string url, Dictionary<string,string> headers = null)`
11. [Versioning / Migration Hooks](#versioning--migration-hooks)

    * 11.1. `RegisterMigration(int version, Func<Dictionary<string, object>, Dictionary<string, object>> migrate)`
    * 11.2. `Migrate(Dictionary<string, object> dict, int fromVersion, int toVersion)`
12. [Custom Converter Registration](#custom-converter-registration)

    * 12.1. `RegisterConverter(JsonConverter converter)`
    * 12.2. Internal `CreateSettings(bool prettyPrint)`
13. [Compression Helpers (GZIP)](#compression-helpers-gzip)

    * 13.1. `CompressJson(string json)`
    * 13.2. `DecompressJson(byte[] compressed)`
14. [Telemetry Hooks](#telemetry-hooks)

    * 14.1. `OnOperationStart` & `OnOperationComplete` events
    * 14.2. `SerializeWithEvents(object data, bool prettyPrint = false)`
15. [Real‐World Usage Examples](#real-world-usage-examples)

    * 15.1. Serialize a GameData object to JSON string
    * 15.2. Read/write JSON file on disk (sync & async)
    * 15.3. Convert JSON to `Dictionary<string, object>` and navigate nested fields
    * 15.4. Merge two JSON objects and save result
    * 15.5. Compare differences between two JSON payloads
    * 15.6. Get or set a deeply nested value by path
    * 15.7. Remove all nulls and flatten a nested dictionary
    * 15.8. Deep‐clone an object via JSON roundtrip
    * 15.9. Save custom data to PlayerPrefs and load it back
    * 15.10. Perform a POST of JSON payload to a REST API and read the response
    * 15.11. Compress a JSON string before saving (and decompress after loading)
    * 15.12. Apply version‐based migrations to a dictionary
    * 15.13. Register a custom converter (e.g., for Unity’s `Vector3`)
    * 15.14. Wrap serialization in telemetry events
16. [Best Practices & Tips](#best-practices--tips)
17. [Summary of Public API](#summary-of-public-api)

---

## 1. Overview & Purpose

```csharp
/// <summary>
/// A comprehensive JSON utility class for Unity and .NET projects,
/// featuring serialization, deserialization, file I/O, schema validation,
/// diff/merge, HTTP requests, Firestore integration, and more.
/// </summary>
public static class JsonHelper
{
    // … (methods described below) …
}
```

**JsonHelper** is intended to unify all common JSON‐centered tasks under one roof, including:

* **Core Serialization/Deserialization**:

  * `Serialize(object)` → JSON string
  * `DeserializeToDictionary(string)` → `Dictionary<string, object>`
  * `TryDeserialize(...)` that doesn’t throw exceptions

* **File I/O**: Synchronous and asynchronous JSON read/write to disk.

* **Formatting**: Pretty‐print or minify JSON strings.

* **Schema Validation** (stubbed): Example code commented out using `Newtonsoft.Json.Schema`.

* **Merge** two `JObject`s with custom merge rules.

* **Diff** between two `JToken`s to get a list of differing paths.

* **Path‐Based Access**: Get or set a value in a nested `Dictionary<string, object>` by using a slash‐ or dot‐delimited path like `"player.inventory.2.name"`.

* **Cleanup & Conversion**:

  * Remove all null values from a nested dictionary/lists.
  * Convert a `Dictionary<string, object>` to a typed object `T` via JSON serialization.
  * Deep‐clone any object via JSON roundtrip.
  * Flatten nested dictionaries into a single‐level dictionary with compound keys.

* **Unity Integration**:

  * In the Editor, create a `ScriptableObject` from JSON and save it as an asset.
  * Save/load any object to `PlayerPrefs` via JSON serialization.

* **HTTP Calls** with JSON payloads using `UnityWebRequest`, returning `Task<T>` for GET, or `UnityWebRequestAsyncOperation` for POST/PUT.

* **Versioning & Migrations**: Register migration functions by version number and apply them to dictionaries that represent older data formats.

* **Custom Converters**: Easily register custom `JsonConverter` instances (e.g., for `DateTime`, `Enum`, or Unity types like `Vector3`).

* **Compression**: Compress JSON to GZIP bytes and decompress back to string.

* **Telemetry Hooks**: Fire simple events (`OnOperationStart`, `OnOperationComplete`) around serialization calls for logging or profiling.

Use **JsonHelper** whenever you need a “one‐stop” JSON solution in Unity/.NET. The examples in Section 15 show how to use each major feature in real projects.

---

## 2. Core Serialize / Deserialize

### 2.1 Method: `public static string Serialize(object data, bool prettyPrint = false)`

```csharp
public static string Serialize(object data, bool prettyPrint = false)
{
    var settings = CreateSettings(prettyPrint);
    return JsonConvert.SerializeObject(data, settings);
}
```

**Purpose**:
Convert any .NET/Unity object (`data`) into a JSON‐formatted string using `Newtonsoft.Json`. Internally calls `CreateSettings(prettyPrint)` to include registered custom converters (plus a `StringEnumConverter`) and configure formatting.

**Parameters**:

* `data` (`object`): The object graph to serialize (e.g., a `PlayerData` instance, a dictionary, a list, etc.).
* `prettyPrint` (`bool`, default `false`): If `true`, produces indented JSON (4‐space by default) for human readability; otherwise, all formatting removed.

**Returns**:
A `string` containing valid JSON. If your object contains circular references, they will be ignored (`ReferenceLoopHandling = Ignore` in settings).

---

### 2.2 Method: `public static Dictionary<string, object> DeserializeToDictionary(string json)`

```csharp
public static Dictionary<string, object> DeserializeToDictionary(string json)
{
    if (string.IsNullOrWhiteSpace(json))
        throw new ArgumentException("JSON string is null or empty.", nameof(json));

    var token = JToken.Parse(json);
    if (token.Type != JTokenType.Object)
        throw new InvalidOperationException("JSON root is not an object.");

    return ParseJObject((JObject)token);
}
```

**Purpose**:
Parse any JSON object (top‐level must be an object/dictionary) into a `Dictionary<string, object>`, where:

* Nested JSON objects become nested `Dictionary<string, object>`.
* JSON arrays become `List<object>`.
* Primitives (`int`, `float`, `string`, `bool`, `DateTime`, `byte[]`) are converted to their corresponding .NET types.
* Any other token type falls back to `token.ToString()`.

**Exceptions**:

* Throws `ArgumentException` if `json` is null/empty/whitespace.
* Throws `InvalidOperationException` if the JSON root is not an object (e.g., a JSON array or primitive).

**Helpers**:

* `ParseJObject(JObject jo)` → iterates properties, calls `ParseToken()` on each value.
* `ParseToken(JToken token)` → recursively handles `JObject`, `JArray`, or returns appropriate .NET type.

---

#### 2.2.1 Helper: `private static Dictionary<string, object> ParseJObject(JObject jo)`

```csharp
private static Dictionary<string, object> ParseJObject(JObject jo)
{
    var dict = new Dictionary<string, object>(StringComparer.OrdinalIgnoreCase);
    foreach (var prop in jo.Properties())
        dict[prop.Name] = ParseToken(prop.Value);
    return dict;
}
```

* Creates a new case‐insensitive dictionary.
* For each property in the `JObject`, calls `ParseToken` to convert the `JToken` into a .NET object (nested dictionary, list, or primitive).

---

#### 2.2.2 Helper: `private static object ParseToken(JToken token)`

```csharp
private static object ParseToken(JToken token)
{
    switch (token.Type)
    {
        case JTokenType.Object:  return ParseJObject((JObject)token);
        case JTokenType.Array:   return token.Select(ParseToken).ToList();
        case JTokenType.Integer: return token.Value<long>();
        case JTokenType.Float:   return token.Value<double>();
        case JTokenType.String:  return token.Value<string>();
        case JTokenType.Boolean: return token.Value<bool>();
        case JTokenType.Null:    return null;
        case JTokenType.Date:    return token.Value<DateTime>();
        case JTokenType.Bytes:   return token.Value<byte[]>();
        default:                 return token.ToString();
    }
}
```

* **`JTokenType.Object`** → recursively call `ParseJObject`.
* **`JTokenType.Array`** → convert to `List<object>` by `token.Select(ParseToken).ToList()`.
* **Numeric types** (`Integer` or `Float`) → convert to `long` or `double` respectively.
* **Strings** → `string`.
* **Boolean** → `bool`.
* **Null** → `null`.
* **Date** → `DateTime`.
* **Bytes** → `byte[]`.
* **Default**: return `token.ToString()` (e.g., for `Guid`, `Uri`, or unknown types).

---

### 2.3 Method: `public static bool TryDeserialize(string json, out Dictionary<string, object> result, out string error)`

```csharp
public static bool TryDeserialize(string json, out Dictionary<string, object> result, out string error)
{
    result = null; error = null;
    try
    {
        result = DeserializeToDictionary(json);
        return true;
    }
    catch (Exception ex)
    {
        error = ex.Message;
        return false;
    }
}
```

**Purpose**:
Safely attempt to parse a JSON string into a dictionary. If parsing succeeds, returns `true` and sets `result`. If any exception occurs (invalid JSON, not an object, etc.), captures the exception’s `Message` in `error` and returns `false`.

**Usage**:

```csharp
string json = /* some user input */;
if (JsonHelper.TryDeserialize(json, out var dict, out var errorMsg))
{
    // Use dict safely
}
else
{
    Debug.LogError($"Failed to parse JSON: {errorMsg}");
}
```

---

## 3. File I/O

### 3.1 Method: `public static void SerializeToFile(object data, string filePath, bool prettyPrint = false)`

```csharp
public static void SerializeToFile(object data, string filePath, bool prettyPrint = false)
{
    var json = Serialize(data, prettyPrint);
    var dir = Path.GetDirectoryName(filePath);
    if (!Directory.Exists(dir)) Directory.CreateDirectory(dir);
    File.WriteAllText(filePath, json);
}
```

**Purpose**:
Serialize `data` (any object) to JSON (optionally pretty‐printed) and save it to `filePath` synchronously. Creates parent directories if they don’t exist.

**Parameters**:

* `data` (`object`): The object to serialize.
* `filePath` (`string`): Absolute or relative file path (e.g., `Application.persistentDataPath + "/gameData.json"`).
* `prettyPrint` (`bool`): If `true`, writes indented JSON.

**Example**:

```csharp
PlayerData pdata = new PlayerData { Level = 5, Score = 12345 };
JsonHelper.SerializeToFile(pdata, Application.persistentDataPath + "/player.json", prettyPrint: true);
```

---

### 3.2 Method: `public static Dictionary<string, object> DeserializeFromFile(string filePath)`

```csharp
public static Dictionary<string, object> DeserializeFromFile(string filePath)
{
    if (!File.Exists(filePath))
        throw new FileNotFoundException($"JSON file not found: {filePath}");
    return DeserializeToDictionary(File.ReadAllText(filePath));
}
```

**Purpose**:
Read the entire contents of `filePath` synchronously and parse it into a dictionary using `DeserializeToDictionary`.

**Exceptions**:

* Throws `FileNotFoundException` if `filePath` does not exist.
* Propagates any JSON parse exceptions from `DeserializeToDictionary`.

**Example**:

```csharp
string path = Application.persistentDataPath + "/player.json";
var dict = JsonHelper.DeserializeFromFile(path);
int level = (int)(long)dict["Level"]; // integer values come back as long
```

---

### 3.3 Method: `public static async Task SerializeToFileAsync(object data, string filePath, bool prettyPrint = false, CancellationToken ct = default)`

```csharp
public static async Task SerializeToFileAsync(object data, string filePath, bool prettyPrint = false, CancellationToken ct = default)
{
    var json = Serialize(data, prettyPrint);
    var dir = Path.GetDirectoryName(filePath);
    if (!Directory.Exists(dir)) Directory.CreateDirectory(dir);
    await File.WriteAllTextAsync(filePath, json, ct).ConfigureAwait(false);
}
```

**Purpose**:
Asynchronously serialize `data` to JSON and write it to `filePath`. Honors a `CancellationToken` if provided.

**Behavior**:

1. `Serialize(data, prettyPrint)` creates the JSON string.
2. Ensure parent directory exists.
3. `await File.WriteAllTextAsync(filePath, json, ct)` writes the file in a background thread.
4. `.ConfigureAwait(false)` ensures that after the await, continuation does not attempt to marshal back to Unity’s main thread (but for file I/O this usually doesn’t matter).

**Example**:

```csharp
async Task SaveSettingsAsync(Settings s)
{
    string path = Path.Combine(Application.persistentDataPath, "settings.json");
    var cts = new CancellationTokenSource();
    try
    {
        await JsonHelper.SerializeToFileAsync(s, path, true, cts.Token);
        Debug.Log("Settings saved asynchronously.");
    }
    catch (OperationCanceledException)
    {
        Debug.LogWarning("Save cancelled.");
    }
}
```

---

### 3.4 Method: `public static async Task<Dictionary<string, object>> DeserializeFromFileAsync(string filePath, CancellationToken ct = default)`

```csharp
public static async Task<Dictionary<string, object>> DeserializeFromFileAsync(string filePath, CancellationToken ct = default)
{
    if (!File.Exists(filePath))
        throw new FileNotFoundException($"JSON file not found: {filePath}");
    var json = await File.ReadAllTextAsync(filePath, ct).ConfigureAwait(false);
    return DeserializeToDictionary(json);
}
```

**Purpose**:
Asynchronously read JSON from `filePath` and parse it into a dictionary. Throws if file is missing.

**Example**:

```csharp
async Task LoadSettingsAsync()
{
    string path = Path.Combine(Application.persistentDataPath, "settings.json");
    var cts = new CancellationTokenSource();
    try
    {
        var dict = await JsonHelper.DeserializeFromFileAsync(path, cts.Token);
        // Convert dict to a Settings object or use directly
        Settings s = JsonHelper.ToObject<Settings>(dict);
    }
    catch (FileNotFoundException)
    {
        Debug.LogWarning("Settings file not found; using defaults.");
    }
}
```

---

## 4. Formatting & Minification

### 4.1 Method: `public static string PrettyPrint(string json)`

```csharp
public static string PrettyPrint(string json) 
    => JToken.Parse(json).ToString(Formatting.Indented);
```

**Purpose**:
Take any valid JSON string and reformat it with indentation (4 spaces per level) so it’s human‐readable. Throws if `json` is invalid.

**Example**:

```csharp
string ugly = "{\"name\":\"Bob\",\"scores\":[10,20,30]}";
string pretty = JsonHelper.PrettyPrint(ugly);
// pretty ==
// {
//   "name": "Bob",
//   "scores": [
//     10,
//     20,
//     30
//   ]
// }
```

---

### 4.2 Method: `public static string Minify(string json)`

```csharp
public static string Minify(string json) 
    => JToken.Parse(json).ToString(Formatting.None);
```

**Purpose**:
Remove all whitespace (indentation, line breaks) from a JSON string, producing one continuous line. Useful for reducing payload size if you’re sending JSON over the wire.

**Example**:

```csharp
string pretty = @"
{
    ""name"": ""Bob"",
    ""scores"": [ 10, 20, 30 ]
}";
string min = JsonHelper.Minify(pretty);  
// min == "{\"name\":\"Bob\",\"scores\":[10,20,30]}"
```

---

### 4.3 Method: `public static T Deserialize<T>(string json)`

```csharp
public static T Deserialize<T>(string json)
{
    if (string.IsNullOrEmpty(json))
        return default;
    return JsonConvert.DeserializeObject<T>(json);
}
```

**Purpose**:
Convenience wrapper around `JsonConvert.DeserializeObject<T>(json)`. If `json` is `null` or empty, returns `default(T)` (e.g., `null` for classes, zero for value types).

**Example**:

```csharp
string json = "{\"Level\":5,\"Name\":\"Hero\"}";
PlayerData pd = JsonHelper.Deserialize<PlayerData>(json);
// pd.Level == 5, pd.Name == "Hero"
```

---

## 5. Schema Validation (Commented Out)

```csharp
// public static void ValidateSchema(string json, string schemaJson)
// {
//     var schema = JSchema.Parse(schemaJson);
//     var token  = JToken.Parse(json);
//     if (!token.IsValid(schema, out IList<string> errors))
//         throw new JsonSchemaException("Schema validation failed: " + string.Join("; ", errors));
// }
```

**Purpose**:
Showcases how to validate a JSON string against a JSON‐Schema using `Newtonsoft.Json.Schema`. Currently commented out—uncomment to enable schema validation in your project after adding the `Newtonsoft.Json.Schema` package.

* **Parameters**:

  * `json`: The JSON to validate.
  * `schemaJson`: The JSON‐Schema as a string (must be a valid JSON‐Schema object).

* **Behavior**:

  * Parse `schemaJson` into a `JSchema`.
  * Parse `json` into a `JToken`.
  * `token.IsValid(schema, out errors)` checks validity; if invalid, throws `JsonSchemaException` listing all errors.

---

## 6. Merge & Diff

### 6.1 Method: `public static void MergeJson(JObject dest, JObject src)`

```csharp
public static void MergeJson(JObject dest, JObject src)
{
    dest.Merge(src, new JsonMergeSettings {
        MergeArrayHandling = MergeArrayHandling.Concat,
        MergeNullValueHandling = MergeNullValueHandling.Ignore
    });
}
```

**Purpose**:
Merge all properties from `src` into `dest` in‐place, using these rules:

* **`MergeArrayHandling = Concat`**: If both `dest` and `src` have arrays at a given path, the source array’s elements are concatenated to the destination array.
* **`MergeNullValueHandling = Ignore`**: If `src` has a property whose value is `null`, do not overwrite the existing value in `dest`.

**Example**:

```csharp
JObject o1 = JObject.Parse("{ \"tags\": [\"a\",\"b\"], \"name\": \"Alice\" }");
JObject o2 = JObject.Parse("{ \"tags\": [\"c\"], \"age\": 30, \"name\": null }");

JsonHelper.MergeJson(o1, o2);
// o1 now becomes:
// {
//   "tags": [ "a", "b", "c" ],  // concatenated
//   "name": "Alice",           // name not overwritten because src.name is null
//   "age": 30
// }
```

---

### 6.2 Method: `public static List<string> DiffJson(JToken a, JToken b)`

```csharp
public static List<string> DiffJson(JToken a, JToken b)
{
    var diffs = new List<string>();
    void Recurse(JToken x, JToken y, string path)
    {
        if (!JToken.DeepEquals(x, y))
        {
            if (x.Type == JTokenType.Object && y.Type == JTokenType.Object)
            {
                var props = new HashSet<string>(((JObject)x).Properties().Select(p => p.Name)
                    .Concat(((JObject)y).Properties().Select(p => p.Name)));
                foreach (var prop in props)
                    Recurse(x[prop], y[prop], $"{path}/{prop}");
            }
            else diffs.Add(path);
        }
    }
    Recurse(a, b, string.Empty);
    return diffs;
}
```

**Purpose**:
Compare two `JToken`s (usually `JObject`s) deeply and return a list of “paths” where they differ. Paths are built by joining property names with `/`. If a difference is found at a non‐object leaf, record that path. If both are objects, recurse into each union of property names.

**Behavior**:

1. Initialize `diffs` as an empty list.
2. Define a local function `Recurse(JToken x, JToken y, string path)`.

   * If `DeepEquals(x, y)` is `false`:

     * If both `x` and `y` are objects, collect all property names from `x` and `y`, then call `Recurse(x[prop], y[prop], $"{path}/{prop}")` for each property.
     * Otherwise, add the current `path` to `diffs` (since at least one value differs or one is missing).
3. Call `Recurse(a, b, string.Empty)`.
4. Return `diffs`.

**Example**:

```csharp
JObject o1 = JObject.Parse("{ \"name\": \"Alice\", \"scores\": [10,20], \"stats\": { \"hp\":100 } }");
JObject o2 = JObject.Parse("{ \"name\": \"Alice\", \"scores\": [10,25], \"stats\": { \"hp\":90 }, \"level\": 5 }");

List<string> diffs = JsonHelper.DiffJson(o1, o2);
// diffs might contain:
// [ "/scores", "/stats/hp", "/level" ]
```

* `/scores` differs because arrays `[10,20]` vs `[10,25]`.
* `/stats/hp` differs (100 vs 90).
* `/level` only exists in `o2`, not in `o1` (so it’s considered different).

---

## 7. Path‐Based Access in Nested Dictionaries

### 7.1 Method: `public static object GetByPath(Dictionary<string, object> dict, string path)`

```csharp
public static object GetByPath(Dictionary<string, object> dict, string path)
{
    var segs = path.TrimStart('/').Split(new[]{'.','/'}, StringSplitOptions.RemoveEmptyEntries);
    object cur = dict;
    foreach (var seg in segs)
    {
        if (cur is Dictionary<string, object> d && d.TryGetValue(seg, out var nxt))
            cur = nxt;
        else if (cur is List<object> l && int.TryParse(seg, out var idx) && idx < l.Count)
            cur = l[idx];
        else
            throw new KeyNotFoundException($"Path not found: {path}");
    }
    return cur;
}
```

**Purpose**:
Navigate a nested data structure of arbitrary depth, where:

* JSON objects are stored as `Dictionary<string, object>`.
* JSON arrays are stored as `List<object>`.
* Path segments can be separated by `.` or `/`.
* If a segment corresponds to a dictionary key, drill into that value;
* If it corresponds to an array index (parsable `int`), drill into that list.
* Otherwise, throw `KeyNotFoundException`.

**Parameters**:

* `dict`: The root dictionary (parsed from JSON).
* `path`: A string like `"player.inventory.2.name"` or `"/settings/audio/volume"`.

**Example**:

```csharp
var json = @"{
    ""player"": {
        ""name"": ""Alice"",
        ""inventory"": [
            { ""itemName"": ""Sword"", ""damage"": 10 },
            { ""itemName"": ""Shield"", ""defense"": 5 },
            { ""itemName"": ""Potion"", ""heal"": 20 }
        ]
    }
}";
var rootDict = JsonHelper.DeserializeToDictionary(json);
object potionName = JsonHelper.GetByPath(rootDict, "player.inventory.2.itemName");
// potionName == "Potion"
```

---

### 7.2 Method: `public static void SetByPath(Dictionary<string, object> dict, string path, object value)`

```csharp
public static void SetByPath(Dictionary<string, object> dict, string path, object value)
{
    var segs = path.TrimStart('/').Split(new[]{'.','/'}, StringSplitOptions.RemoveEmptyEntries);
    object cur = dict;
    for (int i = 0; i < segs.Length - 1; i++)
    {
        var seg = segs[i];
        if (cur is Dictionary<string, object> d)
        {
            if (!d.TryGetValue(seg, out var nxt) || nxt == null)
            {
                nxt = new Dictionary<string, object>();
                d[seg] = nxt;
            }
            cur = nxt;
        }
        else
            throw new InvalidOperationException($"Cannot traverse path segment '{seg}'");
    }
    if (cur is Dictionary<string, object> final)
        final[segs.Last()] = value;
    else
        throw new InvalidOperationException($"Cannot set path on non-object at '{segs.Last()}'");
}
```

**Purpose**:
Set the value at a nested path in a dictionary structure. If intermediate nodes don’t exist or are null, creates new empty dictionaries to allow writing the final value.

**Behavior**:

1. Split `path` into segments `segs`.
2. Traverse `cur` starting at `dict`, for each segment except the last:

   * If `cur` is a `Dictionary<string, object> d`, attempt to get `d[seg]`.

     * If missing or `null`, create a new `Dictionary<string, object>` and assign to `d[seg]`.
     * Then set `cur = d[seg]`.
   * Otherwise, throw `InvalidOperationException` (cannot traverse a non‐dictionary object).
3. After the loop, `cur` should be a dictionary; set its `[segs.Last()] = value`.

   * If it’s not a dictionary, throw `InvalidOperationException`.

**Example**:

```csharp
var dict = new Dictionary<string, object>();
JsonHelper.SetByPath(dict, "settings.audio.volume", 0.8f);
// Now dict == { "settings": { "audio": { "volume": 0.8 } } }

JsonHelper.SetByPath(dict, "settings.display.fullscreen", true);
// dict == { "settings": { "audio": { "volume": 0.8 }, "display": { "fullscreen": true } } }
```

---

## 8. Cleanup & Conversion

### 8.1 Method: `public static void RemoveNulls(object obj)`

```csharp
public static void RemoveNulls(object obj)
{
    switch (obj)
    {
        case Dictionary<string, object> d:
            foreach (var key in d.Keys.ToList())
            {
                var v = d[key];
                if (v == null)
                    d.Remove(key);
                else
                {
                    RemoveNulls(v);
                    if (v is IDictionary<string,object> dd && dd.Count == 0) d.Remove(key);
                    if (v is IList<object> ll && ll.Count == 0) d.Remove(key);
                }
            }
            break;

        case List<object> l:
            for (int i = l.Count - 1; i >= 0; i--)
            {
                var v = l[i];
                if (v == null)
                    l.RemoveAt(i);
                else
                {
                    RemoveNulls(v);
                    if (v is IDictionary<string,object> dd2 && dd2.Count == 0) l.RemoveAt(i);
                    if (v is IList<object> ll2 && ll2.Count == 0) l.RemoveAt(i);
                }
            }
            break;
    }
}
```

**Purpose**:
Traverse a nested JSON‐derived structure (dictionaries and lists) and remove:

* Any dictionary key whose value is `null`.
* Any dictionary key whose value, after recursion, becomes an empty dictionary or empty list.
* Any list element that is `null`, or after recursion becomes an empty dictionary or empty list.

**Example**:

```csharp
var json = @"{ 
    ""a"": null, 
    ""b"": { ""x"": null, ""y"": 5 }, 
    ""c"": [ null, { ""z"": null }, 10 ]
}";
var dict = JsonHelper.DeserializeToDictionary(json);
JsonHelper.RemoveNulls(dict);
// Now dict == { "b": { "y": 5 }, "c": [ 10 ] }
```

---

### 8.2 Method: `public static T ToObject<T>(Dictionary<string, object> dict)`

```csharp
public static T ToObject<T>(Dictionary<string, object> dict)
{
    var json = JsonConvert.SerializeObject(dict);
    return JsonConvert.DeserializeObject<T>(json);
}
```

**Purpose**:
Convert a `Dictionary<string, object>`—often produced by `DeserializeToDictionary`—into a strongly‐typed object `T`. Internally:

1. Re‐serialize the dictionary to a JSON string.
2. Deserialize that JSON into type `T`.

**Constraints**:

* `T` must be something that `JsonConvert.DeserializeObject<T>` can handle (e.g., a POCO with public properties matching the JSON structure).

**Example**:

```csharp
var json = @"{ ""Level"": 3, ""Name"": ""Bob"", ""Stats"": { ""hp"": 100, ""mp"": 50 } }";
var dict = JsonHelper.DeserializeToDictionary(json);
PlayerData pd = JsonHelper.ToObject<PlayerData>(dict);
// Now pd.Level == 3, pd.Name == "Bob", pd.Stats.hp == 100, pd.Stats.mp == 50
```

---

### 8.3 Method: `public static T DeepClone<T>(T obj)`

```csharp
public static T DeepClone<T>(T obj)
{
    var json = JsonConvert.SerializeObject(obj);
    return JsonConvert.DeserializeObject<T>(json);
}
```

**Purpose**:
Produce a deep copy of any serializable object via a JSON roundtrip. Unlike `MemberwiseClone`, this duplicates the entire object graph (assuming no cycles), ensuring no shared references remain.

**Example**:

```csharp
GameSettings original = new GameSettings { Resolution = "1920x1080", Quality = "High" };
GameSettings copy = JsonHelper.DeepClone(original);
copy.Quality = "Low";
// original.Quality still == "High"
```

---

### 8.4 Method: `public static Dictionary<string, object> Flatten(Dictionary<string, object> source, string parentKey = "", char separator = '.')`

```csharp
public static Dictionary<string, object> Flatten(Dictionary<string, object> source, string parentKey = "", char separator = '.')
{
    var res = new Dictionary<string, object>();
    foreach (var kvp in source)
    {
        var key = string.IsNullOrEmpty(parentKey)
            ? kvp.Key
            : parentKey + separator + kvp.Key;

        if (kvp.Value is Dictionary<string, object> nd)
        {
            foreach (var n in Flatten(nd, key, separator))
                res[n.Key] = n.Value;
        }
        else
        {
            res[key] = kvp.Value;
        }
    }
    return res;
}
```

**Purpose**:
Take a nested `Dictionary<string, object>` and return a “flattened” dictionary where nested keys are concatenated using the `separator`.

* If a value is another dictionary, recursively flatten it with the combined key.
* Otherwise, store `key → value` in `res`.

**Example**:

```csharp
var nested = new Dictionary<string, object>
{
    { "player", new Dictionary<string, object> {
        { "name", "Alice" },
        { "stats", new Dictionary<string, object> {
            { "hp", 100 }, { "mp", 50 }
        } }
    } },
    { "level", 3 }
};
var flat = JsonHelper.Flatten(nested);
// flat == {
//   "player.name": "Alice",
//   "player.stats.hp": 100,
//   "player.stats.mp": 50,
//   "level": 3
// }
```

---

### 8.5 Method: `public static bool ContainsPath(Dictionary<string, object> dict, string path)`

```csharp
public static bool ContainsPath(Dictionary<string, object> dict, string path)
{
    try { GetByPath(dict, path); return true; }
    catch { return false; }
}
```

**Purpose**:
Convenience wrapper around `GetByPath`. Returns `true` if the path exists, or `false` if `GetByPath` throws a `KeyNotFoundException`.

**Example**:

```csharp
var dict = JsonHelper.DeserializeToDictionary(@"{ ""settings"": { ""audio"": { ""vol"": 0.8 } } }");
bool hasVol = JsonHelper.ContainsPath(dict, "settings.audio.vol"); // true
bool hasRes = JsonHelper.ContainsPath(dict, "settings.video.res"); // false
```

---

## 9. Unity Editor & PlayerPrefs Helpers

### 9.1 Method: `public static void CreateScriptableObjectFromJson<T>(string json, string assetPath) where T : ScriptableObject`

```csharp
#if UNITY_EDITOR
public static void CreateScriptableObjectFromJson<T>(string json, string assetPath) where T : ScriptableObject
{
    var obj = ScriptableObject.CreateInstance<T>();
    JsonUtility.FromJsonOverwrite(json, obj);
    AssetDatabase.CreateAsset(obj, assetPath);
    AssetDatabase.SaveAssets();
}
#endif
```

**Purpose**:
(Only available in the Editor) Create a new `ScriptableObject` asset of type `T`, populate its fields from a JSON string using Unity’s built‐in `JsonUtility`, and save it at `assetPath` (e.g., `"Assets/Data/MyConfig.asset"`).

**Parameters**:

* `json` (`string`): A JSON string whose keys must match public fields/properties of `T`.
* `assetPath` (`string`): Relative path under `Assets/` where the new `.asset` file is created.

**Example**:

```csharp
#if UNITY_EDITOR
[MenuItem("Assets/Create/Config From JSON")]
public static void CreateConfig()
{
    string json = File.ReadAllText("config.json");
    JsonHelper.CreateScriptableObjectFromJson<MyConfig>("" + json, "Assets/Configs/MyConfig.asset");
}
#endif
```

* After running this menu command, a new `MyConfig.asset` file appears in `Assets/Configs/` with fields populated from `config.json`.

---

### 9.2 Method: `public static void SaveToPrefs(string key, object obj)`

```csharp
public static void SaveToPrefs(string key, object obj)
{
    PlayerPrefs.SetString(key, Serialize(obj));
    PlayerPrefs.Save();
}
```

**Purpose**:
Serialize an object to JSON and store it in `PlayerPrefs` under the given `key`. Immediately calls `PlayerPrefs.Save()` to flush changes.

**Example**:

```csharp
GameSettings gs = new GameSettings { MasterVolume = 0.8f, HighScore = 5000 };
JsonHelper.SaveToPrefs("GameSettings", gs);
// Now PlayerPrefs.GetString("GameSettings") returns JSON of gs.
```

---

### 9.3 Method: `public static Dictionary<string, object> LoadFromPrefs(string key)`

```csharp
public static Dictionary<string, object> LoadFromPrefs(string key)
{
    return PlayerPrefs.HasKey(key)
        ? DeserializeToDictionary(PlayerPrefs.GetString(key))
        : null;
}
```

**Purpose**:
If `PlayerPrefs` contains a value under `key`, read it (JSON string), parse it to a dictionary, and return it. Otherwise, return `null`.

**Example**:

```csharp
var prefsDict = JsonHelper.LoadFromPrefs("GameSettings");
if (prefsDict != null)
{
    float vol = (float)(double)JsonHelper.GetByPath(prefsDict, "MasterVolume"); 
    int highScore = (int)(long)JsonHelper.GetByPath(prefsDict, "HighScore");
}
```

---

## 10. HTTP Requests (UnityWebRequest)

### 10.1 Method: `public static async Task<UnityWebRequestAsyncOperation> SendJsonAsync(string url, object payload, string method = UnityWebRequest.kHttpVerbPOST, Dictionary<string,string> headers = null)`

```csharp
public static async Task<UnityWebRequestAsyncOperation> SendJsonAsync(
    string url,
    object payload,
    string method = UnityWebRequest.kHttpVerbPOST,
    Dictionary<string,string> headers = null)
{
    string json = Serialize(payload);
    byte[] body = Encoding.UTF8.GetBytes(json);
    using var req = new UnityWebRequest(url, method)
    {
        uploadHandler = new UploadHandlerRaw(body),
        downloadHandler = new DownloadHandlerBuffer()
    };
    req.SetRequestHeader("Content-Type", "application/json");
    if (headers != null)
        foreach (var kvp in headers)
            req.SetRequestHeader(kvp.Key, kvp.Value);

    var op = req.SendWebRequest();
    #if UNITY_2020_1_OR_NEWER
    while (!op.isDone) await Task.Yield();
    #else
    await Task.Run(() => { while (!op.isDone) {} });
    #endif
    if (req.result != UnityWebRequest.Result.Success)
        Debug.LogError($"SendJsonAsync Error: {req.error}\n{req.downloadHandler.text}");
    return op;
}
```

**Purpose**:
Issue an HTTP request (`POST`, `PUT`, etc.) sending a JSON‐serialized `payload` to `url`. Returns the `UnityWebRequestAsyncOperation`, which the caller can inspect (e.g., `req.downloadHandler.text`) to see the response body.

**Parameters**:

* `url` (`string`): Endpoint to call (e.g., `"https://api.example.com/data"`).
* `payload` (`object`): Object to serialize as JSON for the request body.
* `method` (`string`): HTTP method (`"POST"` by default, can be `"PUT"`, `"PATCH"`, etc.).
* `headers` (`Dictionary<string,string>`, optional): Additional HTTP headers (e.g., authorization tokens).

**Behavior**:

1. `Serialize(payload)` → JSON string.
2. Convert to UTF‐8 bytes.
3. Create a new `UnityWebRequest(url, method)` with `uploadHandler = new UploadHandlerRaw(body)` and `downloadHandler = new DownloadHandlerBuffer()`.
4. Set `Content-Type: application/json`.
5. Apply extra headers if provided.
6. Start the request via `req.SendWebRequest()`.
7. Crop execution until `op.isDone` by `await Task.Yield()` on Unity 2020.1+, or busy‐wait on earlier versions.
8. If `req.result` indicates an error, log it with `Debug.LogError`, including `req.downloadHandler.text` (response body).
9. Return the operation (`UnityWebRequestAsyncOperation`) so the caller can inspect e.g. `((DownloadHandlerBuffer)op.webRequest.downloadHandler).text`.

**Example**:

```csharp
async Task PostScoreAsync(int score)
{
    string url = "https://api.example.com/highscores";
    var data = new { playerName = "Alice", score = score };
    var headers = new Dictionary<string,string> { { "Authorization", "Bearer abc123" } };
    var op = await JsonHelper.SendJsonAsync(url, data, UnityWebRequest.kHttpVerbPOST, headers);

    string response = op.webRequest.downloadHandler.text;
    Debug.Log($"Server response: {response}");
}
```

---

### 10.2 Method: `public static async Task<T> GetJsonAsync<T>(string url, Dictionary<string,string> headers = null)`

```csharp
public static async Task<T> GetJsonAsync<T>(
    string url,
    Dictionary<string,string> headers = null)
{
    using var req = UnityWebRequest.Get(url);
    req.SetRequestHeader("Content-Type","application/json");
    if (headers != null)
        foreach (var kvp in headers)
            req.SetRequestHeader(kvp.Key, kvp.Value);

    var op = req.SendWebRequest();
    #if UNITY_2020_1_OR_NEWER
    while (!op.isDone) await Task.Yield();
    #else
    await Task.Run(() => { while (!op.isDone) {} });
    #endif

    if (req.result != UnityWebRequest.Result.Success)
    {
        Debug.LogError($"GetJsonAsync Error: {req.error}\n{req.downloadHandler.text}");
        return default;
    }

    return JsonConvert.DeserializeObject<T>(req.downloadHandler.text);
}
```

**Purpose**:
Perform an HTTP GET, expecting a JSON response, then deserialize that JSON into a typed object `T`. Logs any HTTP errors.

**Parameters**:

* `url` (`string`): The endpoint to GET from.
* `headers` (`Dictionary<string,string>`, optional): Additional request headers.

**Example**:

```csharp
async Task FetchPlayerStats()
{
    string url = "https://api.example.com/playerstats/alice";
    PlayerStats stats = await JsonHelper.GetJsonAsync<PlayerStats>(url);
    if (stats != null)
    {
        Debug.Log($"HP: {stats.HP}, MP: {stats.MP}");
    }
}
```

---

## 11. Versioning / Migration Hooks

### 11.1 Method: `public static void RegisterMigration(int version, Func<Dictionary<string, object>, Dictionary<string, object>> migrate)`

```csharp
private static readonly Dictionary<int, Func<Dictionary<string, object>, Dictionary<string, object>>> _migrations =
    new Dictionary<int, Func<Dictionary<string, object>, Dictionary<string, object>>>();

public static void RegisterMigration(int version, Func<Dictionary<string, object>, Dictionary<string, object>> migrate)
    => _migrations[version] = migrate;
```

**Purpose**:
Allow your code to register migration functions that transform a dictionary from one schema version to the next. Store each migration in `_migrations` keyed by the source schema version.

**Parameters**:

* `version` (`int`): The version number of the schema that this migration will handle.
* `migrate` (`Func<Dictionary<string, object>, Dictionary<string, object>>`): A function that takes a dictionary in the old format and returns a new dictionary in the updated format (or modifies in place).

**Example**:

```csharp
// Suppose original schema v1 had "hp" and "mp"; schema v2 renames "hp" -> "health", "mp" -> "mana"

JsonHelper.RegisterMigration(1, oldDict =>
{
    if (oldDict.TryGetValue("hp", out object hpVal) && oldDict.TryGetValue("mp", out object mpVal))
    {
        oldDict.Remove("hp");
        oldDict.Remove("mp");
        oldDict["health"] = hpVal;
        oldDict["mana"] = mpVal;
    }
    return oldDict;
});
```

---

### 11.2 Method: `public static Dictionary<string, object> Migrate(Dictionary<string, object> dict, int fromVersion, int toVersion)`

```csharp
public static Dictionary<string, object> Migrate(Dictionary<string, object> dict, int fromVersion, int toVersion)
{
    for (int v = fromVersion; v < toVersion; v++)
        if (_migrations.TryGetValue(v, out var fn))
            dict = fn(dict);
    return dict;
}
```

**Purpose**:
Given a dictionary that represents JSON data in schema version `fromVersion`, apply all registered migrations in ascending order up to (but not including) `toVersion`.

**Example**:

```csharp
// Current desired version is 3, but data is saved at version 1. 
// You have migrations for version 1 → 2 and version 2 → 3.

Dictionary<string, object> dataV1 = /* load data */;
var dataV3 = JsonHelper.Migrate(dataV1, fromVersion: 1, toVersion: 3);
// dataV3 is now transformed through registered migrations at versions 1 and 2.
```

---

## 12. Custom Converter Registration

### 12.1 Method: `public static void RegisterConverter(JsonConverter converter)`

```csharp
private static readonly List<JsonConverter> _customConverters = new List<JsonConverter>();
public static void RegisterConverter(JsonConverter converter) 
    => _customConverters.Add(converter);
```

**Purpose**:
Allow the user to register any `Newtonsoft.Json.JsonConverter` ahead of time (for example, a custom converter for Unity’s `Vector3` or `Quaternion`). These converters will be included whenever `Serialize(...)` or `Deserialize<T>(...)` is called internally.

**Example**:

```csharp
JsonHelper.RegisterConverter(new Vector3Converter());
// Then later, calls to JsonHelper.Serialize(myObject) will be able to handle Vector3 fields using that converter.
```

---

### 12.2 Helper: `private static JsonSerializerSettings CreateSettings(bool prettyPrint)`

```csharp
private static JsonSerializerSettings CreateSettings(bool prettyPrint)
    => new JsonSerializerSettings
    {
        Formatting = prettyPrint ? Formatting.Indented : Formatting.None,
        Converters = _customConverters.Concat(new JsonConverter[]{ new StringEnumConverter() }).ToList(),
        ReferenceLoopHandling = ReferenceLoopHandling.Ignore,
        NullValueHandling = NullValueHandling.Include
    };
```

**Purpose**:
Build and return `JsonSerializerSettings` used by `Serialize(...)`. Settings include:

* **`Formatting`**: Indented if `prettyPrint` is `true`, otherwise no formatting.
* **`Converters`**: All user‐registered converters (`_customConverters`) plus a default `StringEnumConverter` so that enum values serialize as their string names.
* **`ReferenceLoopHandling.Ignore`**: Prevent infinite loops when serializing object graphs containing circular references.
* **`NullValueHandling.Include`**: Keep properties even if their value is `null` (unless you’ve pruned them yourself with `RemoveNulls`).

---

## 13. Compression Helpers (GZIP)

### 13.1 Method: `public static byte[] CompressJson(string json)`

```csharp
public static byte[] CompressJson(string json)
{
    var bytes = Encoding.UTF8.GetBytes(json);
    using var ms = new MemoryStream();
    using (var gz = new GZipStream(ms, CompressionMode.Compress))
        gz.Write(bytes, 0, bytes.Length);
    return ms.ToArray();
}
```

**Purpose**:
Compress a UTF‐8 JSON string into GZIP‐encoded bytes for more efficient storage or network transmission.

**Behavior**:

1. `Encoding.UTF8.GetBytes(json)` converts to a `byte[]`.
2. Open a `MemoryStream` (in which compressed data will be stored).
3. Wrap it in a `GZipStream` in `CompressionMode.Compress`.
4. Write all bytes to the GZip stream, which compresses them.
5. Return `ms.ToArray()`: the compressed byte array.

**Example**:

```csharp
string json = JsonHelper.Serialize(myLargeObject, prettyPrint: false);
byte[] compressed = JsonHelper.CompressJson(json);
// You can now write `compressed` to a file or send over network in binary form.
```

---

### 13.2 Method: `public static string DecompressJson(byte[] compressed)`

```csharp
public static string DecompressJson(byte[] compressed)
{
    using var ms = new MemoryStream(compressed);
    using var gz = new GZipStream(ms, CompressionMode.Decompress);
    using var sr = new StreamReader(gz);
    return sr.ReadToEnd();
}
```

**Purpose**:
Reverse the above compression: take a GZIP‐compressed JSON byte array and return the original JSON string.

**Example**:

```csharp
byte[] compressed = /* read compressed data from disk or network */;
string json = JsonHelper.DecompressJson(compressed);
var obj = JsonHelper.Deserialize<MyData>(json);
```

---

## 14. Telemetry Hooks

### 14.1 Events: `public static event Action<string> OnOperationStart;` `public static event Action<string, int> OnOperationComplete;`

```csharp
public static event Action<string> OnOperationStart;
public static event Action<string, int> OnOperationComplete;
```

**Purpose**:
Allow external code to subscribe to simple telemetry around JSON operations—e.g., log to console, collect metrics, or time operations.

* **`OnOperationStart(string operationName)`**: Fired before certain methods begin (e.g., `SerializeWithEvents`).
* **`OnOperationComplete(string operationName, int resultSize)`**: Fired after completing, passing the length (in characters) of the produced JSON string.

---

### 14.2 Method: `public static string SerializeWithEvents(object data, bool prettyPrint = false)`

```csharp
public static string SerializeWithEvents(object data, bool prettyPrint = false)
{
    OnOperationStart?.Invoke(nameof(SerializeWithEvents));
    var json = Serialize(data, prettyPrint);
    OnOperationComplete?.Invoke(nameof(SerializeWithEvents), json.Length);
    return json;
}
```

**Purpose**:
Combine serialization with telemetry events:

1. Invoke `OnOperationStart("SerializeWithEvents")`.
2. Call `Serialize(data, prettyPrint)` to get JSON.
3. Invoke `OnOperationComplete("SerializeWithEvents", json.Length)`.
4. Return the JSON string.

**Example**:

```csharp
// Somewhere in initialization:
JsonHelper.OnOperationStart += opName => Debug.Log($"[Telemetry] Starting {opName}");
JsonHelper.OnOperationComplete += (opName, size) => Debug.Log($"[Telemetry] Completed {opName} (size={size} bytes)");

// Usage:
string j = JsonHelper.SerializeWithEvents(myData);
// Console will show two logs, marking start and complete.
```

---

## 15. Real-World Usage Examples

Below are concrete examples illustrating how to leverage **JsonHelper** in typical Unity or .NET scenarios.

---

### 15.1 Serialize a `GameData` Object to JSON String

```csharp
[Serializable]
public class GameData
{
    public string PlayerName;
    public int Level;
    public float HP;
    public List<string> Inventory;
}

void Example1()
{
    var data = new GameData
    {
        PlayerName = "Alice",
        Level = 5,
        HP = 75.5f,
        Inventory = new List<string> { "Sword", "Potion" }
    };

    // Without pretty print (compact):
    string compactJson = JsonHelper.Serialize(data, prettyPrint: false);
    Debug.Log(compactJson);
    // Example output: {"PlayerName":"Alice","Level":5,"HP":75.5,"Inventory":["Sword","Potion"]}

    // With pretty print:
    string prettyJson = JsonHelper.Serialize(data, prettyPrint: true);
    Debug.Log(prettyJson);
    /*
    {
      "PlayerName": "Alice",
      "Level": 5,
      "HP": 75.5,
      "Inventory": [
        "Sword",
        "Potion"
      ]
    }
    */
}
```

---

### 15.2 Read/Write JSON File on Disk (Sync & Async)

```csharp
public class SaveLoadExample : MonoBehaviour
{
    private string dataPath => Path.Combine(Application.persistentDataPath, "gamedata.json");

    private void SaveData()
    {
        var data = new Dictionary<string, object>
        {
            {"PlayerName", "Bob"},
            {"Score", 12345},
            {"Timestamp", DateTime.UtcNow}
        };
        JsonHelper.SerializeToFile(data, dataPath, prettyPrint: true);
        Debug.Log($"Saved data to {dataPath}");
    }

    private async void LoadData()
    {
        try
        {
            Dictionary<string, object> dict = await JsonHelper.DeserializeFromFileAsync(dataPath);
            string name = (string)dict["PlayerName"];
            long score = (long)dict["Score"];
            DateTime ts = (DateTime)dict["Timestamp"];
            Debug.Log($"Loaded: {name}, Score = {score}, Time = {ts}");
        }
        catch (FileNotFoundException)
        {
            Debug.LogWarning($"File not found: {dataPath}");
        }
    }
}
```

* **Sync**: `SerializeToFile` and `DeserializeFromFile` block the main thread.
* **Async**: `SerializeToFileAsync` / `DeserializeFromFileAsync` run on background threads and return `Task`, so you can `await` in an async method.

---

### 15.3 Convert JSON to `Dictionary<string, object>` and Navigate Nested Fields

```csharp
void Example3()
{
    string json = @"
    {
      ""settings"": {
        ""audio"": {
          ""volume"": 0.7,
          ""mute"": false
        },
        ""video"": {
          ""resolution"": ""1920x1080"",
          ""fullscreen"": true
        }
      },
      ""player"": {
        ""name"": ""Charlie"",
        ""level"": 10
      }
    }";

    var dict = JsonHelper.DeserializeToDictionary(json);

    // Access nested values:
    double volume = (double)JsonHelper.GetByPath(dict, "settings.audio.volume"); // 0.7
    bool fullscreen = (bool)JsonHelper.GetByPath(dict, "settings.video.fullscreen"); // true
    string playerName = (string)JsonHelper.GetByPath(dict, "player.name"); // "Charlie"

    Debug.Log($"Volume={volume}, Fullscreen={fullscreen}, Player={playerName}");
}
```

---

### 15.4 Merge Two JSON Objects and Save Result

```csharp
void Example4()
{
    JObject baseConfig = JObject.Parse(@"{ ""theme"": ""light"", ""features"": { ""chat"": true } }");
    JObject userConfig = JObject.Parse(@"{ ""theme"": ""dark"", ""features"": { ""ads"": false } }");

    JsonHelper.MergeJson(baseConfig, userConfig);
    
    // baseConfig now:
    // {
    //   "theme": "dark",
    //   "features": {
    //     "chat": true,
    //     "ads": false
    //   }
    // }
    string mergedJson = baseConfig.ToString(Formatting.Indented);
    Debug.Log(mergedJson);
}
```

* Notice how `features.chat` remains, and `features.ads` is added. The `theme` property is overwritten (non‐null in `userConfig`). If `userConfig.theme` were `null`, it would be ignored.

---

### 15.5 Compare Differences Between Two JSON Payloads

```csharp
void Example5()
{
    JToken v1 = JToken.Parse(@"
    {
      ""name"": ""Alice"",
      ""stats"": { ""hp"": 100, ""mp"": 50 },
      ""inventory"": [""Sword"", ""Shield""]
    }");
    JToken v2 = JToken.Parse(@"
    {
      ""name"": ""Alice"",
      ""stats"": { ""hp"": 90, ""mp"": 50 },
      ""inventory"": [""Sword"", ""Potion""],
      ""level"": 2
    }");

    List<string> diffs = JsonHelper.DiffJson(v1, v2);
    // diffs might contain: [ "/stats/hp", "/inventory", "/level" ]
    foreach (var path in diffs)
        Debug.Log($"Difference at: {path}");
}
```

---

### 15.6 Get or Set a Deeply Nested Value by Path

```csharp
void Example6()
{
    var dict = new Dictionary<string, object>();
    // Set a nested value (creates intermediate dictionaries)
    JsonHelper.SetByPath(dict, "game.settings.audio.vol", 0.9f);
    JsonHelper.SetByPath(dict, "game.settings.video.fullscreen", true);

    // Now get them:
    float vol = (float)(double)JsonHelper.GetByPath(dict, "game.settings.audio.vol");
    bool fs = (bool)JsonHelper.GetByPath(dict, "game.settings.video.fullscreen");

    Debug.Log($"Volume={vol}, Fullscreen={fs}");
}
```

---

### 15.7 Remove All Nulls and Flatten a Nested Dictionary

```csharp
void Example7()
{
    string rawJson = @"
    {
      ""a"": null,
      ""b"": { ""x"": null, ""y"": 5, ""z"": { ""w"": null } },
      ""c"": [null, 10, { ""d"": null }, 20]
    }";

    var dict = JsonHelper.DeserializeToDictionary(rawJson);
    JsonHelper.RemoveNulls(dict);
    // dict now == { "b": { "y": 5 }, "c": [10, 20] }

    var flat = JsonHelper.Flatten(dict);
    // flat == { "b.y": 5, "c.0": 10, "c.1": 20 }
    
    foreach (var kv in flat)
        Debug.Log($"{kv.Key} = {kv.Value}");
}
```

---

### 15.8 Deep-Clone an Object via JSON Roundtrip

```csharp
[Serializable]
public class Settings
{
    public float MasterVolume;
    public List<string> UnlockedLevels;
}

void Example8()
{
    var orig = new Settings { MasterVolume = 0.8f, UnlockedLevels = new List<string> { "Level1", "Level2" } };
    var clone = JsonHelper.DeepClone(orig);
    clone.UnlockedLevels.Add("Level3");

    Debug.Log($"Orig: {string.Join(",", orig.UnlockedLevels)}"); // Level1,Level2
    Debug.Log($"Clone: {string.Join(",", clone.UnlockedLevels)}"); // Level1,Level2,Level3
}
```

---

### 15.9 Save Custom Data to PlayerPrefs and Load It Back

```csharp
void Example9_Save()
{
    var profile = new Dictionary<string, object>
    {
        { "username", "Zoe" },
        { "highScore", 4500 },
        { "lastLogin", DateTime.Now }
    };
    JsonHelper.SaveToPrefs("userProfile", profile);
}

void Example9_Load()
{
    var loaded = JsonHelper.LoadFromPrefs("userProfile");
    if (loaded != null)
    {
        string name = (string)loaded["username"];
        long highScore = (long)loaded["highScore"];
        DateTime dt = (DateTime)loaded["lastLogin"];
        Debug.Log($"Loaded profile: {name}, highScore={highScore}, lastLogin={dt}");
    }
}
```

---

### 15.10 Perform a POST of JSON Payload to a REST API and Read the Response

```csharp
async void Example10()
{
    string url = "https://example.com/api/update";
    var payload = new { id = 123, status = "active" };
    var headers = new Dictionary<string, string> { { "Authorization", "Bearer token123" } };

    var op = await JsonHelper.SendJsonAsync(url, payload, UnityWebRequest.kHttpVerbPOST, headers);
    string responseJson = op.webRequest.downloadHandler.text;

    // Suppose the response JSON is { "success": true, "message": "Updated" }
    var responseDict = JsonHelper.DeserializeToDictionary(responseJson);
    bool success = (bool)responseDict["success"];
    string message = (string)responseDict["message"];
    Debug.Log($"API responded: success={success}, message={message}");
}
```

---

### 15.11 Compress a JSON String Before Saving (and Decompress After Loading)

```csharp
void Example11_CompressSave()
{
    var data = new Dictionary<string, object> { { "largeList", Enumerable.Range(0, 1000).ToList() } };
    string json = JsonHelper.Serialize(data);
    byte[] compressed = JsonHelper.CompressJson(json);
    File.WriteAllBytes("data.gz", compressed);
    Debug.Log($"Compressed size: {compressed.Length} bytes");
}

void Example11_DecompressLoad()
{
    byte[] compressed = File.ReadAllBytes("data.gz");
    string json = JsonHelper.DecompressJson(compressed);
    var dict = JsonHelper.DeserializeToDictionary(json);
    Debug.Log($"Loaded and decompressed: count = {((List<object>)dict["largeList"]).Count}");
}
```

---

### 15.12 Apply Version-Based Migrations to a Dictionary

```csharp
void Example12_Migrations()
{
    // Register migrations at startup (once)
    JsonHelper.RegisterMigration(1, dict =>
    {
        // v1 → v2: rename “oldField” → “newField”
        if (dict.TryGetValue("oldField", out var val))
        {
            dict.Remove("oldField");
            dict["newField"] = val;
        }
        return dict;
    });

    JsonHelper.RegisterMigration(2, dict =>
    {
        // v2 → v3: change “score” from int to long and multiply by 100
        if (dict.TryGetValue("score", out var sVal) && sVal is long s)
        {
            dict["score"] = s * 100L;
        }
        return dict;
    });

    // Suppose we loaded JSON for a v1 dictionary:
    var dictV1 = JsonHelper.DeserializeToDictionary(@"{ ""oldField"": ""abc"", ""score"": 10 }");

    // Now migrate from v1 to v3:
    var dictV3 = JsonHelper.Migrate(dictV1, fromVersion: 1, toVersion: 3);
    // Now dictV3 should have “newField” = "abc", and “score” = 1000 (int→long *100)
    Debug.Log(JsonHelper.Serialize(dictV3, prettyPrint: true));
}
```

---

### 15.13 Register a Custom Converter (e.g., for Unity’s `Vector3`)

```csharp
// Suppose you have a Vector3Converter that inherits JsonConverter:
public class Vector3Converter : JsonConverter<Vector3>
{
    public override void WriteJson(JsonWriter writer, Vector3 value, JsonSerializer serializer)
    {
        writer.WriteStartObject();
        writer.WritePropertyName("x"); writer.WriteValue(value.x);
        writer.WritePropertyName("y"); writer.WriteValue(value.y);
        writer.WritePropertyName("z"); writer.WriteValue(value.z);
        writer.WriteEndObject();
    }

    public override Vector3 ReadJson(JsonReader reader, Type objectType, Vector3 existingValue, bool hasExistingValue, JsonSerializer serializer)
    {
        var jo = JObject.Load(reader);
        float x = jo["x"].Value<float>();
        float y = jo["y"].Value<float>();
        float z = jo["z"].Value<float>();
        return new Vector3(x, y, z);
    }
}

// Register it early in your game initialization:
void Awake()
{
    JsonHelper.RegisterConverter(new Vector3Converter());
}

// Now serialize an object containing Vector3:
public class TransformData
{
    public Vector3 Position;
    public Vector3 Rotation;
}

void Example13()
{
    var td = new TransformData { Position = new Vector3(1,2,3), Rotation = new Vector3(0,90,0) };
    string json = JsonHelper.Serialize(td, prettyPrint: true);
    Debug.Log(json);
    /*
    {
      "Position": { "x": 1.0, "y": 2.0, "z": 3.0 },
      "Rotation": { "x": 0.0, "y": 90.0, "z": 0.0 }
    }
    */
}
```

---

### 15.14 Wrap Serialization in Telemetry Events

```csharp
void Example14()
{
    JsonHelper.OnOperationStart += opName => Debug.Log($"Telemetry: starting {opName}");
    JsonHelper.OnOperationComplete += (opName, size) => Debug.Log($"Telemetry: completed {opName}, len={size}");

    var data = new { timestamp = DateTime.Now, values = new int[] {1,2,3} };
    string json = JsonHelper.SerializeWithEvents(data, prettyPrint: false);
    // Output:
    // [Telemetry] starting SerializeWithEvents
    // [Telemetry] completed SerializeWithEvents, len=...
}
```

---

## 16. Best Practices & Tips

1. **When to Use `DeserializeToDictionary` vs. `Deserialize<T>`**

   * Use `Deserialize<T>(json)` if you know the target C# type.
   * Use `DeserializeToDictionary` when you want a flexible, schema‐less structure at runtime.

2. **“Integer” Values in Dictionaries Are `long`**

   * All JSON integers become `long` in the dictionary. If you need an `int`, cast or convert explicitly: `(int)(long)dict["myIntValue"]`.

3. **Take Care with `null` vs. Missing Keys**

   * After `DeserializeToDictionary`, a JSON `null` value results in a dictionary entry `key → null`.
   * A missing key (not present in JSON) means `ContainsKey(...) == false`.

4. **Using `RemoveNulls` Before Flattening or Exporting**

   * If you want to strip out unwanted `null` branches, call `RemoveNulls(rootDict)` before writing or flattening.

5. **Applying Migrations**

   * Register migrations on startup (once). When loading older data, call `Migrate(dict, fromVersion, toVersion)` to bring data up to date.

6. **Watch Out for Large JSON with `PrettyPrint`**

   * `PrettyPrint` parses the entire JSON string into a `JToken`. For multi-MB JSON, this may be slow—use sparingly or only in the Editor.

7. **HTTP Requests on Unity WebGL**

   * In WebGL builds, you cannot await `req.SendWebRequest()` in a blocking loop. Instead, use a Unity coroutine or adjust for WebGL restrictions.

8. **Custom Converters for Unity Types**

   * Register converters for types like `Vector3`, `Color`, `Quaternion` so that `Serialize` and `Deserialize` handle them out-of-the-box.

9. **GZIP Compression When Transferring Large JSON**

   * Compress before sending over network or writing to disk if size matters. Decompress as close to usage time as possible.

10. **Use `SerializeWithEvents` for Simple Telemetry**

    * If you need to log or monitor how often objects serialize and how large they are, subscribe to `OnOperationStart` / `OnOperationComplete`.

---

## 17. Summary of Public API

Below is a condensed list of every public method and event in **JsonHelper**, organized by category:

### Core Serialize / Deserialize

| Method                                                                                             | Description                                                                      |
| -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `static string Serialize(object data, bool prettyPrint = false)`                                   | Convert any object to JSON (optionally pretty‐printed).                          |
| `static Dictionary<string, object> DeserializeToDictionary(string json)`                           | Parse JSON (root must be object) into nested `Dictionary<string, object>`.       |
| `static bool TryDeserialize(string json, out Dictionary<string, object> result, out string error)` | Attempt `DeserializeToDictionary`, returning `false` + error message on failure. |

### File I/O

| Method                                                                                                                     | Description                                                |
| -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `static void SerializeToFile(object data, string filePath, bool prettyPrint = false)`                                      | Synchronous: write JSON of `data` to disk at `filePath`.   |
| `static Dictionary<string, object> DeserializeFromFile(string filePath)`                                                   | Synchronous: read JSON file and parse into dictionary.     |
| `static Task SerializeToFileAsync(object data, string filePath, bool prettyPrint = false, CancellationToken ct = default)` | Asynchronous version of writing JSON to disk.              |
| `static Task<Dictionary<string, object>> DeserializeFromFileAsync(string filePath, CancellationToken ct = default)`        | Asynchronous version of reading JSON file into dictionary. |

### Formatting & Minification

| Method                                   | Description                                                |
| ---------------------------------------- | ---------------------------------------------------------- |
| `static string PrettyPrint(string json)` | Reformat `json` with indentation for readability.          |
| `static string Minify(string json)`      | Remove all whitespace from `json`, minifying it.           |
| `static T Deserialize<T>(string json)`   | Generic wrapper: `JsonConvert.DeserializeObject<T>(json)`. |

### Merge & Diff

| Method                                             | Description                                                       |
| -------------------------------------------------- | ----------------------------------------------------------------- |
| `static void MergeJson(JObject dest, JObject src)` | Merge `src` into `dest`, concatenating arrays and ignoring nulls. |
| `static List<string> DiffJson(JToken a, JToken b)` | Compare two JSON tokens; return list of paths where they differ.  |

### Path‐Based Access

| Method                                                                              | Description                                                             |
| ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `static object GetByPath(Dictionary<string, object> dict, string path)`             | Retrieve a nested value by path segments (dot or slash separated).      |
| `static void SetByPath(Dictionary<string, object> dict, string path, object value)` | Set a nested value by path, creating missing intermediate dictionaries. |

### Cleanup & Conversion

| Method                                                                                                                      | Description                                                                               |
| --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `static void RemoveNulls(object obj)`                                                                                       | Recursively remove `null` entries and empty containers from nested dictionaries or lists. |
| `static T ToObject<T>(Dictionary<string, object> dict)`                                                                     | Convert a dictionary back into a typed object `T` via JSON roundtrip.                     |
| `static T DeepClone<T>(T obj)`                                                                                              | Deep‐copy any serializable object by serializing and deserializing through JSON.          |
| `static Dictionary<string, object> Flatten(Dictionary<string, object> source, string parentKey = "", char separator = '.')` | Flatten a nested dictionary so that nested keys become `"parent.child.grandchild"`.       |
| `static bool ContainsPath(Dictionary<string, object> dict, string path)`                                                    | Return `true` if `GetByPath(dict, path)` would succeed; otherwise, `false`.               |

### Unity Editor & PlayerPrefs

| Method                                                                                                                  | Description                                                                                       |
| ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `static void CreateScriptableObjectFromJson<T>(string json, string assetPath) where T : ScriptableObject` (Editor only) | Instantiate a new ScriptableObject `T` from JSON and save it as an asset at `assetPath`.          |
| `static void SaveToPrefs(string key, object obj)`                                                                       | Serialize `obj` to JSON and store in `PlayerPrefs` under `key`.                                   |
| `static Dictionary<string, object> LoadFromPrefs(string key)`                                                           | Load JSON from `PlayerPrefs` under `key` and parse into dictionary; return `null` if key missing. |

### HTTP Requests (UnityWebRequest)

| Method                                                                                                                                                 | Description                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| `static Task<UnityWebRequestAsyncOperation> SendJsonAsync(string url, object payload, string method = POST, Dictionary<string,string> headers = null)` | Send a JSON payload via POST/PUT to `url`. Returns the operation so caller can inspect response. |
| `static Task<T> GetJsonAsync<T>(string url, Dictionary<string,string> headers = null)`                                                                 | Perform HTTP GET, deserialize JSON response into type `T`. Returns `default(T)` on HTTP error.   |

### Versioning / Migration Hooks

| Method                                                                                                             | Description                                                                        |
| ------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| `static void RegisterMigration(int version, Func<Dictionary<string, object>, Dictionary<string, object>> migrate)` | Register a function to migrate a dictionary from version `version` to `version+1`. |
| `static Dictionary<string, object> Migrate(Dictionary<string, object> dict, int fromVersion, int toVersion)`       | Apply registered migrations in ascending order from `fromVersion` to `toVersion`.  |

### Custom Converter Registration

| Method                                                                      | Description                                                                                         |
| --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `static void RegisterConverter(JsonConverter converter)`                    | Add a `Newtonsoft.Json.JsonConverter` to the list for subsequent (de)serialization.                 |
| `static JsonSerializerSettings CreateSettings(bool prettyPrint)` (internal) | Build `JsonSerializerSettings` including registered converters and a default `StringEnumConverter`. |

### Compression Helpers (GZIP)

| Method                                            | Description                                    |
| ------------------------------------------------- | ---------------------------------------------- |
| `static byte[] CompressJson(string json)`         | GZIP‐compress a JSON string to a `byte[]`.     |
| `static string DecompressJson(byte[] compressed)` | Decompress GZIP bytes back into a JSON string. |

### Telemetry Hooks

| Member                                                                   | Description                                                                             |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------- |
| `static event Action<string> OnOperationStart`                           | Fired (if subscribed) at the start of certain operations (e.g., `SerializeWithEvents`). |
| `static event Action<string, int> OnOperationComplete`                   | Fired when an operation ends, with the produced JSON length (or similar metric).        |
| `static string SerializeWithEvents(object data, bool prettyPrint=false)` | Convenience wrapper that invokes telemetry events around `Serialize()`.                 |

---

With these methods and examples, **JsonHelper** covers nearly any JSON‐centric need in Unity or .NET. Whether you’re persisting game state, syncing with a server, or manipulating JSON in Editor scripts, you’ll find a convenient, type‐safe, and often asynchronous approach here—without pulling in multiple disparate code snippets.
