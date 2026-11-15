# PlayableKit 2.0 Modules & Runtime API

Source: package `README.md`, `Documentation~/Setup.md`, `Documentation~/UIKit.md` (v2.0.0).

## 1. Modules

One assembly per module, no cycles. Core keeps namespace `PlayableKit`; others use `PlayableKit.<Module>`.
Dependency graph: Core and Controls are leaves; Tweening, Audio, UI sit on Core; Effects on Core + Tweening; Video
on Core + UI; Flow on top of everything.

| Folder | Namespace | Contents |
|---|---|---|
| `Runtime/Core` | `PlayableKit` | `PALunaManager`, `PAStoreReturn`, `PALunaStoreGateway`, `GameEventManager`, `GameSessionState`, `GameState`, `EndReason`, `Orientation`, `PALog` / `PALogLevel`, `PAConfig`; `Analytics/` (`PlayableKit.Analytics`: `PlayableAnalytics`, `IAnalyticsSink`, `LunaAnalyticsSink`, `PAAnalyticsButton`) |
| `Runtime/Tweening` | `PlayableKit.Tweening` | `TweenLite` (DOTween-style fluent API); components `PADoFadeAToB`, `PADoMoveAToB`, `PADoScaleAToB`, `PALunaTweenAnimation`, `PATweenButton`, `PATweenRotateButton`, `PAUiSlide`, `PAUiPageSlide` |
| `Runtime/Audio` | `PlayableKit.Audio` | `PAAudioManagerBase` (loop crossfade, `PlayTap`, one-shots), `PASoundConfig`, `PASoundManager` |
| `Runtime/Effects` | `PlayableKit.Effects` | `PAUIShine`, `PAUIStarBurst`, `PAUIStarTwinkle`, `PAUITapRing`, `PAUIPixelRingPop`, `PAUIPixelSmokeBurst`, `PAUIPraiseFly`, `PAUIProgressFillFX`, `PAUIIdleMotion`, `PAUIPathTwinkle`, `PAUITweenRotate`, `PAUITapHand`, `PAUICoinFly`, `PAFallWindFx`, `PAFlashTrailRendererDriver`, `PAScratchSpriteRenderer`, `PACameraShake`, `PAFreezeSpineRigAfterWarmup`, `PABillboardToCamera` |
| `Runtime/Controls` | `PlayableKit.Controls` | `IMoveInput`, `IDragRotateTarget`, `PAHoldButton`, `PADragScroll` (`ScrollTo(float, bool)`, `IsMoving`), `PAUiDragRotate`, `PAJoystick` |
| `Runtime/UI` | `PlayableKit.UI` | `PABaseUIController`, `PACtaOverlay`, `CtaPresenter`, `CountdownTimer`, `IdleTimer`, `Oriented<T>` / `OrientedFields` / `OrientedExtensions`, `PALayout`, `PAOrientationLayout`, `PAAspectFitter`, `PASafeArea`, `PAPortraitFrame`, `PAScreenOrientationChecker`, `PALoadingScreenController`, `PAMuteToggleButton`, `PATextOutline` |
| `Runtime/Video` | `PlayableKit.Video` | `PAVideoBootstrap`, `PAVideoStepController`, `PAVideoLoopSequenceController`, `PAVideoOrientationSizer`, `PAAutoPlayVideoOnEnable`, `PAKeepRatio`, `VideoPlayerLoopUtility` |
| `Runtime/Flow` | `PlayableKit.Flow` | `PAGameManagerBase`, `PAGameUIController`, `PACtaClickForwarder` |
| `Editor` | `PlayableKit.Editor` (+ `.Editor.Media`) | Tools, FFmpeg locator/process/probe |
| `Tests/Editor` | `PlayableKit.Tests.Editor` | `GameSessionStateTests`, `LunaManagerSessionTests`, `PAVideoEncodePlanTests` |

## 2. Flow: `PAGameManagerBase`

| Hook | Fires |
|---|---|
| `OnAwake()` | End of `Awake()`; wire scene refs |
| `OnBeginPlay()` | First tap |
| `OnPlayingTick()` | Every frame while playing |
| `OnMidpointReached()` | Once, when `PlayElapsed` crosses `sessionDuration` |
| `OnEndGameBegin(reason)` | Immediately on any end |
| `OnBeforeCrashSequence(reason)` | Once, non-timeout ends, before crash slow-mo |
| `OnEndGameAfter(reason)` | Once the end card shows |

Wiring: **Ui** = the `PAGameUIController`; optional **Camera Shake** (`PACameraShake`).

## 3. UI: `PABaseUIController` / `PAGameUIController`

`PABaseUIController` reads `Orientation.IsLandscape` in Awake, polls it every frame, and calls
`OnOrientationChanged(UIOrientation)` on a flip (no checker or adapter needed).

| Hook | When |
|---|---|
| `OnControllerAwake` | Once, after `CurrentOrientation` is set |
| `OnControllerEnable` / `OnControllerDisable` | OnEnable / OnDisable |
| `OnControllerUpdate` | Every frame |
| `OnOrientationChanged(UIOrientation)` | Every landscape/portrait flip |

