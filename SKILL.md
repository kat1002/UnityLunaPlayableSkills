---
name: unity-luna-playworks
description: >-
  Build, optimize, debug, and port Unity Playable Ads using Unity Playworks (Luna / LunaX).
  Use when authoring Luna-safe C# and UGUI, configuring the Luna Plugin UI or luna.json,
  optimizing build sizes under 5MB, wiring Luna.Unity.LifeCycle / Playable / Analytics / LunaPlaygroundField,
  sanitizing UI (ScrollRect, LayoutGroup, Mask), or troubleshooting Luna Stage 1-4 build and runtime bugs.
---

# Unity Luna Playworks (Playable Ads) Skill

Unity Playworks (formerly Luna Playable / Luna Labs) transpiles Unity C# and UGUI into JavaScript (`LunaTemp/stage3/`) and bundles assets into a lightweight HTML5 WebGL package (`LunaTemp/stage4/`). **Code and UI that work in the Unity Editor frequently fail or behave differently in a Luna build.**

Follow the guardrails and workflows below whenever building, refactoring, debugging, or porting a Unity Playable Ad.

---

## 1. Hard Safety Rules (Read First)

1. **NEVER pack SpriteAtlases via Unity MCP (`execute_code`)**:
   - Do **not** call `UnityEditor.U2D.SpriteAtlasUtility.PackAtlases` (or any programmatic atlas packing) via `execute_code`. It causes a hard native crash in `PackSpriteAtlases` in Unity 2022.3.62 (`%LOCALAPPDATA%/Temp/Unity/Editor/Crashes`).
   - Creating or configuring `.spriteatlas` assets via script is safe; leave packing to the user (**Inspector → Pack Preview**) or to Play Mode / Build with Sprite Packer mode `"Always Enabled"`.
   - Keep source PNGs for atlases **uncompressed** in Unity texture import settings to avoid SpriteAtlas packing warnings.
2. **NEVER hand-edit `.unity` scene YAML files**:
   - Modify scenes only via Unity MCP tools or Editor scripts (`SerializedObject`), and **always stop Play Mode first** before compiling scripts or saving scenes (`MarkSceneDirty` / `SaveScene`).
3. **Full Luna Build Required After Schema Changes**:
   - Incremental "Recompile" does not update serialized layouts. After adding, renaming, or retyping any `[SerializeField]` or component, perform a **full Luna build**.
4. **Stop Luna Preview Web Server Before Rebuilding**:
   - If Stage 4 fails with `EBUSY: resource busy or locked, unlink '.../LunaTemp/stage4/develop/engine\unity\bin\bridge.js'`, the Unity Editor's Luna preview web server (`port 8002`) is holding a file handle on `bridge.js`. Stop the Luna preview / close the Playground window before building.

---

## 2. Quick Do / Don't Cheat Sheet (Luna-Safe C# & UI)

