# UnityHelperToolkit 🛠️

[![Unity](https://img.shields.io/badge/Unity-2021.3%2B-blue.svg)](https://unity.com/)
[![.NET Standard](https://img.shields.io/badge/.NET-Standard%202.1-blueviolet.svg)](https://docs.microsoft.com/en-us/dotnet/standard/net-standard)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](#-license)
[![C#](https://img.shields.io/badge/language-C%23-239120.svg)](https://docs.microsoft.com/en-us/dotnet/csharp/)
[![Newtonsoft.Json](https://img.shields.io/badge/Newtonsoft.Json-supported-brightgreen.svg)](https://www.newtonsoft.com/json)
[![Firebase](https://img.shields.io/badge/Firebase-supported-orange.svg)](https://firebase.google.com/)
[![Firestore](https://img.shields.io/badge/Firestore-supported-orange.svg)](https://firebase.google.com/products/firestore)

# Available On
[![Windows](https://img.shields.io/badge/Windows-supported-blue.svg)](https://www.microsoft.com/windows/)
[![macOS](https://img.shields.io/badge/macOS-supported-lightgrey.svg)](https://www.apple.com/macos/)
[![Linux](https://img.shields.io/badge/Linux-supported-yellowgreen.svg)](https://www.linux.org/)
[![Android](https://img.shields.io/badge/Android-supported-brightgreen.svg)](https://www.android.com/)
[![iOS](https://img.shields.io/badge/iOS-supported-lightgrey.svg)](https://www.apple.com/ios/)

A comprehensive collection of helper classes to accelerate Unity C# project development. Each helper focuses on a specific domain—UI, time, scenes, serialization, networking, and more—providing clean, reusable functionality.

## ✨ Features

- 🎮 **25+ Helper Classes** - From animation to UI, everything you need
- 🔌 **Plug and Play** - Easy integration into existing Unity projects
- 📚 **Well Documented** - Detailed documentation and examples for each helper
- ⚡ **Performance Optimized** - Built with efficiency in mind
- 🧪 **Tested** - Thoroughly tested and production-ready

## 🚀 Requirements

- Unity 2020.3 or higher
- .NET Framework 4.x / .NET Standard 2.0
- Dependencies (installed via Unity Package Manager):
  - Newtonsoft.Json
  - Firebase SDK (optional, for cloud features)

## 📥 Installation

1. **Clone or download** this repository into your Unity project's `Assets/` folder:
   ```bash
   git clone https://github.com/yourusername/UnityHelperToolkit.git Assets/UnityHelperToolkit
   ```
2. **Install dependencies** via the Unity Package Manager or manually
3. **Navigate** to `Assets/UnityHelperToolkit/Readme` for detailed documentation

## 📦 Included Helpers

| Helper                                                    | Description                                           |
| ----------------------------                              | ----------------------------------------------------- |
| [**AnimationHelper**](./Contents/AnimationHelper/)        | Animation event callbacks and cross-fades             |
| [**AssetBundleHelper**](./Contents/AssetBundleHelper/)    | AssetBundle loading, caching, and version management  |
| [**AudioHelper**](./Contents/AudioHelper/)                | Audio pooling, mixing, and volume controls            |
| [**CameraHelper**](./Contents/CameraHelper/)              | Camera shakes, follow, and cinematic effects          |
| [**CoroutineHelper**](./Contents/CoroutineHelper/)        | Enhanced coroutine management and scheduling          |
| [**DamagePopupHelper**](./Contents/DamagePopupHelper/)    | Floating text and damage indicators                   |
| [**DebugHelper**](./Contents/DebugHelper/)                | In-game debug console, logs, and profiling UI         |
| [**DialogueHelper**](./Contents/DialogueHelper/)          | Dialogue trees, branching, and UI integration         |
| [**EditorGUIHelper**](./Contents/EditorGUIHelper/)        | Custom Editor windows, inspectors, and menu items     |
| [**EventHelper**](./Contents/EventHelper/)                | Global event system for decoupled communication       |
| [**FirestoreHelper**](./Contents/FirestoreHelper/)        | Firestore CRUD operations with Unity integration      |
| [**IAPHelper**](./Contents/IAPHelper/)                    | IAP helper that does all the dirty works for you      |
| [**InputHelper**](./Contents/InputHelper/)                | Unified input abstraction (touch, keyboard, gamepad)  |
| [**JsonHelper**](./Contents/JsonHelper/)                  | Advanced JSON: serialize, diff, merge, Firestore      |
| [**LocalizationHelper**](./Contents/LocalizationHelper/)  | Multi-language support, string tables, culture switch |
| [**LocalStorageHelper**](./Contents/LocalStorageHelper/)  | File-based local storage for persistent data          |
| [**NavigationHelper**](./Contents/NavigationHelper/)      | NavMeshAgent controllers and waypoint navigation      |
| [**NetworkHelper**](./Contents/NetworkHelper/)            | REST and WebSocket client utilities                   |
| [**ObjectPoolingHelper**](./Contents/ObjectPoolingHelper/)| Generic pooling for GameObjects and components        |
| [**ParticleHelper**](./Contents/ParticleHelper/)          | Particle system pooling, play/stop controls           |
| [**PrefsHelper**](./Contents/PrefsHelper/)                | Typed PlayerPrefs with cloud sync and JSON support    |
| [**SaveSystemHelper**](./Contents/SaveSystemHelper/)      | Binary and JSON save/load of game data                |
| [**SceneHelper**](./Contents/SceneHelper/)                | Async scene loading/unloading with progress callbacks |
| [**SuperScrollRect**](./Contents/SuperScrollRect/)        | infinite scroll with recycling functionallity         |
| [**TimeHelper**](./Contents/TimeHelper/)                  | Time formatting, countdowns, and timers               |
| [**TutorialHelper**](./Contents/TutorialHelper/)          | Tutorial sequences and steps with actions             |
| [**UIHelper**](./Contents/UIHelper/)                      | Common UI utilities: layout, widgets, dialogs         |


## DesignPattern Helpers

| DesighnPattern                                                          | Description
| ---------------------------                                             | -----------------------------------------------------                             |
| [**Factory Pattern**](./Contents/DesignPatternHelpers/FactoryPattern/)  | a generic factory pattern with **object pooling** support for GameObjects         |
| [**Observer Pattern**](./Contents/DesignPatternHelpers/ObserverPattern/)| an observer/event system abstraction                                              |
| [**State Pattern**](./Contents/DesignPatternHelpers/StatePattern/)      | A robust and helpful implementation to help using state pattern faster and simpler|
| [**Strategy Pattern**](./Contents/DesignPatternHelpers/StrategyPattern/)| Useing this helper is a greate strategy to implement strategy pattern             |
| Extra helpers                                                           |
| [**State Machine**](./Contents/DesignPatternHelpers/StateMachine/)      | A generic state machine system to do the dirty works for you                      |


## 🛠️ Usage Examples

```csharp
// Loading a scene with progress updates
await SceneHelper.LoadAsync("Level1", progress => {
    loadingBar.fillAmount = progress;
});

// Saving game state with JSON
GameData data = new GameData { level = 2, lives = 3 };
SaveSystemHelper.SaveJson("save1", data);

// Getting localized text
string message = LocalizationHelper.GetString("Game_Over");
```

## 📚 Documentation

Each helper comes with detailed documentation and examples in its respective folder under `Contents/`. Navigate to each helper's directory to find specific documentation and usage examples.

- 📖 [Helpers Documentation](./Contents/)

## 🤝 Contributing

We welcome contributions! Here's how you can help:

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the MIT License.

MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

## 💬 Support & Contact

- 📧 Email: [najafzadehkaren@gmail.com](mailto:najafzadehkaren@gmail.com)
- 🐛 [Issue Tracker](https://github.com/Karen-Najafzadeh/UnityHelperSDKDocumentation/issues)
