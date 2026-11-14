# Playable Ad Architecture & New Project Porting Guide

Based on `Doc/PA_Architecture_Porting.md`, `Assets/PlayableKit/`, and `Assets/MyLunaUtils/`.

---

## 1. Runtime Architecture Pattern (`IdolMukbang`)

Keep playable ad runtime code self-contained, flat, and free of main-game singletons:

1. **Single Coordinator Pattern**:
   - `PAGameFlow` (abstract state machine: `Playing` -> `StoreOpened` / `TimerComeback` -> `Ended`) is extended by the mode entry script (`PALiveGame` or `PASupermarketGame`).
   - `PALivePanel` holds all serialized scene references and constructs plain C# helper classes (`PALiveFoodSpawner`, `PALiveFeedZone`, `PALiveEatController`, `PALiveHud`, `PALiveTutorialHand`, `PALiveTransformFx`) in `Awake()`, ticking them from `PALivePanel.Update()`.
   - Helpers have **no `MonoBehaviour` inheritance and no singletons**.
2. **Two Luna Touch Points Only**:
   - Only `LunaManager` (`GameStarted`, `GameEnded`, `InstallFullGame`, timers) and `PAGameFlow` (`OnPause` / `OnResume` comeback bonus) call `Luna.Unity.*`.
3. **Pooling Over Runtime Instantiation**:
   - Clone dynamic items (chat comments, smoke/swap FX, emoji particles) from inactive templates already placed in the scene (`PALiveChat`, `EmojiParticlePool`).

---

## 2. Scene Hierarchy & Dual-Orientation Layout

Always **duplicate an existing working PA scene** (e.g., `Assets/ScenePA/ScenePA_Mode_Mukbang_Halloween_01.unity`) rather than building from scratch:

```text
_____________                 separator (Assets/_____________.prefab)
  LunaManager                 [LunaManager, PAIdleTimeout]
  Canvas_CTA_Ver              [Canvas order 999, 1080x1920 match 0.5, PASafeArea] > Ver > Btn (GoToStore), CTA
Ver                           [Canvas order 500, CanvasScaler 1080x1920 match 0.5, PALiveGame]
  RootLayout
    LiveController            [PALivePanel]
      BG/Content              [PAAspectFitter]
      Game                    Characters [PACharacter], Table/1..5 slots, FoodEmoji emitters, FX templates
      HideableUI              [CanvasGroup] PlayerInfo, HandTut, LiveChat [PALiveChat, PALayout], Emoji, Title
  SideCurtains/PlayArea       [PAAspectFitter] > CurtainL / CurtainR
Canvas_CTA_Hor                [Canvas order 999, 1920x1080 match 1] > Hori [PASafeArea] > Btn (GoToStore)
PASound                       [PASound] > Music, Sfx0-2, Noise (AudioSources)
EventSystem                   [EventSystem + StandaloneInputModule in Luna]
PAConfig                      [PAResolution, PAConfig]
Canvas                        background, order 0, 640x1136 > BG [PAAspectFitter]
Main Camera
```

- **Orientation Handling**:
  - Either use a single gameplay canvas with `PAResolution` + `PALayout` (applying a `PARectPreset` per orientation and toggling `Canvas_CTA_Ver` / `Canvas_CTA_Hor`),
  - Or toggle `Ver Canvas` / `Hor Canvas` via `ScreenOrientationChecker` (`GameEventManager` Landscape/Vertical events). When using dual canvases, capture rest `anchoredPosition` in `Awake()` and re-run layout in `OnEnable()`.

---

## 3. Reusable Kits & Editor Tooling

### 3.1 `Assets/PlayableKit/` (Standalone UI & FX Kit)
Depends only on `TweenLite.cs` (`Playable.Tweening`) and `UnityEngine.UI`:
- **Effects**: `UIShine` (requires `Materials/UI_Shine_M.mat` with `PlayableKit/UI_Shine` shader), `UIStarTwinkle`, `UIPathTwinkle` (arc/bezier/child-point sparkle wave), `UIIdleMotion` (sine bob/sway/pulse/fade), `UITweenRotate`, `FlashTrailRendererDriver`, `FallWindFx`.
- **Tweens**: `DoScaleAToB`, `DoMoveAToB`, `DoFadeAToB`, `ButtonRotate`, `UiSlide`, `UiPageSlide`.
- **UI**: `HoldButton`, `UiDragRotate`, `BillboardToCamera`.

### 3.2 Editor Tool Locations (Which Project Has What)
Tools are **not** in every project. Grep for the class name before telling the user to run a menu.
Prefer the package versions; legacy copies are older forks of the same idea.