| Area | ❌ Don't (Breaks or Warns in Luna) | ✅ Do Instead (Luna-Safe) |
|---|---|---|
| **UGUI Layout** | `ScrollRect`, `Scrollbar`, `Slider`, `LayoutGroup` (H/V/Grid), `ContentSizeFitter`, `AspectRatioFitter` (`DrivenRectTransformTracker.Clear` warning & lazy first-open bugs) | Run **Tools/PA/Sanitize UI For Luna** (`PALunaUiSanitizer`) to bake rects; use `PAAspectFitter`, `PADragScroll`, `PALayout` |
| **Sprite shader** | URP `2D/Sprite-Unlit-Default` on a SpriteRenderer (skinning variant uses an `std430` SSBO -> `Failed to compile shader program` in WebGL) | Built-in `Sprites/Default` material (`Sprites-Default.mat`) |
| **UGUI Hiding** | `graphic.enabled = false` to hide (Luna's `Graphic.OnDisable` never clears the `CanvasRenderer` -> last mesh keeps drawing, stale/ghost UI in the build) | `graphic.gameObject.SetActive(false)` (guard with `activeSelf` if called per frame) |
| **UGUI Clipping** | `RectMask2D` (`SetClipRect` is an unimplemented no-op in Luna's `UnityEngine.UI.js`) | Use `Mask` with a plain (`Simple`, non-Sliced) `Image` stencil; call `SetVerticesDirty()` if resized at runtime |
| **EventSystem** | `InputSystemUIInputModule` (missing in Luna's `UnityEngine.UI.js` -> dead UI clicks) | Guard new Input System with `#if ENABLE_INPUT_SYSTEM && !UNITY_LUNA`; ensure `StandaloneInputModule` is added under `#if UNITY_LUNA` |
| **Serialization** | Generic `[Serializable]` types (e.g., `[SerializeField] Oriented<T>` -> silently `null` in Luna 7.2) | Use concrete non-generic `[Serializable]` classes |
| **Component Lookup** | `GetComponentInParent<T>(bool includeInactive)` (CS1501 overload missing in Luna) | Walk `transform.parent` manually and call `GetComponent<T>()` |
| **C# Syntax & APIs** | `Resources.Load` (runtime), `PlayerPrefs`, `JsonUtility`, `System.Reflection`, `try/catch`, `goto`, target-typed `new()`, `is not`/`and`/`or`, `^` indexers, `Array.Empty<T>()`, `UniTask`, `OnMouseDown`, `GameObject.Find` / `FindObjectOfType` in hot paths | Use serialized references, plain C# 7.3 syntax, simple loops, and pooled scene templates (`Instantiate` from inactive scene template) |
| **Coroutines** | Nested `IEnumerator` whose **last** `yield return` is another bare `IEnumerator` (kills parent coroutine in Luna's `MoveNext`) | Wrap nested helpers in `yield return StartCoroutine(Helper())`, or use `Update`-driven timers / `DOVirtual.DelayedCall(t, NamedMethod)` |
| **Tweens** | DOTween `Sequence`, chained lambdas/closures | Single `DOVirtual.Float` callback (`PATween.Play` + `Seg`) on `TweenLite` (`Playable.Tweening`) with named methods |
| **Animator** | `Animator.StringToHash` int overloads (`SetTrigger(hash)` loses animations in Luna) | Always use string overloads: `SetBool("IsMove", true)`, `SetTrigger("Die")` |
| **Particles** | ParticleSystem `Trails` module (ribbon) and `Bursts` / sprite-sheet start frame (vanish in Luna) | Use `TrailRenderer` on a child object; use `rateOverTime` / `rateOverDistance` |
| **Custom Shaders** | Dividing `Sprite.textureRect` (native px) by `Texture.width/height` (Luna downscaled `<=512`), or mutating a cached `Vector4` passed to `Material.SetVector` | Normalize/clamp UV rects (`>1` -> `(0,0,1,1)`), bake sprite sheet rects in Editor, and pass `new Vector4(...)` to `Material.SetVector` |
| **UI Shader Position** | Deriving position from `v.vertex` / `unity_ObjectToWorld` in a UI shader (space differs: Game view vs Scene view vs Luna) | Map from sprite UV with a per-material affine set from C# (world corners); `v.vertex` only for `_ClipRect` |
| **Data Textures** | Fully opaque texture (or `RGB24` default import format) -> Luna exports lossy JPEG, ignoring `luna.json` compression/format; WebGL override ignored when active target is Standalone | Bake alpha 254 into the PNG, default format `RGBA32` uncompressed, `luna.json` `"compression": "none"`; verify `ext: ".png"` in `LunaTemp/stage2/assets/textures/<id>.json` |
| **Audio** | `AudioListener.pause` (CS0117, not in Luna) | `AudioListener.volume = 0f` on pause, restore the saved volume on resume |
| **URP Renderer** | URP **2D Renderer** (`Renderer2DData`) on Unity 6 / URP 17 -> Stage 1 `LP1025` (`copyDepthMode` not found) | Switch the URP asset to a `UniversalRendererData`; remove template `Global Light 2D` |
| **Sliced UV effects** | Sweep/gradient shaders reading raw UVs on `Sliced` images (band jumps/bends) | Remap sliced UVs with border uniforms (`PAUIShine.RefreshSlice`) |
| **Spine & Shaders** | Spine binary `.skel.bytes`, duplicate Spine runtimes, URP `Lit` / `Shader Graph` materials | Export Spine as `.json` (`skeletonJSON`), keep 1 Spine package, use `Sprites/Default`, `UI/Default`, `Unlit/Texture`, or `PlayableKit/UI_Shine` |

---

## 3. Standard Workflows

### Workflow A: Wiring Luna Lifecycle, Timers & CTA
1. Place one `LunaManager` (and `PAIdleTimeout` if used) in the playable scene.
2. **First Interaction**: Call `LunaManager.StartGame()` on the first pointer down -> triggers `Luna.Unity.LifeCycle.GameStarted()` once and starts the end timer (`currentEndTime`).
3. **In-Game CTA Buttons**: Route `Button.onClick` to `LunaManager.GoToStore()` -> calls `StartGame()` and `Luna.Unity.Playable.InstallFullGame()` **without** ending the game.
4. **End Card / Final Exit**: Call `LunaManager.EndGame(bool openStore)` -> calls `Luna.Unity.LifeCycle.GameEnded()` **at most once**, and opens the store if `openStore` is true.
5. **Pause / Resume & Store Return**: Subscribe to `Luna.Unity.LifeCycle.OnPause` and `OnResume`. **Never** gate store-return logic with `Time.unscaledTime` (which freezes while the tab is hidden); use `Time.realtimeSinceStartup`, frame counts, and `LunaManager.ClampTickSeconds(Time.unscaledDeltaTime)`.
- See [Luna API & Lifecycle Reference](./references/luna-api-and-lifecycle.md) for code patterns and `[LunaPlaygroundField]` usage.

### Workflow B: Configuring Plugin UI, `luna.json` & `<5 MB` Budget
1. Enable **only one** target PA scene in `luna.json` (`unity.scenes`) and move all other scenes to `unity.disabledScenes`.
2. Exclude unused code (`**/Editor/`, Ad SDKs, Odin, Obi, SRDebugger, unused game modes) in `unity.scripts.excludes` and unused assets in `unity.assets.excludes`.
3. Keep `Assets/TextMesh Pro/Resources/TMP Settings.asset` in `unity.assets.includes`.
4. Cap textures at `512` px (`png32`, `quantSize` quality `55–65`), audio at mono `64 kbps`, and SDF font atlases at `256x256` or `512x512` (`crush` compression, ASCII-only when English-only).
5. Measure the `create-hub` zip (`LunaTemp/stage4/create-hub/*_build.zip`) to verify it is `<5 MB`.
- See [Plugin UI & `luna.json` Guide](./references/plugin-ui-and-luna-json.md) for full `luna.json` tuning and stage-by-stage build diagnostics.

### Workflow C: Pre-Build Verification & Scene Sanitization
Before triggering a Luna build, run the project's Editor verification tools. **Tools differ per project**: check the
[Tool Locations table](./references/architecture-and-porting.md#32-editor-tool-locations-which-project-has-what) and
use the `com.kat.playablekit` package menus when present, the legacy per-project copies otherwise.
1. **Sanitize UI For Luna** (`PALunaUiSanitizer`; `Tools/PlayableKit/...` in the package, `Tools/PA/...` in legacy copies) — Bakes and removes `LayoutGroup` & `ContentSizeFitter`, converts `AspectRatioFitter` -> `PAAspectFitter`, converts `ScrollRect` -> `PADragScroll`.
2. **Dependency / code scan** — `Tools/PlayableKit/PlayableKit Control Center` (`Ctrl+Shift+K`) in the package, or legacy `Tools/Sp Unity/Playable Ads Checker` (`PlayableAdsCheckerWindow`). Scans active scene dependencies for missing scripts/refs, texture/audio usage, and forbidden C# patterns (`Resources.Load`, `PlayerPrefs`, C# 9+ syntax, `try/catch`, `FindObjectOfType`, etc.).
3. **Compliance & Spine** — Control Center compliance panel, or legacy `Tools/MyLunaHelper/Compliance Tool` (`MyLunaComplianceWindow`) & `Spine Binary Inspector`. Checks for `.skel.bytes`, URP/Lit/ShaderGraph materials, and blocked DOTween files.
4. **Project preflight**, if the project has one (e.g. `BikePlayable/Preflight Validation`).
5. Projects with the package's `PALunaBuildPatcher` re-patch `LunaTemp` JS automatically; force it with `Tools/PlayableKit/Luna/Apply Build Patches Now` after a build.
- Hand-edited `luna.json` while the Luna window is open may be overwritten from memory: reload it (`Tools/Luna/Refresh From Disk` where `LunaTools` exists, otherwise reopen the window / reload scripts). See [Plugin UI guide](./references/plugin-ui-and-luna-json.md#34-hand-edits-to-lunajson-get-overwritten).
- See [Architecture & Porting Guide](./references/architecture-and-porting.md) for scene hierarchy templates, coordinator architecture, and new-project bring-up steps.

---

## 4. Progressive Disclosure References

Read the relevant reference file when working on a specific area:
- **[Luna API & Lifecycle](./references/luna-api-and-lifecycle.md)**: `Luna.Unity.LifeCycle`, `Playable.InstallFullGame`, `Analytics`, `[LunaPlaygroundField]`, `LunaManager.Session`, and store-return pause handling.
- **[Plugin UI & `luna.json` Optimization](./references/plugin-ui-and-luna-json.md)**: Official Plugin UI tabs (`Settings`, `Assets`, `Code`, `Build & Upload`, `Size Breakdown`), `luna.json` configuration, `<5 MB` size reduction playbook, and `EBUSY` / `LP1032` fixes.
- **[Constraints & Runtime Gotchas](./references/constraints-and-gotchas.md)**: Deep dive into Luna C#-to-JS transpiler limits, `EventSystem` / polled input bugs, `Mask` vs `RectMask2D`, custom shader UV/Vector4 bugs, nested coroutine termination, and tween/animator rules.
- **[PA Architecture & Porting](./references/architecture-and-porting.md)**: `PALiveGame` / `PAGameFlow` / `PALivePanel` coordinator pattern, dual-orientation canvas setup, `PlayableKit`, which project has which Editor tool, local Luna SDK copies, and step-by-step checklist for porting to a new Unity project.
