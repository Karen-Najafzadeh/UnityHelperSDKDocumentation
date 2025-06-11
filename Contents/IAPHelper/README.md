**IAPHelper Unity In-App Purchase Manager**

This document provides a detailed overview of the **IAPHelper** class, a singleton manager for handling Unity In-App Purchases (IAP). It covers setup, core methods, and real-world usage examples.

---

## Table of Contents

1. [Overview](#overview)
2. [Setup and Configuration](#setup-and-configuration)
3. [Singleton Pattern & Lifecycle](#singleton-pattern--lifecycle)
4. [Initialization](#initialization)
5. [Purchasing Methods](#purchasing-methods)

   * BuyProduct
   * RestorePurchases
6. [Product Information](#product-information)

   * GetFormattedPrice
   * GetProductInfo
   * IsProductOwned
7. [Purchase Event Handling](#purchase-event-handling)

   * OnPurchaseSucceeded & OnPurchaseFailedEvent
   * ProcessPurchase
8. [Purchase Tracking](#purchase-tracking)

   * TrackPurchaseRoutine & TrackPurchase
   * GetPurchaseHistory
   * GetTimeSinceLastPurchase
9. [Subscription Management](#subscription-management)
10. [Real-World Usage Examples](#real-world-usage-examples)
11. [Appendix: Supporting Classes](#appendix-supporting-classes)

---

## Overview

`IAPHelper` is a single-point manager for Unity IAP functionality. It:

* Defines your product catalog
* Initializes the purchasing subsystem
* Initiates purchases
* Handles success and failure callbacks
* Tracks receipts and purchase history
* Restores non-consumable products on iOS
* Provides localized pricing and subscription details

It implements `IDetailedStoreListener` (an extension of `IStoreListener`) to receive detailed callbacks from the Unity Purchasing system.

---

## Setup and Configuration

1. **Import Unity IAP**: In Unity Editor, go to **Window > Package Manager**, search for **In-App Purchasing** and install.
2. **Attach the Script**: Place `IAPHelper.cs` on a GameObject in your initial scene (e.g., `IAPManager`).
3. **Configure Products**:

   * In the Inspector, under **IAPHelper (Script)**, add entries to the **Products** list.
   * Each entry is an `IAPProduct` struct with fields:

     * `id` (string): Your store product ID.
     * `type` (ProductType): `Consumable`, `NonConsumable`, or `Subscription`.

---

## Singleton Pattern & Lifecycle

* **Instance**: `public static IAPHelper Instance` ensures one manager persists across scenes.
* **Awake()**:

  * Destroys duplicate instances.
  * Calls `DontDestroyOnLoad` to persist.
  * Initiates purchasing via `InitializePurchasing()`.

```csharp
void Awake() {
    if (Instance != null && Instance != this) {
        Destroy(gameObject);
        return;
    }
    Instance = this;
    DontDestroyOnLoad(gameObject);
    InitializePurchasing();
}
```

---

## Initialization

* **InitializePurchasing()**:

  * Checks `IsInitialized()` to avoid reinitialization.
  * Builds a `ConfigurationBuilder` with each `products` entry.
  * Calls `UnityPurchasing.Initialize(this, builder)`.

```csharp
public void InitializePurchasing() {
    if (IsInitialized()) return;
    var builder = ConfigurationBuilder.Instance(StandardPurchasingModule.Instance());
    foreach (var prod in products) {
        builder.AddProduct(prod.id, prod.type);
    }
    UnityPurchasing.Initialize(this, builder);
}
```

* **IsInitialized()**: Returns `true` when both `_storeController` and `_extensionProvider` are set after successful initialization.

---

## Purchasing Methods

### BuyProduct

Initiates a purchase by product ID.

```csharp
public void BuyProduct(string productId) {
    if (!IsInitialized()) { Debug.LogError("Not initialized."); return; }
    Product product = _storeController.products.WithID(productId);
    if (product != null && product.availableToPurchase) {
        _storeController.InitiatePurchase(product);
    } else {
        Debug.LogError($"Product '{productId}' not available.");
    }
}
```

**Example**:

```csharp
// Called when the user taps "Buy Gem Pack"
public void OnBuyGemsClicked() {
    IAPHelper.Instance.BuyProduct("com.mygame.gempack");
}
```

### RestorePurchases

Restores non-consumables/subscriptions on iOS/macOS.

```csharp
public void RestorePurchases() {
    if (!IsInitialized()) return;
    if (Application.platform == RuntimePlatform.IPhonePlayer ||
        Application.platform == RuntimePlatform.OSXPlayer) {
        var apple = _extensionProvider.GetExtension<IAppleExtensions>();
        apple.RestoreTransactions((success, message) => {
            Debug.Log($"Restore success: {success}, message: {message}");
        });
    }
}
```

---

## Product Information

### GetFormattedPrice

Returns localized price string via your `LocalizationManager`.

```csharp
public string GetFormattedPrice(string productId) {
    if (!IsInitialized()) return string.Empty;
    var prod = _storeController.products.WithID(productId);
    return prod != null && prod.availableToPurchase
        ? LocalizationManager.FormatCurrency((decimal)prod.metadata.localizedPrice, prod.metadata.isoCurrencyCode)
        : string.Empty;
}
```

### GetProductInfo

Returns `(title, description, price)` tuple.

```csharp
public (string title, string description, string price) GetProductInfo(string productId) {
    if (!IsInitialized()) return ("", "", "");
    var prod = _storeController.products.WithID(productId);
    return prod != null
        ? (prod.metadata.localizedTitle, prod.metadata.localizedDescription, GetFormattedPrice(productId))
        : ("", "", "");
}
```

### IsProductOwned

Checks if a non-consumable/subscription has been purchased (based on receipt presence).

```csharp
public bool IsProductOwned(string productId) {
    if (!IsInitialized()) return false;
    var prod = _storeController.products.WithID(productId);
    return prod != null && prod.hasReceipt;
}
```

---

## Purchase Event Handling

### Events

* **OnPurchaseSucceeded**: `Action<Product>` invoked on success.
* **OnPurchaseFailedEvent**: `Action<Product, PurchaseFailureReason>` invoked on failure.

### ProcessPurchase

Called by Unity IAP when a purchase completes.

```csharp
public PurchaseProcessingResult ProcessPurchase(PurchaseEventArgs args) {
    CoroutineHelper.StartManagedCoroutine($"track_{args.purchasedProduct.definition.id}", TrackPurchaseRoutine(args.purchasedProduct));
    OnPurchaseSucceeded?.Invoke(args.purchasedProduct);
    return PurchaseProcessingResult.Complete;
}
```

### OnPurchaseFailed

Called when purchase fails.

```csharp
public void OnPurchaseFailed(Product product, PurchaseFailureDescription desc) {
    _onPurchaseFailed?.Invoke(product, desc.reason);
}
```

---

## Purchase Tracking

### TrackPurchaseRoutine & TrackPurchase

Stores purchase info in `PrefsHelper` (local JSON) and updates `_purchaseCache`.

```csharp
private IEnumerator TrackPurchaseRoutine(Product product) {
    yield return null;
    await TrackPurchase(product);
}

private async Task TrackPurchase(Product product) {
    var info = new Dictionary<string, object> {
        ["productId"] = product.definition.id,
        ["purchaseDate"] = DateTime.UtcNow.ToString(PURCHASE_DATE_FORMAT),
        ["transactionId"] = product.transactionID,
        ["receipt"] = product.receipt
    };
    var history = PrefsHelper.GetJson<List<Dictionary<string, object>>, GamePrefs>(GamePrefs.ComplexData) ?? new List<...>();
    history.Add(info);
    await PrefsHelper.SetJson(GamePrefs.ComplexData, history);
    _purchaseCache[product.definition.id] = (DateTime.UtcNow, true);
}
```

### GetPurchaseHistory

Retrieves all purchases for a product.

```csharp
public List<(DateTime, string)> GetPurchaseHistory(string productId) {
    var history = PrefsHelper.GetJson<List<Dictionary<string, object>>, GamePrefs>(GamePrefs.ComplexData);
    return history?
        .Where(p => p["productId"].ToString() == productId)
        .Select(p => (DateTime.ParseExact(p["purchaseDate"].ToString(), PURCHASE_DATE_FORMAT, null), p["transactionId"].ToString()))
        .ToList()
      ?? new List<(DateTime, string)>();
}
```

### GetTimeSinceLastPurchase

Formats elapsed time using `LocalizationManager.GetRelativeTime`.

```csharp
public string GetTimeSinceLastPurchase(string productId) {
    var history = GetPurchaseHistory(productId);
    if (!history.Any()) return string.Empty;
    var last = history.OrderByDescending(h => h.Item1).First().Item1;
    return LocalizationManager.GetRelativeTime(last);
}
```

---

## Subscription Management

### GetSubscriptionInfo

Validates and parses subscription receipts via `SubscriptionManager`.

```csharp
public (bool isSubscribed, DateTime? expiryDate) GetSubscriptionInfo(string productId) {
    var prod = _storeController.products.WithID(productId);
    if (prod == null || !prod.hasReceipt || prod.definition.type != ProductType.Subscription)
        return (false, null);
    try {
        var mgr = new SubscriptionManager(prod, null);
        var info = mgr.getSubscriptionInfo();
        return (info.isSubscribed() == Result.True, info.getExpireDate());
    } catch (Exception e) {
        Debug.LogError($"Failed to parse subscription for {productId}: {e}");
        return (false, null);
    }
}
```

**Example**:

```csharp
var (subscribed, expiry) = IAPHelper.Instance.GetSubscriptionInfo("com.mygame.premium_monthly");
if (subscribed) {
    Debug.Log($"Premium active until: {expiry}");
}
```

---

## Real-World Usage Examples

1. **Displaying the Buy Button**:

   ```csharp
   void Start() {
       var price = IAPHelper.Instance.GetFormattedPrice("gem_pack_small");
       buyButtonText.text = price;
   }
   ```

2. **Listening for Purchase Events**:

   ```csharp
   void OnEnable() {
       IAPHelper.Instance.OnPurchaseSucceeded += OnSuccess;
       IAPHelper.Instance.OnPurchaseFailedEvent += OnFailure;
   }

   void OnDisable() {
       IAPHelper.Instance.OnPurchaseSucceeded -= OnSuccess;
       IAPHelper.Instance.OnPurchaseFailedEvent -= OnFailure;
   }

   void OnSuccess(Product p) {
       Debug.Log($"Purchased: {p.definition.id}");
       GrantContent(p.definition.id);
   }

   void OnFailure(Product p, PurchaseFailureReason r) {
       Debug.LogWarning($"Failed: {p.definition.id}, Reason: {r}");
   }
   ```

3. **Restoring Purchases on a "Restore" Button**:

   ```csharp
   public void OnRestoreClicked() {
       IAPHelper.Instance.RestorePurchases();
   }
   ```

4. **Showing Purchase History**:

   ```csharp
   var history = IAPHelper.Instance.GetPurchaseHistory("gem_pack_small");
   foreach (var (date, tx) in history) {
       Debug.Log($"{date}: {tx}");
   }
   ```

---

## Appendix: Supporting Classes

* **`PrefsHelper`**: Generic JSON storage helper.
* **`LocalizationManager`**: Formats currency and relative times.
* **`CoroutineHelper`**: Manages named coroutines.
* **Unity IAP Namespaces**:

  * `UnityEngine.Purchasing`
  * `UnityEngine.Purchasing.Extension`
  * `UnityEngine.Purchasing.Security`

This completes the detailed documentation for `IAPHelper`. Integrate and adapt these patterns to suit your project’s structure and flow.
