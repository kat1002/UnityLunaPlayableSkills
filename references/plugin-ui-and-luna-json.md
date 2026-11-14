# Luna Plugin UI, `luna.json` & `<5 MB` Size Optimization Guide

Official documentation: [Unity Playworks Plugin UI](https://docs.lunax.app/docs/playable/plugin-ui)

---

## 1. Luna Build Pipeline Stages (`LunaTemp/`)

Understanding what Luna generates in `LunaTemp/` makes debugging fast:

| Stage Folder / File | Purpose | How to Use for Debugging |
|---|---|---|
| `LunaTemp/luna.log` | Full compiler, asset export, and bundler log | First file to inspect when any stage fails. |
| `LunaTemp/stage1/` | Exported Unity scenes (`assets/scenes/0.json`), prefabs (`prefabs/*.json`), and converted GLSL ES 3.00 shaders (`shaders/*.json`) | Check if serialized field values or shader properties exported properly. |
| `LunaTemp/stage2/` & `stage3/` | C#-to-JS transpilation workspace (`stage3/processed-scripts/`, `stage3/engine/unity/bin/UnityEngine.UI.js`, `stage3/engine/luna/script1.js`) | Confirm your modified C# script was actually copied into `processed-scripts` and check how Luna implements a Unity/UGUI API in JS. |
| `LunaTemp/stage4/develop/` | Local develop build (`index.html`, `engine/unity/bin/bridge.js`) served on `http://localhost:8002` | Check file timestamps to ensure you aren't testing a stale build! |
| `LunaTemp/stage4/create-hub/*_build.zip` | Compressed production bundle for uploading to Playworks / ad networks | Measure this `.zip` against the **< 5 MB** ad network limit. |

> [!TIP]
> Ensure `LunaTemp/` is listed in `.gitignore`. Never commit `LunaTemp/` to version control.

---

## 2. Plugin UI Tabs & `luna.json` Mapping

All settings configured in the Unity Playworks Plugin UI (`Window -> Unity Playworks` / `Luna`) are stored in **`luna.json`** at the Unity project root.

### 2.1 Settings Tab (`Basic`, `Advanced`, `Ad Network`, `Account`)
- **Startup & Included Scenes (`Basic`)**:
  - `unity.scenes`: Array containing **exactly one** active playable scene (e.g., `["Assets/ScenePA/ScenePA_Scoop_Mukbang.unity"]`).
  - `unity.disabledScenes`: Put **all** other scenes here so Luna does not export their assets.
  - `creativeName`: Follow naming convention `<Game>_<Theme>_PA_<NN>_<ddMMyy>_<Author>`.
- **Compiler & MSBuild (`Advanced`)**:
  - `unity.scripts.msbuildWin64`: `"C:\\Program Files (x86)\\Microsoft Visual Studio\\2022\\BuildTools\\MSBuild\\Current\\Bin\\MSBuild.exe"` (`msbuildVersion: "17.14"` + .NET Framework 4.7 Developer Pack).
  - Recommended build flags:
    - `minify: true` (shrinks JS output for production; can disable temporarily for faster local debug builds)
    - `forceBase64: false` for standard zip / Create Hub builds (`true` inflates size by ~33% for single-HTML networks)
    - `enableMecanimWASM: true`
    - `disableAntiAliasing: true`
    - `enableLunaCompilerV2: false`
    - `graphicsConstraint: 24`, realtime shadows off, batching off, `postprocesses.ssao.isActive: false`.
- **Ad Networks (`Ad Network`)**:
  - `packages`: Typically `["default", "ironsource", "facebook"]` (or configured in Playworks Create Hub).

---

### 2.2 Assets Tab (`Exclusions`, `Includes`, `Per-Asset Rules`)
- **Exclusions (`unity.assets.excludes`)**:
  - Always exclude `**/Editor/`, 3rd-party ad/analytics SDK folders (`FacebookSDK`, `AdMobPlugin`, `IronSource`, `AppsFlyer`, `Firebase`, `UnityAds`, `UnityPurchasing`), `Stubs/*`, and unused default TextMeshPro assets (`EmojiOne.png`, `LiberationSans SDF.asset`, `LiberationSans SDF - Fallback.asset`).
  - Exclude any unreferenced large textures or sprites reported in Size Breakdown.
- **Force-Includes (`unity.assets.includes`)**:
  - Keep `"Assets/TextMesh Pro/Resources/TMP Settings.asset"` (and `"Assets/Resources/DOTweenSettings.asset"` if DOTween is used). Force-includes override exclusion rules.
- **Texture Compression Rules**:
  - Max dimension: **`512`** (`textureWidth: 512`, `textureHeight: 512`).
  - Format: `png32` with `quantSize` compression at quality **`55–65`**.
  - Font SDF Atlas Textures (e.g., `MainFont SDF.asset`): Set texture rule `compression: "crush"` and `textureWidth/Height: 512` (or regenerate as ASCII-only `256x256` in Unity if the playable is English-only — saves ~120+ KB).
- **Audio Compression Rules**:
  - Force **mono** at **`64 kbps`** bitrate.
  - Use the Plugin UI's **Sound Timeline Cropping tool** to trim silence or unused tails from BGM/SFX clips.
- **Video Assets (`.mp4`)**:
  - **Luna does NO video transcoding** — `.mp4` files ship byte-for-byte. Pre-compress videos externally (e.g., `ffmpeg` H.264 baseline, low bitrate/resolution) if video is required.

---

### 2.3 Code Tab (`Exclusions`, `Stubs`, `Scripting Defines`)
- **Script Exclusions (`unity.scripts.excludes`)**:
  - Exclude `**/Editor/`, `**/Luna/RuntimeScripts/`, all ad/analytics SDKs, `**/Obi/`, `**/Sirenix/Odin Inspector/`, `**/StompyRobot/SRDebugger/`, and unused main-game modes.
  - Remember: **Luna transpiles every non-excluded C# script in the project** even if the active scene doesn't use it! Excluding unused game mode scripts can cut `bridge.js` / game JS size by megabytes.
- **DLL Limitation**:
  - Managed DLL plugins without C# source code are not supported by Luna's transpiler (except supported engine modules). Exclude them and provide lightweight C# stubs under `Stubs/` if needed.
- **Tree-Shaking Lists (`unusedModules`, `unusedClasses`, `unusedMethods`)**:
  - When porting `luna.json` to a **new** project, **clear** the large `unusedModules` / `unusedClasses` / `unusedMethods` arrays so Luna regenerates them cleanly for the new codebase (or un-force `physics3d` / `reflection` if needed).

---

### 2.4 Size Breakdown Tab & Measuring the `<5 MB` Budget
1. In the Plugin UI, open **Size Breakdown** and click **"Build & Estimate size"** to inspect the footprint of JS code, Textures, Audio, Fonts, Meshes, and JSON metadata.
2. Verify the actual compressed `.zip` size generated for Create Hub (`LunaTemp/stage4/create-hub/*_build.zip`) via CLI:
   ```powershell
   python -c "import zipfile,sys;z=zipfile.ZipFile(sys.argv[1]);print(round(sum(i.compress_size for i in z.infolist())/1048576, 3),'MB')" "LunaTemp/stage4/create-hub/<your_build>.zip"
   ```
3. **Top Size Culprits & Fixes**:
   - **JS > 1.5–2.5 MB**: Too many unused scripts compiled -> add unused folders to `unity.scripts.excludes` and enable `minify: true`.
   - **Textures > 1.5 MB**: Downscale textures to `<=512`, lower `quantSize` quality from `65` to `55`, crush SDF font atlases, and exclude unused sprites.
   - **Fonts > 150 KB**: Regenerate static TMP SDF font atlas with ASCII character set only at `256x256`.

---

## 3. Common Build & Preview Errors

### 3.1 Stage 4 `EBUSY: resource busy or locked, unlink '.../bridge.js'` (`Error: PACKAGE_BUNDLE`)
- **Cause**: The Unity Editor process itself locks `LunaTemp/stage4/develop/engine/unity/bin/bridge.js` while the **Luna in-editor preview web server is running** (`"Started Luna web server on port 8002"` in the Unity Console).
- **Fix**:
  1. Stop the Luna preview / close the Luna Playground preview in Unity (confirm `"Stopping Luna web server"` appears in Console).
  2. Delete `LunaTemp/stage2`, `LunaTemp/stage3`, and `LunaTemp/stage4` if needed (`stage1` holds the Unity export; `stage2..4` are safely regenerated).
  3. Rebuild. If the file lock lingers, restart the Unity Editor.

### 3.2 `LP1032: Package(s) not supported in Luna`
- **Cause**: Unsupported Unity packages (e.g., `com.unity.timeline`, XR, Terrain, Vehicles) are present in `Packages/manifest.json`.
- **Fix**: Grep the codebase for references; if unused in the playable, remove the package from `Packages/manifest.json`.

### 3.3 Browser Console: `X is not a function` or `Cannot set properties of undefined (setting 'fontRatio')`
- **`X is not a function`** when C# compiles cleanly: Either the build output in `stage4/develop` is stale (clear `LunaTemp/` and do a full rebuild) or a newer Unity API overload is missing in Luna's JS runtime (check `LunaTemp/stage3/`).
- **`Cannot set properties of undefined (setting 'fontRatio')` at `startGame`**: Happens when previewing `iframe.html` before `engine/luna/script1.js` finishes writing/loading (~5s after build). Hard-refresh (`Ctrl+F5`) after the build completes.

### 3.4 Hand Edits to `luna.json` Get Overwritten
- **Cause**: the Luna plugin window keeps its own in-memory copy of the config and writes it back on save/export, so
  edits made on disk (by hand or by a script/agent) while the window is open can be silently lost.
- **Fix**: after editing on disk, reload before building: `Tools/Luna/Refresh From Disk` (GangterCity `LunaTools`),
  or close/reopen the Luna window or force a script reload. Re-open `luna.json` after the build to confirm the edit stuck.
  Per-asset texture overrides on font assets are regenerated on every export regardless (see constraints 5.0).