| Tool | Menu | Where it lives |
|---|---|---|
| PlayableKit Control Center (dependency/code scanner, Luna compliance + Spine `.skel` scan, UI sanitizer, exporter) | `Tools/PlayableKit/PlayableKit Control Center` (`Ctrl+Shift+K`) | Package `com.kat.playablekit` (`PlayableKit/Packages/com.kat.playablekit/Editor/`) |
| `PALunaUiSanitizer` | `Tools/PlayableKit/Sanitize UI For Luna` | Package; StickiiPA `Assets/PlayableKit/Editor/` |
| `PALunaUiSanitizer` (legacy) | `Tools/PA/Sanitize UI For Luna` | IdolMukbang `Assets/Scripts/Playable/Supermarket/Editor/` |
| `PALunaBuildPatcher` (auto-patches `LunaTemp` JS: physics mesh-shape inertia, VideoPlayer looping; see constraints section 6) | `Tools/PlayableKit/Luna/Apply Build Patches Now` | Package |
| `PAAudioImportOptimizer` (WebGL SFX/Music import presets) | `Tools/PlayableKit/Audio Import Optimizer` | Package |
| `PAPackageExporter` | `Tools/PlayableKit/Export PlayableKit.unitypackage` | Package |
| `PAShoreFieldGenerator` (hole distance field, alpha-254 PNG) | `Tools/PlayableKit/Shore Field Generator` | StickiiPA only |
| `PlayableAdsCheckerWindow` | `Tools/Sp Unity/Playable Ads Checker` | IdolMukbang `MyLunaUtils/Editor`, SatisFix (Home-Clean, perfectcleaning) `Assets/Editor`, `Projects/Package/` |
| `MyLunaComplianceWindow`, `LunaSpineSceneSkelScanner`, `FindAssetByID`, `CleanerTool` | `Tools/MyLunaHelper/...` (Compliance Tool, Spine Binary Inspector, Find Asset By ID, Force Remove All AssetBundles) | `Assets/MyLunaUtils/Editor/` in IdolMukbang, SatisFix, ScoopAndPack |
| `LunaPreflightValidator` + shared `LunaPreflightChecks` (scene readiness, serialized refs, Luna config) | `BikePlayable/Preflight Validation` | BikePlayable (`Assets/Editor/`, `Assets/_Template/Scripts/Editor/`) |
| `LunaTools` (`LunaConfigBridge`, `LunaAssetsSyncOverlay`): reflection bridge into Luna's `EditorScripts.dll` (SHA-256 guarded, disk-only fallback) to re-read / force-save `luna.json`, plus a Refresh/Apply bar on the Luna window | `Tools/Luna/Refresh From Disk`, `Apply To luna.json`, `Diagnose Bridge` | GangterCity `Assets/Editor/LunaTools/` |
| `NoRuntimeResourceLoadTests` (EditMode test, guards runtime `Resources.Load`) | Test Runner | IdolMukbang `Assets/Editor/` |

Reusable shaders: GangterCity `Assets/Shaders/LunaMapStandard{Opaque,Fade,Transparent}.shader`
(Standard-lookalike clones that survive Luna) and `LunaFakeShadow.shader` (texture-free unlit ellipse/box decal shadow).

**Local Luna SDK copies** (read engine JS / pipeline when root-causing a runtime bug):
`D:/Phuc/Projects/Home-Clean-Playable/Doc/LunaSDK/6.4.0/` and `.../7.0.0/` (`engine/`, `pipeline/`, `scripts/`, `tools/`).
Decompiled plugin (`EditorScripts.dll`) behaviour notes: [constraints 5.0](./constraints-and-gotchas.md).

### 3.3 Pre-Build Run Order
1. Sanitize UI For Luna (rows above).
2. Control Center scan, or Playable Ads Checker + Compliance Tool + Spine Binary Inspector.
3. Project preflight and EditMode tests, if the project has them.

---

## 4. Checklist for Porting to a New Unity Project

1. **Unity & Packages**:
   - Create/open project in Unity **2022.3.62f2**.
   - Reference `com.unity.playworks.upp` (Luna 7.2.0 `scripts` folder), TextMeshPro, UGUI, 2D Sprite packages, and **one** Spine runtime source only.
   - Ensure VS 2022 Build Tools (**MSBuild 17.14**) and **.NET Framework 4.7 Developer Pack** are installed on Windows.
2. **Copy Core Folders As-Is**:
   - `Assets/Scripts/Playable/` (including `Editor/`)
   - `Assets/Scripts/Effect/`
   - `Assets/Scripts/Luna/LunaManager.cs`
   - `Assets/Scripts/Utils/TweenLite.cs`, `CountingLabel.cs`, `StringUtils.cs`
   - `Assets/MyLunaUtils/` and `Assets/PlayableKit/`
   - `Assets/Playable/Supermarket/Prefabs/` and `Assets/_____________.prefab`
   - `Assets/csc.rsp`, `Assets/SVC_Luna.asset`, `Assets/Editor/NoRuntimeResourceLoadTests.cs`
   - Template scene(s) from `Assets/ScenePA/`
3. **Untangle Main-Game Enums**:
   - Extract standalone enums (`TScreen`, `FoodTaste`, `FoodSound`, `CharacterState`, `CharacterEmoteType`, `FoodAnimType`) without pulling in main-game classes (`DataServices`, `SoundConfig`, `CountFXManager`, `UIHeader`).
4. **Configure `luna.json`**:
   - Reset `projectId`, `applicationID`, `creativeName`, `pgServer`.
   - Enable only the new PA scene in `unity.scenes`; place all others in `unity.disabledScenes`.
   - Clear `unusedModules`, `unusedClasses`, and `unusedMethods` so Luna regenerates them for the new project.
5. **Verify & Build**:
   - Run **Sanitize UI For Luna** and the Control Center scan (or legacy **Playable Ads Checker**); see 3.2 for which exists.
   - Stop any running Luna preview web server, run a full Luna build, and test both Portrait and Landscape orientations, CTA clicks, store return, idle timeout, and end timer in the browser.