```csharp
using PlayableKit.UI;
using UnityEngine;

public sealed class MyController : PABaseUIController
{
    [SerializeField] private OrientedGameObject myPanel;
    [SerializeField] private OrientedGameObject ctaPanel;
    [SerializeField] private GameObject fullscreenCta;

    protected override void OnControllerAwake() { CtaPanel = new CtaPresenter(ctaPanel, fullscreenCta); }
    protected override void OnControllerEnable() { SetActive(myPanel, true); }   // both Hor and Ver
    protected override void OnOrientationChanged(UIOrientation o) { /* reset held input, re-layout */ }
    public void OnCtaClicked() { GoToStore(); }   // protected static, null-safe into PALunaManager
}
```

Rules: never declare your own `Awake`/`OnEnable`/`OnDisable`/`Update` on a subclass; `GoToStore()` is
`protected static` (elsewhere: `PALunaManager.Instance.GoToStore()` or a `PALunaStoreGateway` Button target);
Unity null checks `== null`.

`PAGameUIController` (Flow) drives tap zones, tutorial hand, progress bar and end card:

| Member | Purpose |
|---|---|
| `ShowIntro()` / `HideIntro()` / `OnFirstTap()` | Intro hint, hand, pulses |
| `SetProgress(float)` | Progress fill 0..1 |
| `ShowEndCard()` | End card (tracks `endcard_shown` once) |
| `Cta()` | Tap sound + `GoToStore()`; wire to the CTA button |
| `BuildCoreRefs()` | Override to copy a richer ref class into `portraitCore` / `landscapeCore` |
| `OnHudTick()` | Throttled by `hudTickInterval` |

`Portrait Core` / `Landscape Core`: root, hand, hint text, start object, progress fill, end card, intro pulses.
Optional `Loading Screen` (`PALoadingScreenController`).

| `UIKit` type | Use |
|---|---|
| `Oriented<T>` / `UIOrientation` | Element authored for `Hor` and `Ver`; **runtime-only** |
| `OrientedGameObject` / `OrientedButton` / `OrientedText` | Serializable versions; convert implicitly to `Oriented<T>` |
| `OrientedExtensions` | `SetActive`, `SetText`, `GetActiveInHierarchy` |
| `CtaPresenter` | Show/hide/retarget CTA pair + optional fullscreen CTA |
| `CountdownTimer` / `IdleTimer` | Countdown-to-zero / count-up-with-reset |
| `PALayout` vs `PAOrientationLayout` | Per-element presets (few elements) vs central slot list + CanvasScaler ref resolution (whole screens). One per element |
| `IMoveInput` | 2D `Value` + `IsPressed`; `PAJoystick` implements it; implement for own steering/tilt |

## 4. Sound

1. *Create > PlayableKit > Sound Config*: BGM clip/volume, SFX channel count, SFX entries (id, clip, volume, min
   interval, pitch jitter, start offset).
2. `PASoundManager` on AudioManager, assign config. Play: `PASoundManager.Play("id")` or Button `PlaySfx(string)`.
3. Mute shared with `PALunaManager`; `PAMuteToggleButton` toggles it. Clips still need **Decompress On Load**.

## 5. Logging

```csharp
PALog.Log("MyGame", "picked item", "score=3");
PALog.Info("MyGame", "Level started");
PALog.Warn("MyGame", "No target", this);     // clickable context
PALog.Error("MyGame", "Missing ref", this);
PALog.Verbose("MyGame", "detail");           // Editor only by default
PALog.Click("PlayButton"); PALog.Cta("EndCard"); PALog.Tap(Input.mousePosition);
```

| Setting | Default | Override |
|---|---|---|
| `PALog.Enabled` | true | `PAConfig` > Enable Game Log |
| `PALog.MinLevel` | Verbose (Editor), Info (builds) | `PAConfig` > Min Log Level |
| `PALog.ShowTime` | true | `PAConfig` > Show Log Time |
| `PA_LOG_OFF` define | unset | Silences all logs at compile time |

## 6. Tunables

`sessionDuration`, `introInactivityStoreDelay` (15 s), `endCardDelay` (0.2 s), `crashShakeAmplitude`,
`crashShakeDuration` (`PAGameManagerBase`); `safetyTimeout` (`PALunaManager`); `hudTickInterval`
(`PAGameUIController`); `loopVolume`, `loopPitchRange`, `sfxVolume` (`PAAudioManagerBase`).
Company idle rule (store 5–7 s after the last touch) is not a kit default: tune for it.

## 7. Assets

| Type | Items |
|---|---|
| Shaders (`PlayableKit/…`) | `PA_ChromaKeyGreenVideo`, `PA_PlayworksUnlitTexture`, `PA_SpriteOutline`, `PA_UI_ProgressFill`, `PA_UI_Shine`, `PA_UI_CircleReveal`, `PA_UI_BackdropBlur`, `PA_UI_GlowOutline`, `PA_FakeShadow` |
| Materials | `PA_UI_ProgressFill_M`, `PA_UI_Shine_M` (required by `PAUIShine`), `PA_Confetti_M` |
| Prefabs | `PALunaRoot`, `PACtaButton`, `PA_ConfettiBurst` (ParticleSystem; call `Play()`) |
| Sprites | `PA_Light`, `PA_Ring`, `PA_Star`, `PA_Star2`, `PA_TapHand_1`, `PA_TapHand_2` |
| Samples | `Template Scene` (`PATemplateScene`), `Luna Config` (`luna.json`) |
