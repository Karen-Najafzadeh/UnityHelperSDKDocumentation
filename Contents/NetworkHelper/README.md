Below is a comprehensive reference for **NetworkRequestManager**, a static utility built on Unity’s `UnityWebRequest` API. It centralizes common HTTP workflows—JSON GET/POST/PUT/DELETE, retries with exponential backoff, timeouts, custom headers, auth tokens, file download/upload, request throttling, and convenience methods for raw string, bytes, and multimedia fetches.

---

## Table of Contents

1. [Overview](#overview)
2. [Configuration & Throttling](#configuration--throttling)
3. [Default Headers & Authentication](#default-headers--authentication)
4. [JSON Requests](#json-requests)

   * [GetJsonAsync](#getjsonasync)
   * [PostJsonAsync](#postjsonasync)
   * [PutJsonAsync](#putjsonasync)
   * [DeleteJsonAsync](#deletejsonasync)
   * [PatchJsonAsync](#patchjsonasync)
   * [SendJsonRequestAsync (private)](#sendjsonrequestasync-private)
   * [ComputeBackoff (private)](#computebackoff-private)
5. [Raw HTTP Requests](#raw-http-requests)

   * [GetStringAsync](#getstringasync)
   * [GetBytesAsync](#getbytesasync)
   * [SendRequestAsync](#sendrequestasync)
   * [HeadAsync](#headasync)
   * [TryGetJsonAsync](#trygetjsonasync)
6. [File Download & Upload](#file-download--upload)

   * [DownloadFileAsync](#downloadfileasync)
   * [UploadFileAsync](#uploadfileasync)
7. [Multimedia Downloads](#multimedia-downloads)

   * [DownloadTextureAsync](#downloadtextureasync)
   * [DownloadAudioClipAsync](#downloadaudioclipasync)
8. [Utility Methods](#utility-methods)

   * [PingAsync](#pingasync)
   * [SetDefaultTimeout](#setdefaulttimeout)
   * [SetMaxRetries](#setmaxretries)
   * [ClearDefaultHeaders](#cleardefaultheaders)
   * [IsInternetAvailable](#isinternetavailable)
9. [Method Summary Table](#method-summary-table)

---

## 1. Overview

`NetworkRequestManager` is designed as a single, static “HTTP client” tailored for Unity. It:

* Uses `UnityWebRequest` under the hood.
* Offers JSON conveniences (`GetJsonAsync<T>`, `PostJsonAsync<T>`, etc.) that automatically serialize/deserialize with `JsonUtility`.
* Retries transient failures up to a configurable `_maxRetries`, with exponential backoff (`ComputeBackoff`).
* Enforces a per‐request timeout (`_defaultTimeout`), after which it throws a `TimeoutException`.
* Limits concurrent HTTP requests to `_maxConcurrentRequests` using a `SemaphoreSlim`.
* Allows setting global default headers and an optional `AuthToken`—automatically appended to each request’s `Authorization: Bearer <token>` header.
* Supports raw string and byte downloads, file download/upload, HTTP HEAD, PATCH, and generic request (`SendRequestAsync`).
* Provides utility methods for “Ping” (HEAD to test reachability), checking `Application.internetReachability`, and adjusting timeout/retry parameters at runtime.

All methods return `Task` or `Task<T>`, making them `async`‐friendly in C# (Unity 2020+/2018+ with .NET 4.x support).

---

## 2. Configuration & Throttling

```csharp
// Maximum number of automatic retry attempts
private static int _maxRetries = 3;

// Per‐request timeout duration (default: 10 seconds)
private static TimeSpan _defaultTimeout = TimeSpan.FromSeconds(10);

// Maximum number of concurrent requests allowed
private static int _maxConcurrentRequests = 4;

// Semaphore to throttle simultaneous requests
private static readonly SemaphoreSlim _throttle =
    new SemaphoreSlim(_maxConcurrentRequests, _maxConcurrentRequests);
```

* **\_maxRetries**: May be changed at runtime via `SetMaxRetries(int)`.
* **\_defaultTimeout**: Can be changed via `SetDefaultTimeout(TimeSpan)`.
* **\_maxConcurrentRequests**: If you need to raise or lower concurrency, adjust `_maxConcurrentRequests` before any requests are queued (or reinitialize the semaphore manually).
* **\_throttle**: Ensures that no more than `_maxConcurrentRequests` calls are actively sending. Any extra callers will await `_throttle.WaitAsync()` until a slot frees.

---

## 3. Default Headers & Authentication

```csharp
// JWT or other auth token—if set, each request includes "Authorization: Bearer <AuthToken>"
public static string AuthToken { get; set; }

// Global headers included in every request
private static readonly Dictionary<string, string> _defaultHeaders = new Dictionary<string, string>();

/// <summary>Add or update a header that will be included in every request.</summary>
public static void SetDefaultHeader(string key, string value)
{
    _defaultHeaders[key] = value;
}

/// <summary>Remove a default header.</summary>
public static void RemoveDefaultHeader(string key)
{
    _defaultHeaders.Remove(key);
}

/// <summary>Remove all default headers at once.</summary>
public static void ClearDefaultHeaders()
{
    _defaultHeaders.Clear();
}
```

* Before making any calls, you can do:

  ```csharp
  NetworkRequestManager.AuthToken = userJwtToken;
  NetworkRequestManager.SetDefaultHeader("X-App-Version", "1.2.3");
  ```
* When sending a request, every header in `_defaultHeaders` is appended. If `AuthToken` is non‐empty, an `Authorization` header is added automatically.

---

## 4. JSON Requests

### GetJsonAsync

```csharp
/// <summary>
/// Send a GET request and parse the JSON response into type T using JsonUtility.
/// </summary>
/// <typeparam name="T">Type to deserialize into (must be serializable by JsonUtility).</typeparam>
/// <param name="url">Request URL.</param>
/// <param name="headers">Optional per‐call headers.</param>
/// <param name="cancellationToken">Optional cancellation token.</param>
/// <returns>Deserialized T on success; throws on failure.</returns>
public static async Task<T> GetJsonAsync<T>(
    string url,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    return await SendJsonRequestAsync<T>(UnityWebRequest.kHttpVerbGET, url, null, headers, cancellationToken);
}
```

* Invokes `SendJsonRequestAsync<T>("GET", url, null, headers, ct)`.
* On success, returns `T` after `JsonUtility.FromJson<T>(responseText)`.
* On HTTP error or timeout, throws an exception (which may be retried internally up to `_maxRetries`).

---

### PostJsonAsync

```csharp
/// <summary>
/// Send a POST request with a JSON‐serialized payload, returning T from the JSON response.
/// </summary>
/// <typeparam name="T">Type to deserialize response into.</typeparam>
/// <param name="url">Endpoint URL.</param>
/// <param name="payload">Object to serialize as JSON body.</param>
/// <param name="headers">Optional per‐call headers.</param>
/// <param name="cancellationToken">Optional cancellation token.</param>
/// <returns>Deserialized T on success.</returns>
public static async Task<T> PostJsonAsync<T>(
    string url,
    object payload,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    string body = JsonUtility.ToJson(payload);
    return await SendJsonRequestAsync<T>(UnityWebRequest.kHttpVerbPOST, url, body, headers, cancellationToken);
}
```

* Serializes `payload` with `JsonUtility.ToJson`.
* Invokes `SendJsonRequestAsync<T>("POST", url, jsonBody, headers, ct)`.

---

### PutJsonAsync

```csharp
/// <summary>
/// Send a PUT request with JSON payload, returning T from the JSON response.
/// </summary>
public static async Task<T> PutJsonAsync<T>(
    string url,
    object payload,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    string body = JsonUtility.ToJson(payload);
    return await SendJsonRequestAsync<T>(UnityWebRequest.kHttpVerbPUT, url, body, headers, cancellationToken);
}
```

* Same pattern as `PostJsonAsync`, but uses HTTP “PUT”.

---

### DeleteJsonAsync

```csharp
/// <summary>
/// Send a DELETE request, parsing the JSON response into T.
/// </summary>
public static async Task<T> DeleteJsonAsync<T>(
    string url,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    return await SendJsonRequestAsync<T>(UnityWebRequest.kHttpVerbDELETE, url, null, headers, cancellationToken);
}
```

* Uses HTTP “DELETE” with no request body.

---

### PatchJsonAsync

```csharp
/// <summary>
/// Send a PATCH request with JSON payload; parse response JSON into T.
/// </summary>
public static async Task<T> PatchJsonAsync<T>(
    string url,
    object payload,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    string body = JsonUtility.ToJson(payload);
    // Note: UnityWebRequest doesn’t have a built‐in kHttpVerbPATCH, so we pass "PATCH" explicitly.
    return await SendJsonRequestAsync<T>("PATCH", url, body, headers, cancellationToken);
}
```

* Exactly like POST/PUT but uses method="PATCH".

---

#### SendJsonRequestAsync (private)

```csharp
/// <summary>
/// Internal helper that executes a JSON request with retries, timeout, and deserialization.
/// </summary>
/// <typeparam name="T">Type to deserialize response into.</typeparam>
/// <param name="method">HTTP verb (GET/POST/PUT/DELETE/PATCH).</param>
/// <param name="url">Target URL.</param>
/// <param name="jsonBody">Request body (JSON‐encoded string) or null if none.</param>
/// <param name="headers">Optional per‐call headers.</param>
/// <param name="cancellationToken">Optional cancellation token.</param>
/// <returns>Deserialized object of type T.</returns>
private static async Task<T> SendJsonRequestAsync<T>(
    string method,
    string url,
    string jsonBody,
    Dictionary<string, string> headers,
    CancellationToken cancellationToken)
{
    int attempt = 0;
    Exception lastError = null;

    while (attempt < _maxRetries)
    {
        attempt++;
        await _throttle.WaitAsync(cancellationToken);
        using (var req = new UnityWebRequest(url, method))
        {
            try
            {
                // If there is a JSON body, assign uploadHandler
                if (!string.IsNullOrEmpty(jsonBody))
                {
                    byte[] bodyRaw = System.Text.Encoding.UTF8.GetBytes(jsonBody);
                    req.uploadHandler = new UploadHandlerRaw(bodyRaw);
                }

                // Always include a DownloadHandlerBuffer
                req.downloadHandler = new DownloadHandlerBuffer();
                req.SetRequestHeader("Content-Type", "application/json");

                // If an AuthToken exists, add Bearer header
                if (!string.IsNullOrEmpty(AuthToken))
                    req.SetRequestHeader("Authorization", $"Bearer {AuthToken}");

                // Default headers (global)
                foreach (var kv in _defaultHeaders)
                    req.SetRequestHeader(kv.Key, kv.Value);

                // Per‐call custom headers
                if (headers != null)
                    foreach (var kv in headers)
                        req.SetRequestHeader(kv.Key, kv.Value);

                // Send request and await completion or timeout
                var op = req.SendWebRequest();
                using (var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken))
                {
                    timeoutCts.CancelAfter(_defaultTimeout);
                    while (!op.isDone && !timeoutCts.IsCancellationRequested)
                        await Task.Yield();

                    if (timeoutCts.IsCancellationRequested)
                        throw new TimeoutException($"Request to {url} timed out after {_defaultTimeout.TotalSeconds}s");
                }

                // Check for HTTP or network errors
                if (req.result != UnityWebRequest.Result.Success)
                    throw new Exception($"HTTP {req.responseCode}: {req.error}\n{req.downloadHandler.text}");

                // Successful response—deserialize JSON
                string responseText = req.downloadHandler.text;
                return JsonUtility.FromJson<T>(responseText);
            }
            catch (Exception ex) when (attempt < _maxRetries)
            {
                lastError = ex;
                // Exponential backoff before retrying
                await Task.Delay(ComputeBackoff(attempt), cancellationToken);
            }
            finally
            {
                _throttle.Release();
            }
        }
    }

    // If we exit loop, all retries failed—throw final exception
    throw new Exception($"Failed after {_maxRetries} attempts", lastError);
}
```

* **Retry Loop**: up to `_maxRetries` attempts.
* **Throttling**: Each attempt first awaits `_throttle.WaitAsync()`, limiting concurrency. Release the slot in `finally`.
* **Body & Headers**: If `jsonBody` is non‐null, an `UploadHandlerRaw` is assigned. Then “Content-Type: application/json”, “Authorization: Bearer …” (if `AuthToken` is set), all `_defaultHeaders`, then per‐call `headers`.
* **Timeout Handling**: Creates a linked `CancellationTokenSource` that cancels after `_defaultTimeout`. If canceled, throws `TimeoutException`.
* **Error Check**: If `req.result != Success`, throws an exception containing HTTP status and response text—caught by the `catch` and triggers a backoff.
* **Success**: Calls `JsonUtility.FromJson<T>(responseText)` and returns.

---

#### ComputeBackoff (private)

```csharp
/// <summary>
/// Compute an exponential backoff delay for retry attempts.
/// Example: 500ms × 2^(attempt - 1).
/// </summary>
/// <param name="attempt">1‐based attempt number.</param>
/// <returns>A TimeSpan to await before the next try.</returns>
private static TimeSpan ComputeBackoff(int attempt)
{
    return TimeSpan.FromMilliseconds(500 * Math.Pow(2, attempt - 1));
}
```

* **Attempt 1** → 500ms
* **Attempt 2** → 1000ms
* **Attempt 3** → 2000ms
* …and so on, doubling each time.

---

## 5. Raw HTTP Requests

Sometimes you need the raw response (string or bytes), or wish to send arbitrary HTTP verbs without JSON deserialization.

### GetStringAsync

```csharp
/// <summary>
/// Send a GET request and return the raw response body as a string.
/// Throws on HTTP errors or timeouts.
/// </summary>
public static async Task<string> GetStringAsync(
    string url,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    await _throttle.WaitAsync(cancellationToken);
    using (var req = UnityWebRequest.Get(url))
    {
        if (!string.IsNullOrEmpty(AuthToken))
            req.SetRequestHeader("Authorization", $"Bearer {AuthToken}");

        foreach (var kv in _defaultHeaders)
            req.SetRequestHeader(kv.Key, kv.Value);

        if (headers != null)
            foreach (var kv in headers)
                req.SetRequestHeader(kv.Key, kv.Value);

        var op = req.SendWebRequest();
        while (!op.isDone && !cancellationToken.IsCancellationRequested)
            await Task.Yield();

        _throttle.Release();

        if (req.result != UnityWebRequest.Result.Success)
            throw new Exception($"HTTP {req.responseCode}: {req.error}");

        return req.downloadHandler.text;
    }
}
```

**Usage**: Retrieving raw JSON (without deserialization), HTML, plain‐text endpoints, or any other string‐based response.

---

### GetBytesAsync

```csharp
/// <summary>
/// Send a GET request and return the raw response body as a byte array.
/// Useful for binary endpoints (images, files, etc.).
/// </summary>
public static async Task<byte[]> GetBytesAsync(
    string url,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    await _throttle.WaitAsync(cancellationToken);
    using (var req = UnityWebRequest.Get(url))
    {
        if (!string.IsNullOrEmpty(AuthToken))
            req.SetRequestHeader("Authorization", $"Bearer {AuthToken}");

        foreach (var kv in _defaultHeaders)
            req.SetRequestHeader(kv.Key, kv.Value);

        if (headers != null)
            foreach (var kv in headers)
                req.SetRequestHeader(kv.Key, kv.Value);

        var op = req.SendWebRequest();
        while (!op.isDone && !cancellationToken.IsCancellationRequested)
            await Task.Yield();

        _throttle.Release();

        if (req.result != UnityWebRequest.Result.Success)
            throw new Exception($"HTTP {req.responseCode}: {req.error}");

        return req.downloadHandler.data;
    }
}
```

**Usage**: Fetching arbitrary binary data (e.g., small files) without relying on `DownloadHandlerFile`.

---

### SendRequestAsync

```csharp
/// <summary>
/// Send a custom HTTP request with arbitrary method, body, headers, and content type.
/// Returns the raw response body as a string. Throws on errors/timeouts.
/// </summary>
public static async Task<string> SendRequestAsync(
    string method,
    string url,
    string body = null,
    Dictionary<string, string> headers = null,
    string contentType = "application/json",
    CancellationToken cancellationToken = default)
{
    await _throttle.WaitAsync(cancellationToken);
    using (var req = new UnityWebRequest(url, method))
    {
        if (!string.IsNullOrEmpty(body))
        {
            byte[] bodyRaw = System.Text.Encoding.UTF8.GetBytes(body);
            req.uploadHandler = new UploadHandlerRaw(bodyRaw);
        }

        req.downloadHandler = new DownloadHandlerBuffer();
        req.SetRequestHeader("Content-Type", contentType);

        if (!string.IsNullOrEmpty(AuthToken))
            req.SetRequestHeader("Authorization", $"Bearer {AuthToken}");

        foreach (var kv in _defaultHeaders)
            req.SetRequestHeader(kv.Key, kv.Value);

        if (headers != null)
            foreach (var kv in headers)
                req.SetRequestHeader(kv.Key, kv.Value);

        var op = req.SendWebRequest();
        while (!op.isDone && !cancellationToken.IsCancellationRequested)
            await Task.Yield();

        _throttle.Release();

        if (req.result != UnityWebRequest.Result.Success)
            throw new Exception($"HTTP {req.responseCode}: {req.error}");

        return req.downloadHandler.text;
    }
}
```

* You choose `method = "GET"`, `"POST"`, `"PUT"`, `"DELETE"`, `"PATCH"`, or even `"OPTIONS"` or any custom verb.
* If `body` is non‐null, it’s sent as raw bytes with `UploadHandlerRaw`.
* `contentType` defaults to `"application/json"`, but you can override (e.g., `"text/plain"`, `"application/xml"`).

---

### HeadAsync

```csharp
/// <summary>
/// Send an HTTP HEAD request to retrieve only headers (no body). 
/// Returns the UnityWebRequest so you can inspect `responseCode`, `GetResponseHeader(...)`, etc.
/// </summary>
public static async Task<UnityWebRequest> HeadAsync(
    string url,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    await _throttle.WaitAsync(cancellationToken);
    using (var req = new UnityWebRequest(url, UnityWebRequest.kHttpVerbHEAD))
    {
        req.downloadHandler = new DownloadHandlerBuffer();

        if (!string.IsNullOrEmpty(AuthToken))
            req.SetRequestHeader("Authorization", $"Bearer {AuthToken}");

        foreach (var kv in _defaultHeaders)
            req.SetRequestHeader(kv.Key, kv.Value);

        if (headers != null)
            foreach (var kv in headers)
                req.SetRequestHeader(kv.Key, kv.Value);

        var op = req.SendWebRequest();
        while (!op.isDone && !cancellationToken.IsCancellationRequested)
            await Task.Yield();

        _throttle.Release();
        return req; // Caller checks req.result and req.responseCode
    }
}
```

**Usage**: Verify resource existence, fetch metadata (like `Content-Length`) without downloading the body.

---

### TryGetJsonAsync

```csharp
/// <summary>
/// Attempt a GET + JSON parse; returns a tuple (Success, Result). 
/// If any exception occurs, returns (false, default(T)) instead of throwing.
/// </summary>
public static async Task<(bool Success, T Result)> TryGetJsonAsync<T>(
    string url,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    try
    {
        var result = await GetJsonAsync<T>(url, headers, cancellationToken);
        return (true, result);
    }
    catch
    {
        return (false, default);
    }
}
```

---

## 6. File Download & Upload

### DownloadFileAsync

```csharp
/// <summary>
/// Download data from `url` and save directly to `localPath` (file system).
/// Reports progress via `IProgress<float>`. Throws on failure.
/// </summary>
public static async Task DownloadFileAsync(
    string url,
    string localPath,
    IProgress<float> progress = null,
    CancellationToken cancellationToken = default)
{
    await _throttle.WaitAsync(cancellationToken);
    using (var req = UnityWebRequest.Get(url))
    {
        // Instruct the download handler to write directly to file
        req.downloadHandler = new DownloadHandlerFile(localPath);

        var op = req.SendWebRequest();
        while (!op.isDone && !cancellationToken.IsCancellationRequested)
        {
            progress?.Report(op.progress); // 0.0 to 1.0
            await Task.Yield();
        }
        _throttle.Release();

        if (req.result != UnityWebRequest.Result.Success)
            throw new Exception($"Download error: {req.error}");
    }
}
```

* **Usage**: Large file downloads (assets, binaries) that shouldn’t be buffered fully in memory.
* The `progress` callback allows UI progress bars.

---

### UploadFileAsync

```csharp
/// <summary>
/// Upload a file (in memory as byte[]) via multipart/form-data to `url`. 
/// Returns the server’s raw string response.
/// </summary>
public static async Task<string> UploadFileAsync(
    string url,
    byte[] fileData,
    string fileName,
    string formField = "file",
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    await _throttle.WaitAsync(cancellationToken);

    // Build a multipart form containing the file
    var form = new List<IMultipartFormSection>
    {
        new MultipartFormFileSection(formField, fileData, fileName, "application/octet-stream")
    };

    using (var req = UnityWebRequest.Post(url, form))
    {
        if (!string.IsNullOrEmpty(AuthToken))
            req.SetRequestHeader("Authorization", $"Bearer {AuthToken}");

        if (headers != null)
            foreach (var kv in headers)
                req.SetRequestHeader(kv.Key, kv.Value);

        var op = req.SendWebRequest();
        while (!op.isDone && !cancellationToken.IsCancellationRequested)
            await Task.Yield();

        _throttle.Release();

        if (req.result != UnityWebRequest.Result.Success)
            throw new Exception($"Upload error: {req.error}");

        return req.downloadHandler.text;
    }
}
```

* **Usage**: Send images, documents, or other binary data via a standard `multipart/form-data` POST.
* The server must expect a form‐field named `formField` (default `"file"`).

---

## 7. Multimedia Downloads

### DownloadTextureAsync

```csharp
/// <summary>
/// Download an image from `url` and return it as a Texture2D. 
/// Throws on error or timeout.
/// </summary>
public static async Task<Texture2D> DownloadTextureAsync(
    string url,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    await _throttle.WaitAsync(cancellationToken);
    using (var req = UnityWebRequestTexture.GetTexture(url))
    {
        if (!string.IsNullOrEmpty(AuthToken))
            req.SetRequestHeader("Authorization", $"Bearer {AuthToken}");

        foreach (var kv in _defaultHeaders)
            req.SetRequestHeader(kv.Key, kv.Value);

        if (headers != null)
            foreach (var kv in headers)
                req.SetRequestHeader(kv.Key, kv.Value);

        var op = req.SendWebRequest();
        while (!op.isDone && !cancellationToken.IsCancellationRequested)
            await Task.Yield();

        _throttle.Release();

        if (req.result != UnityWebRequest.Result.Success)
            throw new Exception($"Texture download error: {req.error}");

        return DownloadHandlerTexture.GetContent(req);
    }
}
```

* **Usage**: Dynamically fetch avatar images, UI textures, or remote sprites at runtime.

---

### DownloadAudioClipAsync

```csharp
/// <summary>
/// Download an audio clip (e.g. .wav, .mp3) from `url` by specifying the AudioType. 
/// Returns an AudioClip ready for playback (or null if failure).
/// </summary>
public static async Task<AudioClip> DownloadAudioClipAsync(
    string url,
    AudioType audioType,
    Dictionary<string, string> headers = null,
    CancellationToken cancellationToken = default)
{
    await _throttle.WaitAsync(cancellationToken);
    using (var req = UnityWebRequestMultimedia.GetAudioClip(url, audioType))
    {
        if (!string.IsNullOrEmpty(AuthToken))
            req.SetRequestHeader("Authorization", $"Bearer {AuthToken}");

        foreach (var kv in _defaultHeaders)
            req.SetRequestHeader(kv.Key, kv.Value);

        if (headers != null)
            foreach (var kv in headers)
                req.SetRequestHeader(kv.Key, kv.Value);

        var op = req.SendWebRequest();
        while (!op.isDone && !cancellationToken.IsCancellationRequested)
            await Task.Yield();

        _throttle.Release();

        if (req.result != UnityWebRequest.Result.Success)
            throw new Exception($"Audio download error: {req.error}");

        return DownloadHandlerAudioClip.GetContent(req);
    }
}
```

* **Usage**: Streaming background music or sound effects from a remote server at runtime.

---

## 8. Utility Methods

### PingAsync

```csharp
/// <summary>
/// Send a HEAD request to `url` to check if the server is reachable.
/// Returns true if the response code is in [200..399].
/// </summary>
public static async Task<bool> PingAsync(
    string url,
    int timeoutMs = 2000,
    CancellationToken cancellationToken = default)
{
    try
    {
        await _throttle.WaitAsync(cancellationToken);
        using (var req = new UnityWebRequest(url, UnityWebRequest.kHttpVerbHEAD))
        {
            req.downloadHandler = new DownloadHandlerBuffer();
            req.timeout = timeoutMs / 1000;

            var op = req.SendWebRequest();
            while (!op.isDone && !cancellationToken.IsCancellationRequested)
                await Task.Yield();

            _throttle.Release();

            return req.result == UnityWebRequest.Result.Success && 
                   req.responseCode >= 200 && req.responseCode < 400;
        }
    }
    catch
    {
        return false;
    }
}
```

* **Usage**: Quickly verify that a server or endpoint is up before issuing heavier requests.

---

### SetDefaultTimeout

```csharp
/// <summary>
/// Change the per‐request timeout for subsequent calls.
/// </summary>
public static void SetDefaultTimeout(TimeSpan timeout)
{
    _defaultTimeout = timeout;
}
```

---

### SetMaxRetries

```csharp
/// <summary>
/// Change the maximum number of retry attempts for JSON requests.
/// </summary>
public static void SetMaxRetries(int retries)
{
    _maxRetries = retries;
}
```

---

### ClearDefaultHeaders

```csharp
/// <summary>
/// Remove all headers that were registered via SetDefaultHeader.
/// </summary>
public static void ClearDefaultHeaders()
{
    _defaultHeaders.Clear();
}
```

---

### IsInternetAvailable

```csharp
/// <summary>
/// Returns whether the device currently has any internet connectivity
/// (Unity’s Application.internetReachability != NotReachable).
/// </summary>
public static bool IsInternetAvailable()
{
    return Application.internetReachability != NetworkReachability.NotReachable;
}
```

* Equivalent to checking Unity’s built‐in reachability enum.

---

## 9. Method Summary Table

| Method                             | Signature                                                                                                                                                                                         | Description                                                                                                           |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **SetDefaultHeader**               | `void SetDefaultHeader(string key, string value)`                                                                                                                                                 | Add or update a key/value that will be included in every outgoing request.                                            |
| **RemoveDefaultHeader**            | `void RemoveDefaultHeader(string key)`                                                                                                                                                            | Remove a previously added default header.                                                                             |
| **ClearDefaultHeaders**            | `void ClearDefaultHeaders()`                                                                                                                                                                      | Clear all global default headers.                                                                                     |
| **AuthToken (property)**           | `public static string AuthToken { get; set; }`                                                                                                                                                    | JWT or other token appended as `Authorization: Bearer <AuthToken>`.                                                   |
| **GetJsonAsync**                   | `Task<T> GetJsonAsync<T>(string url, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                                                   | GET JSON from `url` and deserialize into `T`. Retries up to `_maxRetries`, applies `_defaultTimeout` and `_throttle`. |
| **PostJsonAsync**                  | `Task<T> PostJsonAsync<T>(string url, object payload, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                                  | Serialize `payload` to JSON, send POST, then parse the JSON response into `T`.                                        |
| **PutJsonAsync**                   | `Task<T> PutJsonAsync<T>(string url, object payload, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                                   | Serialize `payload` to JSON, send PUT, then parse JSON response into `T`.                                             |
| **DeleteJsonAsync**                | `Task<T> DeleteJsonAsync<T>(string url, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                                                | Send DELETE, parse JSON response into `T`.                                                                            |
| **PatchJsonAsync**                 | `Task<T> PatchJsonAsync<T>(string url, object payload, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                                 | Send PATCH with JSON payload, parse JSON response into `T`.                                                           |
| **GetStringAsync**                 | `Task<string> GetStringAsync(string url, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                                               | GET raw response as a string.                                                                                         |
| **GetBytesAsync**                  | `Task<byte[]> GetBytesAsync(string url, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                                                | GET raw response as a byte array.                                                                                     |
| **SendRequestAsync**               | `Task<string> SendRequestAsync(string method, string url, string body = null, Dictionary<string,string> headers = null, string contentType = "application/json", CancellationToken ct = default)` | General‐purpose HTTP request with any `method` and optional `body`. Returns raw string response.                      |
| **HeadAsync**                      | `Task<UnityWebRequest> HeadAsync(string url, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                                           | Send HEAD request, returning the raw `UnityWebRequest` so caller can inspect `responseCode` and headers.              |
| **TryGetJsonAsync**                | `Task<(bool Success, T Result)> TryGetJsonAsync<T>(string url, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                         | Attempts `GetJsonAsync<T>`—returns `(false, default(T))` on any exception, instead of throwing.                       |
| **DownloadFileAsync**              | `Task DownloadFileAsync(string url, string localPath, IProgress<float> progress = null, CancellationToken ct = default)`                                                                          | Download file binary data from `url` directly into `localPath` on disk. Reports `progress` (0…1).                     |
| **UploadFileAsync**                | `Task<string> UploadFileAsync(string url, byte[] fileData, string fileName, string formField = "file", Dictionary<string,string> headers = null, CancellationToken ct = default)`                 | Upload a file via `multipart/form-data` POST, return raw server response text.                                        |
| **DownloadTextureAsync**           | `Task<Texture2D> DownloadTextureAsync(string url, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                                                      | Download an image from `url` as a `Texture2D`.                                                                        |
| **DownloadAudioClipAsync**         | `Task<AudioClip> DownloadAudioClipAsync(string url, AudioType audioType, Dictionary<string,string> headers = null, CancellationToken ct = default)`                                               | Download audio from `url` (specifying `AudioType`) and return as `AudioClip`.                                         |
| **PingAsync**                      | `Task<bool> PingAsync(string url, int timeoutMs = 2000, CancellationToken ct = default)`                                                                                                          | HEAD request to check if `url` is reachable (returns true if HTTP 2xx–3xx).                                           |
| **SetDefaultTimeout**              | `void SetDefaultTimeout(TimeSpan timeout)`                                                                                                                                                        | Change the default request timeout for all subsequent calls.                                                          |
| **SetMaxRetries**                  | `void SetMaxRetries(int retries)`                                                                                                                                                                 | Change how many times the code will retry a failed JSON request before giving up.                                     |
| **IsInternetAvailable**            | `bool IsInternetAvailable()`                                                                                                                                                                      | Returns `true` if `Application.internetReachability != NotReachable`.                                                 |
| **ComputeBackoff (private)**       | `TimeSpan ComputeBackoff(int attempt)`                                                                                                                                                            | Returns an exponential backoff delay `500ms × 2^(attempt-1)`.                                                         |
| **SendJsonRequestAsync (private)** | `Task<T> SendJsonRequestAsync<T>(string method, string url, string jsonBody, Dictionary<string,string> headers, CancellationToken ct)`                                                            | Core engine for GET/POST/PUT/DELETE/PATCH JSON requests with retries, timeout, and `_throttle`.                       |

---

### Method Use Cases

* **JSON endpoints**

  * GET/POST/PUT/DELETE/PATCH JSON:

    * `GetJsonAsync<T>("/api/user/123")`
    * `PostJsonAsync<MyDto>("/api/items", newItem)`

* **Raw string/bytes**

  * `GetStringAsync("https://example.com/data.txt")`
  * `GetBytesAsync("https://example.com/archive.zip")`

* **File streaming**

  * `await DownloadFileAsync(url, Application.persistentDataPath + "/updates.zip", progress)`
  * `await UploadFileAsync(uploadUrl, fileBytes, "avatar.png");`

* **Texture & Audio**

  * `Texture2D tex = await DownloadTextureAsync(imageUrl);`
  * `AudioClip clip = await DownloadAudioClipAsync(audioUrl, AudioType.WAV);`

* **Concurrency & timeouts**

  * Automatically restricted to `_maxConcurrentRequests` at once.
  * Each request times out after `_defaultTimeout` if no response.
  * Retries on failure up to `_maxRetries` with exponential backoff.

* **Utilities**

  * `bool alive = await PingAsync("https://api.example.com/health");`
  * `NetworkRequestManager.IsInternetAvailable()` to check simple offline/online.

---

## Usage Example

Below is a simple usage scenario: imagine you need to fetch a list of items from a JSON API, then download an image for each:

```csharp
public async void FetchItemsAndImages()
{
    // 1) Ensure we have an auth token set (if required by your API)
    NetworkRequestManager.AuthToken = PlayerPrefs.GetString("jwt_token");

    // 2) Add a default header
    NetworkRequestManager.SetDefaultHeader("X-App-Version", Application.version);

    try
    {
        // 3) Fetch a JSON list of items
        var (success, items) = await NetworkRequestManager.TryGetJsonAsync<List<ItemDto>>(
            "https://api.example.com/items");

        if (!success)
        {
            Debug.LogWarning("Failed to fetch items.");
            return;
        }

        // 4) For each item, download its image and assign to a UI element
        foreach (var item in items)
        {
            try
            {
                Texture2D tex = await NetworkRequestManager.DownloadTextureAsync(item.ImageUrl);
                // Suppose you have a method to create an in‐scene sprite from Texture2D:
                UIManager.Instance.CreateItemIcon(item.Id, tex);
            }
            catch (Exception texEx)
            {
                Debug.LogError($"Error downloading image for item {item.Id}: {texEx.Message}");
            }
        }
    }
    catch (Exception ex)
    {
        Debug.LogError($"Unexpected error in FetchItemsAndImages: {ex.Message}");
    }
    finally
    {
        // 5) Clean up any default headers if you no longer need them globally
        NetworkRequestManager.RemoveDefaultHeader("X-App-Version");
    }
}
```

* **Flow**:

  1. Configure `AuthToken` and default headers.
  2. Call `TryGetJsonAsync<List<ItemDto>>` to fetch an array of items.
  3. For each item, call `DownloadTextureAsync` to load its image.
  4. Each call respects `_maxConcurrentRequests = 4` (by default), so at most four simultaneous downloads run.
  5. Any failed image download is caught and logged, without crashing the overall workflow.

---

## Conclusion

`NetworkRequestManager` provides a robust, single‐point HTTP client tailored for Unity:

* **Retries, Timeouts, Backoff**: Resilient to intermittent network failures.
* **Throttling**: Avoids flooding mobile or web platforms with too many concurrent requests.
* **JSON Helpers**: Simplify serialization/deserialization using Unity’s built‐in `JsonUtility`.
* **File & Multimedia**: Built‐in support for file‐to‐disk, texture, and audio downloads.
* **Configuration**: Runtime control over timeout, retries, default headers, and auth tokens.

Use the table above as quick reference, and the example to see how it fits into a real project. Adapt `_maxConcurrentRequests`, `_defaultTimeout`, and `_maxRetries` to suit your game’s performance/bandwidth profile.
