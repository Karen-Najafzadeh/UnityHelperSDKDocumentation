Below is a complete reference for **LocalizationManager**—a static utility that loads JSON‐based translation tables at runtime, retrieves translated strings (with support for formatting, pluralization, date/number/currency formatting, list formatting, relative time, and more), and lets you switch languages on the fly. It assumes you place files named `<lang>.json` under `Resources/Localization/` (for example, `Resources/Localization/en.json`, `Resources/Localization/fr.json`, etc.).

---

## Table of Contents

1. [Overview & Purpose](#overview--purpose)
2. [Directory & File Conventions](#directory--file-conventions)
3. [Initialization & Language Switching](#initialization--language-switching)

   * 3.1. `Initialize(string defaultLanguage = "en")`
   * 3.2. `ChangeLanguage(string langCode, bool useFallback = true)`
   * 3.3. `ReloadCurrentLanguage()`
   * 3.4. `GetAvailableLanguages()` (Editor vs. Build)
4. [Retrieving Translations](#retrieving-translations)

   * 4.1. `Get(string key, params object[] args)`
   * 4.2. `GetPlural(string key, int count, params object[] extraArgs)`
   * 4.3. Fallback Variants: `TryGet`, `GetOrDefault`, `GetOrNull`, `GetOrThrow`
   * 4.4. Case Variants: `GetTitleCase`, `GetUpper`, `GetLower`
   * 4.5. Whitespace & Rich‐Text Variants: `GetPlain`, `GetCollapsedWhitespace`
   * 4.6. `HasKey(string key)` & `GetAllKeys()` & `GetAllWithPrefix(string prefix)`
   * 4.7. `GetRandom(string baseKey, int count, params object[] args)`
   * 4.8. `GetLocalizedText(string key)` (loads fallback automatically)
5. [Formatting Helpers (Culture‐Aware)](#formatting-helpers-culture-aware)

   * 5.1. `FormatDate(DateTime date, string format = null)`
   * 5.2. `FormatNumber(decimal number, string format = null)`
   * 5.3. `FormatCurrency(decimal amount, string currencyCode = null)`
   * 5.4. `GetRelativeTime(DateTime dateTime)`
   * 5.5. `FormatList(IEnumerable<string> items)`
6. [Currency Parsing Helpers](#currency-parsing-helpers)

   * 6.1. `TryParsePrice(string priceString, out string currencySymbol, out decimal amount)`
   * 6.2. `ExtractCurrencySymbol(string priceString)`
   * 6.3. `ExtractPriceAmount(string priceString)`
7. [Language Metadata & Utilities](#language-metadata--utilities)

   * 7.1. `GetLanguageDisplayName(string langCode)`
   * 7.2. `GetLanguageNativeName(string langCode)`
   * 7.3. `GetAvailableLanguageDisplayNames()` / `GetAvailableLanguageNativeNames()`
   * 7.4. `GetSystemLanguageCode()`
8. [JSON Table Structure](#json-table-structure)

   * 8.1. Example JSON: `{ "entries": [ { "key":"HELLO", "value":"Hello" }, … ] }`
   * 8.2. Internal `LocalizationTable` → `Dictionary<string,string>`
9. [Real‐World Usage Examples](#real-world-usage-examples)

   * 9.1. Basic Setup & Switching Languages
   * 9.2. Retrieving a Simple Translation
   * 9.3. Formatting with Arguments (`Get`)
   * 9.4. Pluralization (`GetPlural`)
   * 9.5. Date, Number, and Currency Formatting
   * 9.6. Relative Time (`GetRelativeTime`)
   * 9.7. List Formatting (`FormatList`)
   * 9.8. Randomized Variations (`GetRandom`)
   * 9.9. Case Transformations (`GetTitleCase`, `GetUpper`, `GetLower`)
   * 9.10. Currency Parsing (`TryParsePrice`, `ExtractCurrencySymbol`, `ExtractPriceAmount`)
   * 9.11. Enumerating Keys & Prefixes (`GetAllKeys`, `GetAllWithPrefix`)
   * 9.12. Fallback Variants (`GetOrDefault`, `GetOrNull`, `GetOrThrow`)
10. [Best Practices & Tips](#best-practices--tips)
11. [Summary of Public API](#summary-of-public-api)

---

## 1. Overview & Purpose

```csharp
public static class LocalizationManager
{
    // … (fields, methods, inner types) …
}
```

**LocalizationManager** provides:

* **Runtime loading** of a JSON file for the chosen language under `Resources/Localization/<lang>.json`.
* **Translation lookup** by key (with optional `{0}`, `{1}`… placeholders).
* **Plural‐aware** lookup (expects two keys: `"<base>.singular"` and `"<base>.plural"`).
* **Culture‐aware formatting** of dates, numbers, and currencies.
* **Relative‐time** generation (e.g. “2 hours ago,” “in 5 days”) based on translation keys.
* **List formatting** (e.g. “apple, banana, and orange” in English vs. “pomme, banane et orange” in French).
* **Randomized variations** (e.g. multiple “error” messages: `error.1`, `error.2`, …).
* **Case transformations** (`TitleCase`, `Upper`, `Lower`) according to the current culture.
* **Fallback** to a default language (e.g. English) if the requested language or key is missing.
* **Currency‐parsing** helpers to split a string like `"$12.34"` into symbol `"$"` and numeric `12.34m`.
* A small **event** (`OnLanguageChanged`) fired whenever the active language changes.

---

## 2. Directory & File Conventions

1. **Folder**: Create a folder `Assets/Resources/Localization/`.

2. **Files**: Place JSON files named exactly like the culture code, e.g.:

   * `Assets/Resources/Localization/en.json`
   * `Assets/Resources/Localization/fr.json`
   * `Assets/Resources/Localization/es-US.json`

3. **JSON Structure**: Each JSON file should conform to:

   ```jsonc
   {
     "entries": [
       { "key": "HELLO", "value": "Hello" },
       { "key": "GOODBYE", "value": "Goodbye" },
       { "key": "items.singular", "value": "You have {0} item" },
       { "key": "items.plural",   "value": "You have {0} items" },
       { "key": "time.seconds_ago", "value": "{0} seconds ago" },
       { "key": "time.in_seconds",  "value": "in {0} seconds" },
       { "key": "list.separator",   "value": ", " },
       { "key": "list.two_items",   "value": "{0} and {1}" },
       { "key": "list.format",      "value": "{0} and {1}" }
       // … etc.
     ]
   }
   ```

   Internally, `LocalizationTable.ToDictionary()` flattens `"entries"` into a `Dictionary<string, string>` for fast lookup.

4. **Naming Keys**:

   * Use a consistent naming scheme (e.g. all‐caps with dots: `UI.PLAY_BUTTON`, `ERROR.INVALID_EMAIL`).
   * For pluralization, add `.singular` and `.plural` suffixes.
   * For lists, include keys for separators and templates (`list.separator`, `list.two_items`, `list.format`).
   * For relative time, include keys like `time.seconds_ago`, `time.in_seconds`, `time.minutes_ago`, etc.

---

## 3. Initialization & Language Switching

### 3.1 `Initialize(string defaultLanguage = "en")`

```csharp
public static void Initialize(string defaultLanguage = "en")
{
    _fallbackLanguage = defaultLanguage;
    ChangeLanguage(Application.systemLanguage.ToString(), useFallback: true);
}
```

* **Purpose**

  * Sets `_fallbackLanguage` (e.g. `"en"`) for key lookups if the requested language fails to load or a key is missing.
  * Attempts to load the system’s language (via `Application.systemLanguage.ToString()`, e.g. `"English"` or `"French"`—Unity’s `SystemLanguage` enum values). If that fails (or file is missing), falls back to the default (`"en"`).

* **Usage Example**

  ```csharp
  public class GameInitializer : MonoBehaviour
  {
      void Start()
      {
          LocalizationManager.Initialize("en");
          // Now LocalizationManager.CurrentLanguage holds the actual code (e.g. "en", "fr", etc.)
      }
  }
  ```

  After calling `Initialize()`, the JSON for the chosen language (or fallback) is loaded into `_translations`, and `OnLanguageChanged` is invoked.

---

### 3.2 `ChangeLanguage(string langCode, bool useFallback = true)`

```csharp
public static bool ChangeLanguage(string langCode, bool useFallback = true)
{
    if (langCode == _currentLanguage)
        return true;

    // Try loading e.g. Resources/Localization/fr.json if langCode == "fr"
    var asset = Resources.Load<TextAsset>($"Localization/{langCode}");
    if (asset == null && useFallback)
        asset = Resources.Load<TextAsset>($"Localization/{_fallbackLanguage}");

    if (asset == null)
    {
        Debug.LogError($"[Localization] Could not load language '{langCode}' or fallback '{_fallbackLanguage}'.");
        return false;
    }

    try
    {
        _translations = JsonUtility.FromJson<LocalizationTable>(asset.text).ToDictionary();
        _currentLanguage = langCode;
        OnLanguageChanged?.Invoke(_currentLanguage);
        Debug.Log($"[Localization] Loaded language: {_currentLanguage}");
        return true;
    }
    catch (Exception ex)
    {
        Debug.LogError($"[Localization] Failed to parse '{langCode}' JSON: {ex}");
        return false;
    }
}
```

* **Purpose**

  1. If `langCode` matches the already‐loaded `_currentLanguage`, return `true` immediately.
  2. Attempt to load `Resources/Localization/{langCode}.json`.
  3. If that JSON is missing and `useFallback == true`, attempt to load `Resources/Localization/{_fallbackLanguage}.json` instead.
  4. If still missing, log an error and return `false`.
  5. Otherwise, parse the JSON via `JsonUtility.FromJson<LocalizationTable>(…)`, convert to a `Dictionary<string,string>`, store it in `_translations`, set `_currentLanguage`, invoke the event `OnLanguageChanged`, and return `true`.

* **Usage Example**

  ```csharp
  public class LanguageSelector : MonoBehaviour
  {
      public void OnFrenchButtonPressed()
      {
          bool success = LocalizationManager.ChangeLanguage("fr", useFallback: true);
          if (!success)
              Debug.LogWarning("French translations not found—still using fallback.");
      }

      public void OnSpanishButtonPressed()
      {
          bool success = LocalizationManager.ChangeLanguage("es", useFallback: true);
          if (!success)
              Debug.LogWarning("Spanish translations not found—still using fallback.");
      }
  }
  ```

  After `ChangeLanguage("fr")`, `_translations` holds all `"fr"` entries, and any UI text that subscribes to `OnLanguageChanged` can refresh itself.

---

### 3.3 `ReloadCurrentLanguage()`

```csharp
public static void ReloadCurrentLanguage()
{
    ChangeLanguage(_currentLanguage, useFallback: true);
}
```

* **Purpose**

  * Re‐loads the same `_currentLanguage` JSON from disk.
  * Useful in development if you tweak the JSON files and want to hot‐reload without restarting the game.

* **Usage Example**

  ```csharp
  // In Editor only—hot‐reload when “R” is pressed:
  void Update()
  {
  #if UNITY_EDITOR
      if (Input.GetKeyDown(KeyCode.R))
      {
          LocalizationManager.ReloadCurrentLanguage();
          Debug.Log("Localization JSON reloaded.");
      }
  #endif
  }
  ```

---

### 3.4 `GetAvailableLanguages()`

```csharp
public static List<string> GetAvailableLanguages()
{
#if UNITY_EDITOR
    var list = new List<string>();
    var folder = Path.Combine(Application.dataPath, "Resources/Localization");
    if (Directory.Exists(folder))
    {
        foreach (var file in Directory.GetFiles(folder, "*.json"))
            list.Add(Path.GetFileNameWithoutExtension(file));
    }
    return list;
#else
    // At runtime Resources cannot enumerate files—just return current & fallback
    return new List<string> { _currentLanguage, _fallbackLanguage };
#endif
}
```

* **Purpose**

  * In the **Editor**, scans `Assets/Resources/Localization/` for all `.json` files and returns their filenames without extension (e.g. `["en", "fr", "es-US"]`).
  * In a **Build**, Unity’s `Resources` cannot enumerate files at runtime, so it returns only `{ _currentLanguage, _fallbackLanguage }` to indicate which languages are relevant.

* **Usage Example**

  ```csharp
  public class LanguageDropdown : MonoBehaviour
  {
      public UnityEngine.UI.Dropdown dropdown;

      void Start()
      {
          var codes = LocalizationManager.GetAvailableLanguages();
          dropdown.ClearOptions();
          dropdown.AddOptions(codes);
      }
  }
  ```

  * In the Editor, this populates the dropdown with all JSON filenames in that folder.
  * In a runtime build, it shows only the two (current & fallback) entries, so you may prefer to maintain a separate manifest of supported languages for a build.

---

## 4. Retrieving Translations

Once `_translations` is populated (after `ChangeLanguage`), you can look up translated strings by key. If a key is missing, by default you’ll see `[KEY]` (e.g. `[HELLO]`), making it obvious that the translation is absent.

### 4.1 `Get(string key, params object[] args)`

```csharp
public static string Get(string key, params object[] args)
{
    if (!_translations.TryGetValue(key, out var format))
        format = $"[{key}]";

    return args != null && args.Length > 0
        ? string.Format(format, args)
        : format;
}
```

* **Purpose**

  1. Attempt to look up `key` in the current `_translations` dictionary.
  2. If found, `format = translationValue`. If not found, return `"[KEY]"`.
  3. If `args` is non‐empty, call `string.Format(format, args)` so you can embed placeholders in your JSON (e.g. `"Welcome, {0}!"`).
  4. Otherwise return the raw `format`.

* **Usage Example**

  ```csharp
  // JSON entry in en.json: { "key":"GREETING", "value":"Welcome, {0}!" }
  string welcomeEn = LocalizationManager.Get("GREETING", "Alice");
  // => "Welcome, Alice!" if currentLanguage is "en"

  // If currentLanguage is "fr" and fr.json has:
  // { "key":"GREETING", "value":"Bienvenue, {0}!" }
  string welcomeFr = LocalizationManager.Get("GREETING", "Alice");
  // => "Bienvenue, Alice!"
  ```

  If `"GREETING"` is missing from the loaded dictionary, you’ll get `"[GREETING]"`.

---

### 4.2 `GetPlural(string key, int count, params object[] extraArgs)`

```csharp
public static string GetPlural(string key, int count, params object[] extraArgs)
{
    string entryKey = count == 1 ? $"{key}.singular" : $"{key}.plural";
    var args = new object[(extraArgs?.Length ?? 0) + 1];
    args[0] = count;
    if (extraArgs != null)
        Array.Copy(extraArgs, 0, args, 1, extraArgs.Length);
    return Get(entryKey, args);
}
```

* **Purpose**

  1. Given a base key (e.g. `"ITEMS"`), determine whether to use `"{key}.singular"` (if `count == 1`) or `"{key}.plural"` (if `count != 1`).
  2. Build an argument array where `args[0] = count`, and any `extraArgs` follow.
  3. Call `Get(entryKey, args)`, so your JSON can embed `{0}` as the count and `{1}`, `{2}`… for extra context.

* **JSON Conventions**
  In your `en.json`, include:

  ```jsonc
  { "key": "items.singular", "value": "You have {0} item" }
  { "key": "items.plural",   "value": "You have {0} items" }
  ```

  In `fr.json`:

  ```jsonc
  { "key": "items.singular", "value": "Vous avez {0} article" }
  { "key": "items.plural",   "value": "Vous avez {0} articles" }
  ```

* **Usage Example**

  ```csharp
  int count = 3;
  string message = LocalizationManager.GetPlural("items", count);
  // => "You have 3 items" (if en), "Vous avez 3 articles" (if fr)

  int single = 1;
  string message2 = LocalizationManager.GetPlural("items", single);
  // => "You have 1 item" or "Vous avez 1 article"
  ```

  If either `"items.singular"` or `"items.plural"` is missing, `Get()` will return `"[items.singular]"` or `"[items.plural]"` accordingly.

---

### 4.3 Fallback Variants

#### `TryGet(string key, out string value)`

```csharp
public static bool TryGet(string key, out string value)
{
    return _translations.TryGetValue(key, out value);
}
```

* **Purpose**:
  Check if a translation key exists. Returns `true` and the translation in `value` if found; otherwise returns `false` and `value = null`.

* **Usage Example**

  ```csharp
  if (LocalizationManager.TryGet("MENU.START", out string startText))
  {
      playButtonText.text = startText;
  }
  else
  {
      playButtonText.text = "Start"; // or some fallback
  }
  ```

#### `GetOrDefault(string key, string fallback, params object[] args)`

```csharp
public static string GetOrDefault(string key, string fallback, params object[] args)
{
    if (!_translations.TryGetValue(key, out var format))
        format = fallback;
    return args != null && args.Length > 0 ? string.Format(format, args) : format;
}
```

* **Purpose**:
  Return `fallback` if the key is missing—otherwise behave exactly like `Get()`.

* **Usage Example**

  ```csharp
  string label = LocalizationManager.GetOrDefault("BUTTON.EXIT", "Exit");
  // If translation exists, use it; otherwise just show "Exit".
  ```

#### `GetOrNull(string key, params object[] args)`

```csharp
public static string GetOrNull(string key, params object[] args)
{
    if (!_translations.TryGetValue(key, out var format))
        return null;
    return args != null && args.Length > 0 ? string.Format(format, args) : format;
}
```

* **Purpose**:
  Return `null` if the key is missing—otherwise behave like `Get()`.

* **Usage Example**

  ```csharp
  string info = LocalizationManager.GetOrNull("DIALOGUE.OPTION2");
  if (info != null)
      ShowOption2Button(info);
  else
      HideOption2Button();
  ```

#### `GetOrThrow(string key, params object[] args)`

```csharp
public static string GetOrThrow(string key, params object[] args)
{
    if (!_translations.TryGetValue(key, out var format))
        throw new KeyNotFoundException($"Translation key not found: {key}");
    return args != null && args.Length > 0 ? string.Format(format, args) : format;
}
```

* **Purpose**:
  Throws an exception if the key is missing—useful during development to catch missing translations early.

* **Usage Example**

  ```csharp
  try
  {
      string text = LocalizationManager.GetOrThrow("HUD.SCORE");
      scoreLabel.text = text;
  }
  catch (KeyNotFoundException e)
  {
      Debug.LogError(e.Message);
      scoreLabel.text = "Score"; // fallback
  }
  ```

---

### 4.4 Case Variants

#### `GetTitleCase(string key, params object[] args)`

```csharp
public static string GetTitleCase(string key, params object[] args)
{
    try
    {
        var culture = CultureInfo.GetCultureInfo(_currentLanguage);
        var textInfo = culture.TextInfo;
        return textInfo.ToTitleCase(Get(key, args).ToLower(culture));
    }
    catch
    {
        return Get(key, args);
    }
}
```

* **Purpose**:
  Fetch the localized string (via `Get(key, args)`), convert it to lowercase under the current culture, then re‐apply Title Case (capitalizing the first letter of each word) according to cultural rules.

* **Usage Example**

  ```csharp
  // In en.json: "menu.settings" = "Game Settings"
  string title = LocalizationManager.GetTitleCase("menu.settings");
  // => "Game Settings" (already title‐cased)

  // In fr.json: "menu.settings" = "paramètres du jeu"
  string titleFr = LocalizationManager.GetTitleCase("menu.settings");
  // => "Paramètres Du Jeu" (French Title Case)
  ```

#### `GetUpper(string key, params object[] args)`

```csharp
public static string GetUpper(string key, params object[] args)
{
    try
    {
        var culture = CultureInfo.GetCultureInfo(_currentLanguage);
        return Get(key, args).ToUpper(culture);
    }
    catch
    {
        return Get(key, args).ToUpper();
    }
}
```

* **Purpose**:
  Return the translated string in UPPERCASE, using the current culture’s rules (e.g. Turkish dotted/dotless “i”).

* **Usage Example**

  ```csharp
  // In en: "alert" => "Alert"
  LocalizationManager.GetUpper("alert"); // => "ALERT"

  // In tr: "warning" => "uyarı"
  LocalizationManager.GetUpper("warning"); // => "UYARI" (Turkish uppercasing rules)
  ```

#### `GetLower(string key, params object[] args)`

```csharp
public static string GetLower(string key, params object[] args)
{
    try
    {
        var culture = CultureInfo.GetCultureInfo(_currentLanguage);
        return Get(key, args).ToLower(culture);
    }
    catch
    {
        return Get(key, args).ToLower();
    }
}
```

* **Purpose**:
  Return the translated string in lowercase, respecting culture‐specific casing.

* **Usage Example**

  ```csharp
  LocalizationManager.GetLower("MENU.PLAY"); // e.g. "play" or culture‐specific variant
  ```

---

### 4.5 Whitespace & Rich‐Text Variants

#### `GetPlain(string key, params object[] args)`

```csharp
public static string GetPlain(string key, params object[] args)
{
    var str = Get(key, args);
    return System.Text.RegularExpressions.Regex.Replace(str, "<.*?>", string.Empty);
}
```

* **Purpose**:
  Retrieves the translation, then removes all `<…>` tags (e.g. `<b>`, `<i>`, `<color=…>`). Useful if you need to display text without Unity‐rich‐text formatting.

* **Usage Example**

  ```csharp
  // In en.json: "score" = "<b>Score:</b> {0}"
  string raw = LocalizationManager.GetPlain("score", 100);
  // => "Score: 100" (no <b> tags)
  ```

#### `GetCollapsedWhitespace(string key, params object[] args)`

```csharp
public static string GetCollapsedWhitespace(string key, params object[] args)
{
    var str = Get(key, args);
    return System.Text.RegularExpressions.Regex.Replace(str, @"\s+", " ").Trim();
}
```

* **Purpose**:
  Collapse any sequence of whitespace (spaces, tabs, newlines) into a single space. Useful if your JSON has line breaks or indentations.

* **Usage Example**

  ```csharp
  // In fr.json: 
  // "description" = "Ligne 1\nLigne 2  \n Ligne 3"
  string collapsed = LocalizationManager.GetCollapsedWhitespace("description");
  // => "Ligne 1 Ligne 2 Ligne 3"
  ```

---

### 4.6 Key Existence & Enumeration

#### `HasKey(string key)`

```csharp
public static bool HasKey(string key)
{
    return _translations.ContainsKey(key);
}
```

* **Purpose**:
  Quickly check whether a particular translation key is present (in the **current** language’s dictionary).

* **Usage Example**

  ```csharp
  if (!LocalizationManager.HasKey("UI.EXIT_BUTTON"))
  {
      Debug.LogWarning("Missing translation for EXIT_BUTTON");
  }
  ```

#### `GetAllKeys()`

```csharp
public static IEnumerable<string> GetAllKeys()
    => _translations.Keys;
```

* **Purpose**:
  Return an enumeration of every key loaded into `_translations` for the current language.

* **Usage Example**

  ```csharp
  foreach (var key in LocalizationManager.GetAllKeys())
  {
      Debug.Log($"Loaded key: {key}");
  }
  ```

#### `GetAllWithPrefix(string prefix)`

```csharp
public static Dictionary<string, string> GetAllWithPrefix(string prefix)
{
    return _translations
        .Where(kvp => kvp.Key.StartsWith(prefix, StringComparison.OrdinalIgnoreCase))
        .ToDictionary(kvp => kvp.Key, kvp => kvp.Value, StringComparer.OrdinalIgnoreCase);
}
```

* **Purpose**:
  Return a new dictionary containing only those entries whose keys start with the given `prefix` (case‐insensitive). Useful for grouping (e.g., all `"error.*"`, all `"menu.*"`).

* **Usage Example**

  ```csharp
  var errorMessages = LocalizationManager.GetAllWithPrefix("error.");
  // errorMessages might contain keys: "error.network", "error.server", etc.
  ```

---

### 4.7 `GetRandom(string baseKey, int count, params object[] args)`

```csharp
public static string GetRandom(string baseKey, int count, params object[] args)
{
    if (count <= 0)
        return Get(baseKey, args);
    int index = UnityEngine.Random.Range(1, count + 1);
    return Get($"{baseKey}.{index}", args);
}
```

* **Purpose**:
  If you have translations like `"error.1"`, `"error.2"`, `"error.3"`, you can call `GetRandom("error", 3)` and it will pick one of `"error.1"`, `"error.2"`, or `"error.3"` at random. If `count <= 0`, it just returns `Get(baseKey)`.

* **Usage Example**

  ```csharp
  // en.json:
  // { "key":"error.1", "value":"Oops, something went wrong." }
  // { "key":"error.2", "value":"That didn’t work—try again." }
  // { "key":"error.3", "value":"Error occurred! Please retry." }

  string randomError = LocalizationManager.GetRandom("error", 3);
  // => one of the three messages above, at random
  ```

---

### 4.8 `GetLocalizedText(string key)`

```csharp
public static string GetLocalizedText(string key)
{
    if (string.IsNullOrEmpty(key)) 
        return key;

    // 1. Try current language
    if (!string.IsNullOrEmpty(_currentLanguage) &&
        _translations.TryGetValue(key, out string translation))
    {
        return translation;
    }

    // 2. Try fallback language
    if (_fallbackLanguage != _currentLanguage)
    {
        var fallbackTranslations = LoadTranslations(_fallbackLanguage);
        if (fallbackTranslations.TryGetValue(key, out translation))
        {
            return translation;
        }
    }

    // 3. Return key if missing
    return key;
}
```

* **Purpose**:
  Similar to `Get()`, but instead of returning `"[key]"` on a missing translation, it falls back in two steps:

  1. If `key` is in `_translations` (current language), return it.
  2. Otherwise, load the fallback language’s translations (via `LoadTranslations`) and check there. If found, return that.
  3. If still missing, return `key` itself (so a UI element bound to this string shows “HELLO” rather than `"[HELLO]"`).

* **Usage Example**

  ```csharp
  string label = LocalizationManager.GetLocalizedText("UI.RESUME");
  // If “UI.RESUME” exists in French (current) → "Reprendre"
  // Else if exists in English (fallback) → "Resume"
  // Else → "UI.RESUME"
  ```

---

## 5. Formatting Helpers (Culture‐Aware)

All of these rely on .NET’s `System.Globalization.CultureInfo` to format according to the conventions of the current language (e.g. date formats, decimal separators).

### 5.1 `FormatDate(DateTime date, string format = null)`

```csharp
public static string FormatDate(DateTime date, string format = null)
{
    try
    {
        var culture = CultureInfo.GetCultureInfo(_currentLanguage);
        return format != null
            ? date.ToString(format, culture)
            : date.ToLocalTime().ToString(culture);
    }
    catch
    {
        return date.ToString(format);
    }
}
```

* **Purpose**

  * If `format` is provided (e.g. `"yyyy MMM dd"`), format `date` with that pattern under the current culture.
  * If `format` is `null`, call `date.ToLocalTime().ToString(culture)`, which uses the culture’s default date/time format.
  * If `_currentLanguage` is invalid or missing, it falls back to `date.ToString(format)` (invariant).

* **Usage Example**

  ```csharp
  DateTime now = DateTime.UtcNow;
  // If currentLanguage == "en-US":
  LocalizationManager.FormatDate(now);            // e.g. "8/24/2023 4:15:30 PM"
  LocalizationManager.FormatDate(now, "D");       // e.g. "Thursday, August 24, 2023"

  // If currentLanguage == "fr-FR":
  LocalizationManager.FormatDate(now);            // e.g. "24/08/2023 16:15:30"
  LocalizationManager.FormatDate(now, "D");       // e.g. "jeudi 24 août 2023"
  ```

---

### 5.2 `FormatNumber(decimal number, string format = null)`

```csharp
public static string FormatNumber(decimal number, string format = null)
{
    try
    {
        var culture = CultureInfo.GetCultureInfo(_currentLanguage);
        return format != null
            ? number.ToString(format, culture)
            : number.ToString(culture);
    }
    catch
    {
        return number.ToString(format);
    }
}
```

* **Purpose**

  * If `format` is provided (e.g. `"N2"` for two decimals, `"P"` for percent), use that format under the current culture.
  * If `format` is `null`, simply call `number.ToString(culture)`, which applies thousands separators and decimal separators appropriate to the culture (e.g. `1 234 567,89` in French).

* **Usage Example**

  ```csharp
  decimal value = 1234567.89m;

  // In en-US:
  LocalizationManager.FormatNumber(value);      // "1,234,567.89"
  LocalizationManager.FormatNumber(value, "N0");// "1,234,568"
  LocalizationManager.FormatNumber(0.153m, "P"); // "15.30 %"

  // In fr-FR:
  LocalizationManager.ChangeLanguage("fr-FR");
  LocalizationManager.FormatNumber(value);      // "1 234 567,89"
  LocalizationManager.FormatNumber(value, "N0");// "1 234 568"
  LocalizationManager.FormatNumber(0.153m, "P"); // "15,30 %"
  ```

---

### 5.3 `FormatCurrency(decimal amount, string currencyCode = null)`

```csharp
public static string FormatCurrency(decimal amount, string currencyCode = null)
{
    try
    {
        var culture = CultureInfo.GetCultureInfo(_currentLanguage);
        if (currencyCode != null)
        {
            culture = (CultureInfo)culture.Clone();
            culture.NumberFormat.CurrencySymbol = currencyCode;
        }
        return amount.ToString("C", culture);
    }
    catch
    {
        return amount.ToString("C");
    }
}
```

* **Purpose**

  * If `currencyCode` is `null`, use the current culture’s default currency symbol (e.g. `$` for `en-US`, `€` for `fr-FR`).
  * If `currencyCode` is provided (e.g. `"USD"`, `"EUR"`), clone the `CultureInfo` and override its `NumberFormat.CurrencySymbol` to that code, then format with `"C"`.
  * If anything fails, fallback to `amount.ToString("C")` (invariant culture).

* **Usage Example**

  ```csharp
  decimal price = 1234.56m;

  LocalizationManager.ChangeLanguage("en-US");
  LocalizationManager.FormatCurrency(price);         // "$1,234.56"
  LocalizationManager.FormatCurrency(price, "USD");  // "USD1,234.56"

  LocalizationManager.ChangeLanguage("fr-FR");
  LocalizationManager.FormatCurrency(price);         // "1 234,56 €"
  LocalizationManager.FormatCurrency(price, "EUR");  // "EUR 1 234,56"
  ```

---

### 5.4 `GetRelativeTime(DateTime dateTime)`

```csharp
public static string GetRelativeTime(DateTime dateTime)
{
    var timeSpan = DateTime.Now - dateTime;
    bool isFuture = timeSpan.TotalMilliseconds < 0;
    timeSpan = timeSpan.Duration();

    if (timeSpan.TotalSeconds < 60)
        return isFuture ? Get("time.in_seconds", (int)timeSpan.TotalSeconds)
                        : Get("time.seconds_ago", (int)timeSpan.TotalSeconds);
    if (timeSpan.TotalMinutes < 60)
        return isFuture ? Get("time.in_minutes", (int)timeSpan.TotalMinutes)
                        : Get("time.minutes_ago", (int)timeSpan.TotalMinutes);
    if (timeSpan.TotalHours < 24)
        return isFuture ? Get("time.in_hours", (int)timeSpan.TotalHours)
                        : Get("time.hours_ago", (int)timeSpan.TotalHours);
    if (timeSpan.TotalDays < 30)
        return isFuture ? Get("time.in_days", (int)timeSpan.TotalDays)
                        : Get("time.days_ago", (int)timeSpan.TotalDays);
    if (timeSpan.TotalDays < 365)
        return isFuture ? Get("time.in_months", (int)(timeSpan.TotalDays / 30))
                        : Get("time.months_ago", (int)(timeSpan.TotalDays / 30));

    return isFuture ? Get("time.in_years", (int)(timeSpan.TotalDays / 365))
                    : Get("time.years_ago", (int)(timeSpan.TotalDays / 365));
}
```

* **Purpose**

  * Compute the time span between now and `dateTime`.
  * If `dateTime` is in the future, use `"time.in_*"` keys; otherwise use `"time.*_ago"` keys.
  * Depending on the magnitude, choose seconds, minutes, hours, days, months (≈30 days), or years (≈365 days).
  * Each translation must have keys like:

    ```jsonc
    { "key":"time.seconds_ago","value":"{0} seconds ago" }
    { "key":"time.in_seconds",   "value":"in {0} seconds" }
    { "key":"time.minutes_ago","value":"{0} minutes ago" }
    { "key":"time.in_minutes",  "value":"in {0} minutes" }
    // … and so on for hours, days, months, years
    ```

* **Usage Example**

  ```csharp
  // Suppose today is 2023-08-24 12:00
  DateTime fiveMinutesAgo = DateTime.Now.AddMinutes(-5);
  string rel1 = LocalizationManager.GetRelativeTime(fiveMinutesAgo);
  // => "5 minutes ago" (if en) or "il y a 5 minutes" (if fr, given the proper JSON entries)

  DateTime inTwoDays = DateTime.Now.AddDays(2);
  string rel2 = LocalizationManager.GetRelativeTime(inTwoDays);
  // => "in 2 days" or "dans 2 jours"
  ```

---

### 5.5 `FormatList(IEnumerable<string> items)`

```csharp
public static string FormatList(IEnumerable<string> items)
{
    var itemsList = items.ToList();
    if (itemsList.Count == 0) 
        return string.Empty;
    if (itemsList.Count == 1) 
        return itemsList[0];
    if (itemsList.Count == 2) 
        return Get("list.two_items", itemsList[0], itemsList[1]);

    var allButLast = string.Join(Get("list.separator"), itemsList.Take(itemsList.Count - 1));
    return Get("list.format", allButLast, itemsList.Last());
}
```

* **Purpose**

  * If there are zero items, return `""`.
  * If one item, return it directly.
  * If two items, use the key `"list.two_items"` with `{0}` and `{1}` placeholders.
  * If ≥ 3, join all but the last with the `"list.separator"` (e.g. `", "` or `"; "`), then pass those two parts into `"list.format"`—for instance, `"{0}, and {1}"` in English, `"{0} et {1}"` in French.

* **JSON Conventions** (en.json):

  ```jsonc
  { "key":"list.separator", "value":", " }
  { "key":"list.two_items","value":"{0} and {1}" }
  { "key":"list.format",   "value":"{0}, and {1}" }
  ```

  (fr.json):

  ```jsonc
  { "key":"list.separator", "value":", " }
  { "key":"list.two_items","value":"{0} et {1}" }
  { "key":"list.format",   "value":"{0} et {1}" }
  ```

* **Usage Example**

  ```csharp
  var fruits1 = new List<string> { "apple" };
  LocalizationManager.FormatList(fruits1); // => "apple"

  var fruits2 = new List<string> { "apple", "banana" };
  LocalizationManager.FormatList(fruits2); // => "apple and banana" (en)

  var fruits3 = new List<string> { "apple", "banana", "orange" };
  LocalizationManager.FormatList(fruits3); // => "apple, banana, and orange" (en)
  // In French: "apple, banana et orange"
  ```

---

## 6. Currency Parsing Helpers

Sometimes you need to parse user‐entered price strings or read a price text and extract the symbol/amount. The following methods help with that.

### 6.1 `TryParsePrice(string priceString, out string currencySymbol, out decimal amount)`

```csharp
public static bool TryParsePrice(string priceString, out string currencySymbol, out decimal amount)
{
    currencySymbol = null;
    amount = 0m;
    if (string.IsNullOrWhiteSpace(priceString))
        return false;

    priceString = priceString.Trim();
    // Find the first digit in the string
    int digitIndex = priceString.IndexOfAny("0123456789".ToCharArray());
    if (digitIndex == -1)
        return false;

    currencySymbol = priceString.Substring(0, digitIndex).Trim();
    string numberPart = priceString.Substring(digitIndex).Trim();
    if (decimal.TryParse(numberPart, NumberStyles.Any, CultureInfo.InvariantCulture, out amount))
        return true;
    // Try with current culture as fallback
    if (decimal.TryParse(numberPart, NumberStyles.Any, CultureInfo.CurrentCulture, out amount))
        return true;
    return false;
}
```

* **Purpose**

  1. Strip leading/trailing whitespace, find the first digit.
  2. Everything before that digit is treated as the `currencySymbol` (e.g. `"$"`, `"€"`, `"¥"`).
  3. Everything from that digit onward is the numeric part, which we attempt to parse first under `InvariantCulture`, then under `CurrentCulture`.
  4. Returns `true` if parsing succeeded, `false` otherwise.

* **Usage Example**

  ```csharp
  if (LocalizationManager.TryParsePrice("$12.34", out var symbol, out var amt))
  {
      Debug.Log($"Symbol: {symbol}, Amount: {amt}"); // "$" and 12.34
  }

  if (LocalizationManager.TryParsePrice("€1 234,56", out symbol, out amt))
  {
      Debug.Log($"Symbol: {symbol}, Amount: {amt}"); // "€" and 1234.56 (assuming fr-FR culture)
  }
  ```

---

### 6.2 `ExtractCurrencySymbol(string priceString)`

```csharp
public static string ExtractCurrencySymbol(string priceString)
{
    if (string.IsNullOrWhiteSpace(priceString))
        return null;
    priceString = priceString.Trim();
    int digitIndex = priceString.IndexOfAny("0123456789".ToCharArray());
    if (digitIndex == -1)
        return null;
    return priceString.Substring(0, digitIndex).Trim();
}
```

* **Purpose**:
  Return only whatever appears before the first digit (trimmed)—likely the currency symbol. Returns `null` if no digits found.

* **Usage Example**

  ```csharp
  string sym1 = LocalizationManager.ExtractCurrencySymbol("$99.99"); // "$"
  string sym2 = LocalizationManager.ExtractCurrencySymbol("£1234");  // "£"
  string sym3 = LocalizationManager.ExtractCurrencySymbol("123");    // "" or null (no symbol)
  ```

---

### 6.3 `ExtractPriceAmount(string priceString)`

```csharp
public static decimal? ExtractPriceAmount(string priceString)
{
    if (string.IsNullOrWhiteSpace(priceString))
        return null;
    priceString = priceString.Trim();
    int digitIndex = priceString.IndexOfAny("0123456789".ToCharArray());
    if (digitIndex == -1)
        return null;
    string numberPart = priceString.Substring(digitIndex).Trim();
    if (decimal.TryParse(numberPart, NumberStyles.Any, CultureInfo.InvariantCulture, out var amount))
        return amount;
    if (decimal.TryParse(numberPart, NumberStyles.Any, CultureInfo.CurrentCulture, out amount))
        return amount;
    return null;
}
```

* **Purpose**:
  Return only the numeric portion after the currency symbol, parsed as `decimal`. Returns `null` if parsing fails.

* **Usage Example**

  ```csharp
  decimal? amt1 = LocalizationManager.ExtractPriceAmount("$12.34");   // 12.34m
  decimal? amt2 = LocalizationManager.ExtractPriceAmount("€1 234,56"); // 1234.56m (fr-FR)
  decimal? amt3 = LocalizationManager.ExtractPriceAmount("abc");      // null
  ```

---

## 7. Language Metadata & Utilities

### 7.1 `GetLanguageDisplayName(string langCode)`

```csharp
public static string GetLanguageDisplayName(string langCode)
{
    try
    {
        var culture = CultureInfo.GetCultureInfo(langCode);
        return culture.EnglishName;
    }
    catch
    {
        return langCode;
    }
}
```

* **Purpose**:
  Given a code like `"fr-FR"`, return `"French (France)"`. If the code is invalid, return the code itself.

* **Usage Example**

  ```csharp
  LocalizationManager.GetLanguageDisplayName("en");     // "English"
  LocalizationManager.GetLanguageDisplayName("fr-FR");  // "French (France)"
  LocalizationManager.GetLanguageDisplayName("xx-YY");  // "xx-YY" (fallback)
  ```

---

### 7.2 `GetLanguageNativeName(string langCode)`

```csharp
public static string GetLanguageNativeName(string langCode)
{
    try
    {
        var culture = CultureInfo.GetCultureInfo(langCode);
        return culture.NativeName;
    }
    catch
    {
        return langCode;
    }
}
```

* **Purpose**:
  Return the language’s native name (e.g. `"Français (France)"` for `"fr-FR"`).

* **Usage Example**

  ```csharp
  LocalizationManager.GetLanguageNativeName("en");    // "English"
  LocalizationManager.GetLanguageNativeName("fr-FR"); // "français (France)"
  ```

---

### 7.3 `GetAvailableLanguageDisplayNames()` / `GetAvailableLanguageNativeNames()`

```csharp
public static List<string> GetAvailableLanguageDisplayNames()
{
    var codes = GetAvailableLanguages();
    var names = new List<string>();
    foreach (var code in codes)
        names.Add(GetLanguageDisplayName(code));
    return names;
}

public static List<string> GetAvailableLanguageNativeNames()
{
    var codes = GetAvailableLanguages();
    var names = new List<string>();
    foreach (var code in codes)
        names.Add(GetLanguageNativeName(code));
    return names;
}
```

* **Purpose**:
  Build upon `GetAvailableLanguages()` to return lists of user‐friendly names—either in English or in the language’s own script.

* **Usage Example**

  ```csharp
  var codes = LocalizationManager.GetAvailableLanguages();            // e.g. ["en","fr","es"]
  var displayNames = LocalizationManager.GetAvailableLanguageDisplayNames(); // ["English","French","Spanish"]
  var nativeNames = LocalizationManager.GetAvailableLanguageNativeNames();   // ["English","français","español"]
  ```

---

### 7.4 `GetSystemLanguageCode()`

```csharp
public static string GetSystemLanguageCode()
{
    return Application.systemLanguage.ToString();
}
```

* **Purpose**:
  Return Unity’s `Application.systemLanguage` enum value as a string (e.g. `"English"`, `"French"`). Use this to suggest a default on first launch.

* **Usage Example**

  ```csharp
  string sysLang = LocalizationManager.GetSystemLanguageCode();
  Debug.Log($"System language: {sysLang}");
  ```

---

## 8. JSON Table Structure

### 8.1 Example JSON

Place under `Assets/Resources/Localization/en.json` (as an example):

```jsonc
{
  "entries": [
    { "key": "HELLO",            "value": "Hello" },
    { "key": "GOODBYE",          "value": "Goodbye" },
    { "key": "items.singular",   "value": "You have {0} item" },
    { "key": "items.plural",     "value": "You have {0} items" },
    { "key": "time.seconds_ago", "value": "{0} seconds ago" },
    { "key": "time.in_seconds",  "value": "in {0} seconds" },
    { "key": "list.separator",   "value": ", " },
    { "key": "list.two_items",   "value": "{0} and {1}" },
    { "key": "list.format",      "value": "{0}, and {1}" },
    { "key": "menu.play",        "value": "Play" },
    { "key": "menu.settings",    "value": "Settings" }
    // … etc.
  ]
}
```

* The top‐level object must contain an array named `"entries"`.
* Each entry has a `"key"` and a `"value"`.
* Duplicate keys are ignored—only the first occurrence is stored.

---

### 8.2 Internal `LocalizationTable` → `Dictionary<string,string>`

```csharp
[Serializable]
private class LocalizationTable
{
    public Entry[] entries;
    [Serializable]
    public struct Entry { public string key; public string value; }

    public Dictionary<string,string> ToDictionary()
    {
        var dict = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
        if (entries != null)
        {
            foreach (var e in entries)
            {
                if (!dict.ContainsKey(e.key))
                    dict.Add(e.key, e.value);
            }
        }
        return dict;
    }
}
```

* **Purpose**:

  * `JsonUtility.FromJson<LocalizationTable>(text)` parses the JSON into a `LocalizationTable` object.
  * Calling `ToDictionary()` builds a case‐insensitive `Dictionary<string,string>` so lookups ignore casing.

---

## 9. Real‐World Usage Examples

Below are sample scenarios illustrating how to integrate **LocalizationManager** into a Unity project.

---

### 9.1 Basic Setup & Switching Languages

1. **Place JSON Files** under `Assets/Resources/Localization/`:

   * `en.json`, `fr.json`, `es-US.json`, etc.

2. **Call Initialize at Startup**:

   ```csharp
   public class Startup : MonoBehaviour
   {
       void Awake()
       {
           LocalizationManager.Initialize("en");
           // Automatically loads Application.systemLanguage or falls back to "en".
       }
   }
   ```

3. **Subscribe to Language Changes** (optional):

   ```csharp
   public class UITranslator : MonoBehaviour
   {
       void OnEnable()
       {
           LocalizationManager.OnLanguageChanged += OnLanguageChanged;
       }

       void OnDisable()
       {
           LocalizationManager.OnLanguageChanged -= OnLanguageChanged;
       }

       private void OnLanguageChanged(string newLang)
       {
           RefreshAllUIText();
       }

       private void RefreshAllUIText()
       {
           // E.g. find all UILocalizedText components and call UpdateText().
       }
   }
   ```

4. **Switch Language via UI Button**:

   ```csharp
   public class LanguageButton : MonoBehaviour
   {
       public void OnEnglishClicked() => LocalizationManager.ChangeLanguage("en");
       public void OnFrenchClicked()  => LocalizationManager.ChangeLanguage("fr");
       public void OnSpanishClicked() => LocalizationManager.ChangeLanguage("es-US");
   }
   ```

---

### 9.2 Retrieving a Simple Translation

```csharp
public class WelcomeScreen : MonoBehaviour
{
    public UnityEngine.UI.Text welcomeLabel;

    void Start()
    {
        string hello = LocalizationManager.Get("HELLO");
        welcomeLabel.text = hello; 
        // If en: "Hello", if fr: "Bonjour", else "[HELLO]" if missing
    }
}
```

---

### 9.3 Formatting with Arguments (`Get`)

```csharp
// en.json: { "key":"USER_GREETING","value":"Welcome back, {0}!" }
// fr.json: { "key":"USER_GREETING","value":"Bienvenue, {0}!" }

public class PlayerWelcome : MonoBehaviour
{
    public UnityEngine.UI.Text greetingLabel;

    void ShowGreeting(string playerName)
    {
        string template = LocalizationManager.Get("USER_GREETING", playerName);
        greetingLabel.text = template;
    }
}
```

* If `playerName = "Alice"` and currentLanguage = `"en"`, label shows `"Welcome back, Alice!"`.

---

### 9.4 Pluralization (`GetPlural`)

```csharp
// en.json:
//   { "key":"coins.singular","value":"You have {0} coin" }
//   { "key":"coins.plural",  "value":"You have {0} coins" }

// fr.json:
//   { "key":"coins.singular","value":"Vous avez {0} pièce" }
//   { "key":"coins.plural",  "value":"Vous avez {0} pièces" }

public class CoinDisplay : MonoBehaviour
{
    public UnityEngine.UI.Text coinsLabel;

    void UpdateCoins(int coinCount)
    {
        string text = LocalizationManager.GetPlural("coins", coinCount);
        coinsLabel.text = text;
    }
}
```

* If `coinCount = 1` and language = `"en"`, shows `"You have 1 coin"`.
* If `coinCount = 5` and language = `"fr"`, shows `"Vous avez 5 pièces"`.

---

### 9.5 Date, Number, and Currency Formatting

```csharp
public class StatsDisplay : MonoBehaviour
{
    public UnityEngine.UI.Text dateLabel;
    public UnityEngine.UI.Text scoreLabel;
    public UnityEngine.UI.Text balanceLabel;

    void UpdateStats(DateTime lastLogin, decimal highScore, decimal balance)
    {
        dateLabel.text = LocalizationManager.FormatDate(lastLogin, "D");
        // e.g. "Thursday, August 24, 2023" (en) or "jeudi 24 août 2023" (fr)

        scoreLabel.text = LocalizationManager.FormatNumber(highScore, "N0");
        // e.g. "1,234,567" (en) or "1 234 567" (fr)

        balanceLabel.text = LocalizationManager.FormatCurrency(balance);
        // e.g. "$123.45" (en) or "123,45 €" (fr)
    }
}
```

* Change the current language at runtime and these calls automatically use the correct cultural conventions.

---

### 9.6 Relative Time (`GetRelativeTime`)

```csharp
public class Notification : MonoBehaviour
{
    public UnityEngine.UI.Text timestampLabel;
    private DateTime messageTime;

    void Start()
    {
        messageTime = DateTime.Now.AddHours(-3); // message sent 3 hours ago
        timestampLabel.text = LocalizationManager.GetRelativeTime(messageTime);
        // => "3 hours ago" (en) or "il y a 3 heures" (fr)
    }
}
```

* If `messageTime` is in the future, it will produce `"in X hours"` instead.

---

### 9.7 List Formatting (`FormatList`)

```csharp
public class InventoryDisplay : MonoBehaviour
{
    public UnityEngine.UI.Text inventoryLabel;

    void ShowInventory(List<string> items)
    {
        // items might be localized item names already
        inventoryLabel.text = LocalizationManager.FormatList(items);
        // => "Sword, Shield, and Potion" (en)
        // => "Épée, Bouclier et Potion" (fr)
    }
}
```

* If only two items (`["Sword","Shield"]`), it uses `"list.two_items"`; if ≥3, it uses `"list.format"`.

---

### 9.8 Randomized Variations (`GetRandom`)

```csharp
// en.json:
//   { "key":"taunt.1","value":"Is that all you got?" }
//   { "key":"taunt.2","value":"Come on, do better!" }
//   { "key":"taunt.3","value":"You call that a move?" }

public class EnemyAI : MonoBehaviour
{
    public void TauntPlayer()
    {
        string taunt = LocalizationManager.GetRandom("taunt", 3);
        Debug.Log(taunt);
    }
}
```

* Each call to `GetRandom("taunt", 3)` picks one of `"taunt.1"`, `"taunt.2"`, or `"taunt.3"` at random.

---

### 9.9 Case Transformations (`GetTitleCase`, `GetUpper`, `GetLower`)

```csharp
public class UIHeader : MonoBehaviour
{
    public UnityEngine.UI.Text headerText;

    void Start()
    {
        // Suppose en.json: "main_menu" = "main menu"
        string raw = LocalizationManager.Get("main_menu");
        headerText.text = LocalizationManager.GetTitleCase("main_menu");
        // => "Main Menu"

        // If you want an all‐caps label:
        headerText.text = LocalizationManager.GetUpper("main_menu"); // => "MAIN MENU"
    }
}
```

---

### 9.10 Currency Parsing (`TryParsePrice`, `ExtractCurrencySymbol`, `ExtractPriceAmount`)

```csharp
public class PriceInputField : MonoBehaviour
{
    public UnityEngine.UI.InputField priceField;
    public UnityEngine.UI.Text symbolLabel;

    void OnPriceChanged(string text)
    {
        if (LocalizationManager.TryParsePrice(text, out var symbol, out var amount))
        {
            symbolLabel.text = symbol;         // e.g. "$" or "€"
            Debug.Log($"Parsed amount: {amount}");
        }
        else
        {
            symbolLabel.text = "";
            Debug.Log("Invalid price format.");
        }
    }
}
```

* If user enters `"￥5000"`, `symbolLabel` shows `"￥"`, and `amount` is `5000m`.

---

### 9.11 Enumerating Keys & Prefixes

```csharp
public class DebugLocalization : MonoBehaviour
{
    void OnGUI()
    {
        if (GUILayout.Button("Log All Keys"))
        {
            foreach (var key in LocalizationManager.GetAllKeys())
                Debug.Log(key);
        }

        if (GUILayout.Button("Log All Errors"))
        {
            var errors = LocalizationManager.GetAllWithPrefix("error.");
            foreach (var kvp in errors)
                Debug.Log($"{kvp.Key} = {kvp.Value}");
        }
    }
}
```

* Use `GetAllWithPrefix("menu.")` to find all menu‐related translations.

---

### 9.12 Fallback Variants (`GetOrDefault`, `GetOrNull`, `GetOrThrow`)

```csharp
public class SettingsUI : MonoBehaviour
{
    public UnityEngine.UI.Text advancedLabel;

    void Start()
    {
        // Suppose "settings.advanced" might not exist in some languages:
        string adv = LocalizationManager.GetOrDefault("settings.advanced", "Advanced");
        advancedLabel.text = adv;
    }
}
```

* If `"settings.advanced"` is missing in JSON, you get `"Advanced"` instead of `"[settings.advanced]"` or `null` or exception (depending on which variant you choose).

---

## 10. Best Practices & Tips

1. **Keep Translation Files Small & Modular**

   * If you have hundreds of keys, consider splitting into multiple JSONs per category (e.g. `ui.json`, `gameplay.json`) and loading/merging them yourself. This manager always expects a single JSON per language under `Localization/`.

2. **Use Consistent Key Naming**

   * Stick to a convention like `CATEGORY.ITEM_DESCRIPTION` or `UI.BUTTON.PLAY`.
   * Keep keys lowercase or uppercase—`Dictionary` is case‐insensitive, but your authors can be consistent.

3. **Always Provide a Fallback Language**

   * If you don’t ship a full translation in French, put at least the English keys in `fr.json` that you want to override. Or rely on fallback by leaving out certain keys—`GetLocalizedText` will return the English version or the raw key.

4. **Test Pluralization in All Languages**

   * Some languages have more than two plural forms (e.g. Russian, Arabic). The simple `.singular` vs. `.plural` logic only covers “1” vs. “not 1.” For more complex rules, you’ll need to extend this system (e.g. detect `count % 10`).

5. **Use `GetRandom` for Variety**

   * Avoid static repeated text—if you have multiple variants for “error” or “level up,” use `GetRandom("error", 3)` so the player sees diversity.

6. **Be Careful with Rich Text Tags**

   * If you include `<color>` or `<b>` tags in your JSON, any call to `GetPlain()` will strip them. Use `GetPlain` only when you need raw text without formatting.

7. **Remember `Resources.Load` Limitations**

   * `Resources.Load<...>("Localization/en")` works at runtime. But if you rely on `GetAvailableLanguages()` in a build, Unity cannot list files under `Resources`; you must maintain your own manifest or rely on the Editor‐only version.

8. **Handle Missing Keys Gracefully**

   * Use `GetOrDefault` or `TryGet` if you expect some keys to be missing. In production, avoid throwing exceptions for missing keys (use `GetOrDefault(key, fallback)`).

9. **Hot‐Reload in Editor for Faster Iteration**

   * Call `LocalizationManager.ReloadCurrentLanguage()` after editing the JSON in the Editor to see immediate changes without recompiling.

10. **Be Mindful of Culture Codes**

    * `CultureInfo.GetCultureInfo("en")` returns English neutral. For region‐specific formatting (dates, numbers), use culture codes like `"en-US"`, `"fr-FR"`.
    * In JSON filenames, match those codes exactly (e.g. `fr-FR.json`, `en-US.json`). Otherwise, `GetCultureInfo` may throw.

---

## 11. Summary of Public API

Below is a quick glance at all public methods and properties on **LocalizationManager**. See the sections above for detailed explanations and examples.

### Initialization & Language Switching

| Signature                                                              | Description                                                                                                                            |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `static void Initialize(string defaultLanguage = "en")`                | Set `fallbackLanguage`, attempt to load system language (or fallback); populates `_translations`.                                      |
| `static bool ChangeLanguage(string langCode, bool useFallback = true)` | Load `Resources/Localization/{langCode}.json`, or fallback if missing. Populate `_translations` and fire `OnLanguageChanged`.          |
| `static void ReloadCurrentLanguage()`                                  | Re‐loads JSON for `_currentLanguage` (useful in Editor for hot‐reload).                                                                |
| `static List<string> GetAvailableLanguages()`                          | **Editor**: return all JSON filenames under `Assets/Resources/Localization/`. **Build**: return `{currentLanguage, fallbackLanguage}`. |
| `static string CurrentLanguage { get; }`                               | Read‐only property holding the currently loaded culture code (e.g. `"en"`, `"fr"`).                                                    |
| `static string FallbackLanguage { get; }`                              | Read‐only property for the fallback culture code (as set via `Initialize` or `ChangeLanguage`).                                        |
| `static string GetSystemLanguageCode()`                                | Returns `Application.systemLanguage.ToString()` (Unity’s system language enum as a string).                                            |

### Retrieving Translations

| Signature                                                                       | Description                                                                                                                       |
| ------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `static string Get(string key, params object[] args)`                           | Return translation for `key`, or `"[key]"` if missing. Optionally use `string.Format` on `{0}`, `{1}`, etc.                       |
| `static string GetPlural(string key, int count, params object[] extraArgs)`     | Look up `"{key}.singular"` if `count == 1`, else `"{key}.plural"`, then insert `count` + any extra arguments.                     |
| `static bool TryGet(string key, out string value)`                              | Return `true` and `value` if `key` exists in `_translations`; otherwise `false` and `value = null`.                               |
| `static string GetOrDefault(string key, string fallback, params object[] args)` | If `key` exists, return its translation (formatted); otherwise return `fallback` (formatted).                                     |
| `static string GetOrNull(string key, params object[] args)`                     | Return translation or `null`.                                                                                                     |
| `static string GetOrThrow(string key, params object[] args)`                    | Return translation or throw `KeyNotFoundException`.                                                                               |
| `static string GetTitleCase(string key, params object[] args)`                  | Return translation in Title Case according to current culture.                                                                    |
| `static string GetUpper(string key, params object[] args)`                      | Return translation in UPPERCASE according to current culture.                                                                     |
| `static string GetLower(string key, params object[] args)`                      | Return translation in lowercase according to current culture.                                                                     |
| `static string GetPlain(string key, params object[] args)`                      | Return translation with all `<…>` tags stripped.                                                                                  |
| `static string GetCollapsedWhitespace(string key, params object[] args)`        | Return translation with sequences of whitespace replaced by a single space.                                                       |
| `static bool HasKey(string key)`                                                | Return `true` if `key` exists in `_translations`, `false` otherwise.                                                              |
| `static IEnumerable<string> GetAllKeys()`                                       | Return all translation keys currently loaded.                                                                                     |
| `static Dictionary<string, string> GetAllWithPrefix(string prefix)`             | Return all `[key, value]` pairs where `key.StartsWith(prefix, StringComparison.OrdinalIgnoreCase)`.                               |
| `static string GetRandom(string baseKey, int count, params object[] args)`      | If `count > 0`, pick one of `"{baseKey}.1"`, `"{baseKey}.2"`, … `"{baseKey}.{count}"` at random; otherwise return `Get(baseKey)`. |
| `static string GetLocalizedText(string key)`                                    | Attempt lookup in current language; if missing, attempt fallback; if still missing, return `key`.                                 |

### Formatting Helpers (Culture‐Aware)

| Signature                                                                  | Description                                                                                                                                          |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `static string FormatDate(DateTime date, string format = null)`            | Format `date` according to current culture. If `format != null`, use that pattern; otherwise use culture’s default date/time format.                 |
| `static string FormatNumber(decimal number, string format = null)`         | Format `number` according to current culture. If `format != null`, use e.g. `"N2"`, `"P"`, etc.; otherwise use culture’s default numeric format.     |
| `static string FormatCurrency(decimal amount, string currencyCode = null)` | Format `amount` as currency. If `currencyCode == null`, use culture’s default symbol; otherwise override `NumberFormat.CurrencySymbol` to that code. |
| `static string GetRelativeTime(DateTime dateTime)`                         | Return a localized “x seconds/minutes/hours/days/months/years ago” or “in x …” string, depending on how far `dateTime` is from `DateTime.Now`.       |
| `static string FormatList(IEnumerable<string> items)`                      | Format a list of strings using `"list.two_items"`, `"list.separator"`, `"list.format"` keys—e.g. “a, b, and c” in English, “a, b et c” in French.    |

### Currency Parsing Helpers

| Signature                                                                                      | Description                                                                                                                |
| ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `static bool TryParsePrice(string priceString, out string currencySymbol, out decimal amount)` | Return `true` if `priceString` (“\$12.34”, “€1 234,56”) can be parsed into `currencySymbol` + `amount`; otherwise `false`. |
| `static string ExtractCurrencySymbol(string priceString)`                                      | Return everything before the first digit (trimmed)—likely the currency symbol; `null` if no digits.                        |
| `static decimal? ExtractPriceAmount(string priceString)`                                       | Return only the numeric portion after the currency symbol, parsed as `decimal`; `null` if parsing fails.                   |

### Language Metadata & Utilities

| Signature                                                | Description                                                                          |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `static string GetLanguageDisplayName(string langCode)`  | Return `CultureInfo.GetCultureInfo(langCode).EnglishName`, or `langCode` if invalid. |
| `static string GetLanguageNativeName(string langCode)`   | Return `CultureInfo.GetCultureInfo(langCode).NativeName`, or `langCode` if invalid.  |
| `static List<string> GetAvailableLanguageDisplayNames()` | Return display names for all codes from `GetAvailableLanguages()`.                   |
| `static List<string> GetAvailableLanguageNativeNames()`  | Return native names for all codes from `GetAvailableLanguages()`.                    |
| `static string GetSystemLanguageCode()`                  | Return `Application.systemLanguage.ToString()`.                                      |

### JSON Loading & Internals

| Signature                                                                | Description                                                                                                           |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| `static Dictionary<string,string> LoadTranslations(string languageCode)` | Private helper that loads and parses `Resources/Localization/{languageCode}.json` into a `Dictionary<string,string>`. |
| (inner) `private class LocalizationTable { Entry[] entries; … }`         | Matches JSON structure `{ "entries":[ { "key": ..., "value": ... }, … ] }` and converts to a dictionary.              |

---

With this documentation and the examples above, you should be able to:

1. **Populate**: Create JSON translation tables under `Resources/Localization/<lang>.json`.
2. **Initialize**: Call `LocalizationManager.Initialize("en")` at startup (or explicitly choose a language).
3. **Switch**: Use `LocalizationManager.ChangeLanguage("fr")` when the player picks French.
4. **Retrieve**: Call `LocalizationManager.Get("KEY")`, `GetPlural(…)`, or any of the other variants to fetch translated, formatted text.
5. **Format**: Use `FormatDate`/`FormatNumber`/`FormatCurrency`/`GetRelativeTime`/`FormatList` to present localized data in UI.
6. **Parse**: Use `TryParsePrice` or `ExtractCurrencySymbol` when dealing with user‐entered prices or similar strings.

Happy localizing!
