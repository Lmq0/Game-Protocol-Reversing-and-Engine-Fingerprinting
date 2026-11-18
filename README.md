# Game-Protocol-Reversing-and-Engine-Fingerprinting

## 🎯 Motivation & Objective

Have you ever wondered what truly lies beneath the glossy UI of a mobile game? How does the game engine know exactly which image to render when you summon a pet, and how are thousands of assets organized inside a single APK file?

This project is driven by pure technical curiosity: **to demystify the internal architecture of an Android game and understand how its resources interact with the underlying engine.**

Unlike standard applications that simply unpack their assets, this game employs a custom UE3 engine and aggressively obfuscates its resources. The primary objectives of this repository are:

1.  **Deconstruct the APK:** To explore the directory structure and trace the execution flow from the Java layer (`classes.dex`) down to the native C++ layer (`libHOGame.so`).
2.  **Fingerprint the Engine:** To analyze the binary file and identify the core rendering and logic engines (e.g., Unreal Engine 3, Lua VM) buried within.
3.  **Trace Asset Pipelines:** To investigate how the game's visual and audio resources are encoded, disguised, and mapped. Specifically, understanding how the engine parses its custom binary formats (e.g., custom magic numbers, zlib-compressed chunks) during runtime.
4.  **Bridge Network and Assets:** To connect the dots between the network protocol (how the server tells the client what to load) and the local asset files (how the engine actually finds and loads them).

This is a journey of peeling back the layers of obfuscation to see the raw, unadulterated mechanics of a custom game engine.

## Phase 1: APK Extraction & Engine Fingerprinting

### Step 1: Unpacking the APK & Directory Inspection
*   **Tools Used**: WinRAR / 7-Zip / MT Manager
*   **Action**: Treated the game `.apk` file as a standard `.zip` archive and extracted it.
*   **Purpose & Finding**: To quickly investigate the asset structure. We discovered that inside the `lib` directory, there is only one massive 41MB dynamic library: `libHOGame.so`. Furthermore, in the `assets` directory, we found a large number of files with numeric names (e.g., `1527585152.ast.png`) and a folder named `DataAndroid`. This initially indicated that the game assets are heavily packaged or obfuscated.

```
📦 game_apk_extracted/
├── 📁 assets/
│   ├── 📁 Audio/                  # Game background music and sound effects (音效)
│   ├── 📁 DataAndroid/            # 🎯 Core target: Custom UE3 asset chunks (核心资产包)
│   │   ├── 📄 884290.ast.png      # Renamed UE3 binary packages (伪装成图片的资源包)
│   │   ├── 📄 997926831.ast.png   # Contains custom magic number (86 C7 E4 47)
│   │   ├── 📄 1527585152.fst.png  # Fragmented data chunks (碎片数据块)
│   │   └── ......
│   ├── 📁 Music/                  # Music tracks (音乐)
│   ├── 📄 54107989.mst.png        # Renamed Music file (伪装后的音乐文件)
│   ├── 📄 813231027.mst.png       # Music file
│   └── ......
│
├── 📁 lib/                        # Native Libraries (C++ 动态链接库)
│   └── 📁 arm64-v8a/
│       └── ⚙️ libHOGame.so        # 🎯 41MB Custom UE3 Engine (自研/魔改的虚幻3引擎核心)
│                                  # Contains: Lua VM, UE3 classes, Socket logic
│
├── 📁 kotlin/                     # Kotlin metadata (Kotlin 元数据目录)
│
├── 📁 META-INF/                   # APK signature and manifest (签名验证信息)
│
├── 📁 res/                        # Standard Android resources (安卓标准资源)
│
├── 📄 AndroidManifest.xml         # App permissions and components (安卓应用清单)
├── 📄 billing.properties          # Google Play in-app billing config (谷歌内购配置)
├── 📄 classes.dex                 # 7.9MB Java/Kotlin bytecode (Java 层可执行代码)
│
├── 📄 firebase-encoders.properties# 
├── 📄 play-services-*.properties  # Third-party Google SDK configs (谷歌服务SDK配置)
├── 📄 transport-api.properties    # 
├── 📄 resources.arsc              # Compiled Android resource mapping (编译后的资源映射表)
└── 📄 DebugProbesKt.bin           # Debug tools metadata
```

### Step 2: Analyzing `classes.dex` (Java Layer Logic)
*   **Tools Used**: JADX-GUI (Open-source decompiler)
*   **Action**: Dragged the `.apk` directly into JADX-GUI and waited for the decompilation process to finish.
*   **Purpose & Finding**: To examine the game's entry point and loading logic. Inside the `com.jbgames.HOGame` package, we found the `GameApp` class, which contains a critical line of code: `System.loadLibrary("HOGame");`. This proves that the game loads the 41MB C++ native library upon startup. Furthermore, we noticed that core network requests and gameplay logic are entirely absent in the Java layer, indicating that they are all handled deep within the C++ code.

<div align="center">
  <img src="images/jadx_decompile.png" alt="JADX Decompilation" width="600" style="border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.2);">
  <br>
  <em>Figure 2: JADX-GUI showing the System.loadLibrary call in GameApp.</em>
</div>

### Step 3: Reverse Engineering the `.so` File (Engine Fingerprinting)
*   **Tools Used**: HxD (A lightweight hex editor)
*   **Action**: Opened `libHOGame.so` in HxD and used `Ctrl+F` to search for plain-text strings.
*   **Purpose & Finding**: **This was the decisive step in identifying the game engine.** Since `.so` is compiled machine code, we cannot read the source directly, but we can search for string constants. We found:
    *   `luaL_loadbuffer`, `lua_`: Proving the engine embeds a Lua scripting virtual machine.
    *   `ParticleSystem`, `AnimSequence`, `SlateUI`: These are exclusive class names from **Unreal Engine 3 (UE3)**.
    *   Therefore, we confirmed that this game is developed based on a heavily customized version of the UE3 engine.

<div align="center">
  <img src="images/hxd_engine_fingerprint.png" alt="HxD Engine Fingerprint" width="600" style="border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.2);">
  <br>
  <em>Figure 3: HxD search results showing UE3 class names and Lua APIs inside libHOGame.so.</em>
</div>

### Step 4: Asset File Header Analysis (Capturing the Magic Number)
*   **Tools Used**: HxD / Python Scripts
*   **Action**: Opened several `.ast.png` and `.fst.png` files in the `DataAndroid` directory and inspected the first 4 bytes of their headers.
*   **Purpose & Finding**: We discovered that all files consistently start with `86 C7 E4 47`. This is **NOT** the standard UE3 magic number (which should be `C1 83 2A 9E`). This is a custom "fake magic number" applied by the game developers. By scanning the directory, we confirmed that the vast majority of the asset files share this custom header to bypass standard unpacking tools.

<div align="center">
  <img src="images/hxd_magic_number.png" alt="Custom Magic Number" width="600" style="border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.2);">
  <br>
  <em>Figure 4: HxD showing the custom magic number "86 C7 E4 47" at the beginning of the asset files.</em>
</div>
