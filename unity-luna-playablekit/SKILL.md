---
name: unity-luna-playablekit
description: >-
  Work with PlayableKit (com.kat.playablekit), the company's Luna-safe toolkit for Unity playable ads: install or
  migrate it, start a PA from its Template Scene, use PALunaManager, PAGameManagerBase, PAGameUIController,
  PASoundManager, TweenLite, Oriented fields, PA* effects/controls/video, and its Tools/PlayableKit editor tools
  (Control Center, Sanitize UI For Luna, Luna Compliance, Playable Ads Checker, PALunaBuildPatcher). Use when a
  project has the package or a vendored Assets/**/PlayableKit copy. Sub-skill of unity-luna-playworks.
---

# PlayableKit (`com.kat.playablekit`)

Sub-skill of [`unity-luna-playworks`](../SKILL.md). The parent skill holds the Luna rules; this one says how the kit
already solves them. Kit rules never override Luna rules: a kit component still has to pass a Luna build.
Links to `../` point into the parent skill folder; if this sub-skill is installed on its own (symlink/junction),
load the `unity-luna-playworks` skill instead of following them.

Repo: `https://github.com/kat1002/PlayableKit` (source of truth: `Packages/com.kat.playablekit/`, v2.0.0).
Needs Unity 2021.3+ (tested 2022.3 and Unity 6). Luna 6.4 / Playworks 7.2 optional: without Luna the kit compiles,
lifecycle/analytics calls are no-ops and `InstallFullGame` only logs.

---

## 1. Detect which kit layout the project has

| Signal | Layout | Do |
|---|---|---|
| `"com.kat.playablekit"` in `Packages/manifest.json` | **2.0 package** | Use this skill as written |
| `Assets/PlayableKit/` (StickiiPA, CraftPlayable), `Assets/_Template/PlayableKit` + `PlayableUIKit` (GangterCity), `Assets/Models/PlayableKit` (Obby), vendored copies in mukbang/scoop/squishy | **1.x vendored** | Old names (`LunaManager`, `UIShine`, `DoScaleAToB`, `Tools/PA`, `Tools/MyLunaHelper`); map with [migration.md](./references/migration.md) |
| Both | **Broken**: `CS0433` ambiguous types, duplicate GUIDs | Delete the vendored copy, keep the package |

Never assume a menu or type exists: grep the project for the class / `MenuItem(` string first.

---

## 2. Install / migrate

1. **Vendored copy present?** Delete it first (GUIDs are identical, so scene references survive the switch).
2. Install one way:
   - Package Manager → Add package from git URL:
     `https://github.com/kat1002/PlayableKit.git?path=Packages/com.kat.playablekit#v2.0.0`
   - or `manifest.json` `dependencies`: `"com.kat.playablekit": "<same URL>"`
   - or import `PlayableKit.unitypackage` from the repo root (lands in `Packages/com.kat.playablekit`).
3. Add `using PlayableKit.<Module>;` lines to own scripts; rename 1.x types ([migration.md](./references/migration.md)).
4. No TMP font yet: **Window > TextMeshPro > Import TMP Essential Resources**.
5. Luna defines: asmdefs define `PLAYABLEKIT_LUNA` when `com.unity.playworks.upp` / `uk.lunalabs.luna` is installed;
   Luna builds define `UNITY_LUNA`. Either enables real `Luna.Unity` calls.

---

## 3. Start a new PA from the kit

1. Package Manager → PlayableKit → Samples → **Template Scene** → Import; open `PATemplateScene`.
   Optional **Luna Config** sample: copy its `luna.json` to the project root, fill `projectId`, `applicationID`,
   `environmentID`, `creativeName`.
2. Scene objects (each on its own empty GameObject):

   | Object | Component | Notes |
   |---|---|---|
   | `PALunaRoot` (prefab) | `PALunaManager` + CTA UI | **Exactly one per scene**: lifecycle, store, mute sync |
   | `GameManager` | your `PAGameManagerBase` subclass | Assign **Ui** |
   | `UI` (on Canvas) | `PAGameUIController` or subclass | Fill `Portrait Core` / `Landscape Core` |
   | `AudioManager` | `PASoundManager` + `PASoundConfig` asset | *Create > PlayableKit > Sound Config* |
   | `PAConfig` (optional) | `PAConfig` | Log level/time/enable; execution order −1000 |
   | `EventSystem` | EventSystem + input module | In the template |

3. Subclass the flow:
   ```csharp
   using PlayableKit;
   using PlayableKit.Flow;

   public sealed class MyGame : PAGameManagerBase
   {
       protected override void OnBeginPlay() { /* first tap */ }
       protected override void OnPlayingTick() { /* every frame while playing */ }
       protected override void OnEndGameBegin(EndReason reason) { /* stop loops */ }
   }
   ```
