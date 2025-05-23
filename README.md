# UnityHelperSDKDocumentation
A brief summary of how the UnityHeleprSDK works, with examples and usecases.


# UnityHelperToolkit

A comprehensive collection of helper classes to accelerate Unity C# project development. Each helper focuses on a specific domain—UI, time, scenes, serialization, networking, and more—providing clean, reusable functionality.

---

## 🚀 Getting Started

1. **Clone or download** this repository into your Unity project's `Assets/` folder.
2. **Install dependencies** (e.g., Newtonsoft.Json, Firebase SDK) via the Unity Package Manager or manually.
3. **Navigate** to `Assets/UnityHelperToolkit/Readme` for detailed documentation on each helper.

---

## 📦 Included Helpers

| Helper                       | Description                                           |
| ---------------------------- | ----------------------------------------------------- |
| [**AnimationHelper**](./Contents/AnimationHelper/)     | Animation event callbacks and cross-fades             |
| [**AssetBundleHelper**](./Contents/AssetBundleHelper/)   | AssetBundle loading, caching, and version management  |
| [**AudioHelper**](./Contents/AudioHelper/)         | Audio pooling, mixing, and volume controls            |
| [**CameraHelper**](./Contents/CameraHelper/)        | Camera shakes, follow, and cinematic effects          |
| [**CoroutineHelper**](./Contents/CoroutineHelper/)     | Enhanced coroutine management and scheduling          |
| [**DamagePopupHelper**](./Contents/DamagePopupHelper/)   | Floating text and damage indicators                   |
| [**DebugHelper**](./Contents/DebugHelper/)         | In-game debug console, logs, and profiling UI         |
| [**DialogueHelper**](./Contents/DialogueHelper/)      | Dialogue trees, branching, and UI integration         |
| [**EditorGUIHelper**](./Contents/EditorGUIHelper/)     | Custom Editor windows, inspectors, and menu items     |
| [**FirestoreHelper**](./Contents/FirestoreHelper/)     | Firestore CRUD operations with Unity integration      |
| [**InputHelper**](./Contents/InputHelper/)         | Unified input abstraction (touch, keyboard, gamepad)  |
| [**JsonHelper**](./Contents/JsonHelper/)          | Advanced JSON: serialize, diff, merge, Firestore      |
| [**LocalizationHelper**](./Contents/LocalizationHelper/)  | Multi-language support, string tables, culture switch |
| [**LocalStorageHelper**](./Contents/LocalStorageHelper/)  | File-based local storage for persistent data          |
| [**NavigationHelper**](./Contents/NavigationHelper/)    | NavMeshAgent controllers and waypoint navigation      |
| [**NetworkHelper**](./Contents/NetworkHelper/)       | REST and WebSocket client utilities                   |
| [**ObjectPoolingHelper**](./Contents/ObjectPoolingHelper/) | Generic pooling for GameObjects and components        |
| [**ParticleHelper**](./Contents/ParticleHelper/)      | Particle system pooling, play/stop controls           |
| [**PrefsHelper**](./Contents/PrefsHelper/)         | Typed PlayerPrefs with cloud sync and JSON support    |
| [**SaveSystemHelper**](./Contents/SaveSystemHelper/)    | Binary and JSON save/load of game data                |
| [**SceneHelper**](./Contents/SceneHelper/)         | Async scene loading/unloading with progress callbacks |
| [**TimeHelper**](./Contents/TimeHelper/)          | Time formatting, countdowns, and timers               |
| [**UIHelper**](./Contents/UIHelper/)            | Common UI utilities: layout, widgets, dialogs         |

---

## 🔗 Documentation

Detailed docs and examples for each helper:

* `Docs/JsonHelper.md` – JSON utilities & Firestore
* `Docs/PrefsHelper.md` – PlayerPrefs & cloud sync
* `Docs/UIHelper.md` – UI workflows and patterns
* ... (and so on)

---

## 🛠 Usage

```csharp
// Example: Loading a scene with progress
await SceneHelper.LoadAsync("Level1", progress => {
    loadingBar.fillAmount = progress;
});

// Example: Saving game state
GameData data = new GameData { level = 2, lives = 3 };
SaveSystemHelper.SaveJson("save1", data);

// Example: Getting a localized string
string message = LocalizationHelper.GetString("Game_Over");
```

---

## 🎓 Contributing

1. Fork the repo
2. Create a feature branch
3. Submit a pull request with tests and documentation

---

## 📄 License

MIT License. See [LICENSE](LICENSE) for details.

---

## ✉️ Contact

Questions? Open an issue or email [maintainer@example.com](mailto:maintainer@example.com)