4. Wire the CTA button to `PAGameUIController.Cta()` (or a full-screen catcher with `PACtaClickForwarder`).
5. Then follow the parent skill's shipping checklist (naming, `luna.json`, build, test, deliverable).

API details: [modules-and-api.md](./references/modules-and-api.md).

---

## 4. Kit answers to Luna rules

| Luna rule (parent skill) | Kit way |
|---|---|
| Lifecycle: `GameStarted` once, `GameEnded` once, sync store, 0.25 s guard, clamp dt | `PALunaManager` (`GoToStore()`, `MaxTickSeconds` 0.25); outside a controller `PALunaManager.Instance.GoToStore()` or a `PALunaStoreGateway` Button target |
| Pause without `AudioListener.pause` | `PALunaManager` lowers `AudioListener.volume`; mute shared with `PASoundManager` / `PAMuteToggleButton` |
| No generic `[Serializable]` | `OrientedGameObject` / `OrientedButton` / `OrientedText` (serializable); `Oriented<T>` runtime-only |
| Portrait + landscape | `Orientation.IsLandscape` (single definition); `PABaseUIController.OnOrientationChanged`; `PALayout` (per-element presets) or `PAOrientationLayout` (central slot list); `PASafeArea` |
| No LayoutGroup / fitters / ScrollRect | `Tools/PlayableKit/Sanitize UI For Luna` → bakes rects, `AspectRatioFitter`→`PAAspectFitter`, `ScrollRect`→`PADragScroll` |
| No DOTween Sequence | `TweenLite` (`PlayableKit.Tweening`, DOTween-style fluent API, no external lib); components `PADo*AToB`, `PAUiSlide`, `PATweenButton` |
| Audio import | `Tools/PlayableKit/Audio Import Optimizer` (still set **Decompress On Load**) |
| Sliced-image sweep shader | `PAUIShine` + `PA_UI_Shine` (9-slice uniforms `_ShineSliceUV`, `_ShineSliceDrawn`) |
| Fake shadows, reveals, blur | Shaders `PA_FakeShadow`, `PA_UI_CircleReveal`, `PA_UI_BackdropBlur`, `PA_UI_GlowOutline` |
| Video `Play()` retry, muted autoplay, loop | `PAVideoBootstrap`, `PAVideoStepController`, `PAVideoLoopSequenceController`, `PAAutoPlayVideoOnEnable` + `PALunaBuildPatcher` loop patch |
| physics3d mesh-collider crash, video loop | `PALunaBuildPatcher` (automatic, idempotent) |
| Video size | `Tools/PlayableKit/Video/Video Optimizer` (H.264 Baseline, yuv420p, no B-frames, size budget; needs FFmpeg) |
| Logging that also shows in the browser | `PALog` (`[PA][Class] t | msg | state`); `PA_LOG_OFF` define silences all |
| Pre-build scan | Control Center (`Ctrl+Shift+K`): checker, compliance, Spine inspector, sanitizer |

Kit-specific rules:
- On `PABaseUIController` subclasses **never** declare `Awake`/`OnEnable`/`OnDisable`/`Update`: override
  `OnControllerAwake` / `OnControllerEnable` / `OnControllerDisable` / `OnControllerUpdate` / `OnOrientationChanged`.
- `PA_ConfettiBurst` and other particles: world space or Screen Space – Camera canvas; trigger with `Play()`.
- The kit's Luna Compliance assumes Built-in-pipeline shaders; in URP projects its URP warnings need judgement
  (parent skill [constraints §5.4](../references/constraints-and-gotchas.md#54-urp-projects)).
- If DOTween is also installed, never `using DG.Tweening;` and `using PlayableKit.Tweening;` in the same file; alias one.

---

## 5. Pre-build order (kit projects)

1. `Tools/PlayableKit/Sanitize UI For Luna` (undoable, never saves, safe to run twice).
2. Save the scene and add it to Build Settings, then `Tools/PlayableKit/Luna Compliance` (Errors otherwise).
3. `Tools/PlayableKit/Playable Ads Checker`: Missing Scripts 0, Obj Missing 0, no code hits in runtime scripts.
4. Optional `Tools/PlayableKit/Audio Import Optimizer`.
5. Window > Unity Playworks → build. Confirm the Console shows `PALunaBuildPatcher` patched files; force with
   `Tools/PlayableKit/Luna/Apply Build Patches Now`.

Tool details: [tools-and-build-patcher.md](./references/tools-and-build-patcher.md).

## References

| File | Read when |
|---|---|
| [modules-and-api.md](./references/modules-and-api.md) | Using kit runtime types: modules, flow hooks, UI controllers, sound, logging, tunables, assets |
| [tools-and-build-patcher.md](./references/tools-and-build-patcher.md) | Running/explaining kit editor tools, build patcher, video tools |
| [migration.md](./references/migration.md) | Vendored 1.x → 2.0 package, old project names → kit names |
