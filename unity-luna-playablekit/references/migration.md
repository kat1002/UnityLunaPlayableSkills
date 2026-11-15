# Migrating to PlayableKit 2.0

Namespaces changed, **script GUIDs did not**: scenes and prefabs keep their references after the switch.

## Steps

1. Delete the vendored copy (`Assets/PlayableKit`, `Assets/_Template/PlayableKit` + `PlayableUIKit`,
   `Assets/Models/PlayableKit`, …). Keeping both → `CS0433` ambiguous types + GUID clashes.
2. Install the package ([SKILL.md §2](../SKILL.md#2-install--migrate)).
3. Fix compile errors with the tables below; add `using PlayableKit.<Module>;`.
4. Remove per-project duplicates the kit replaces (physics/video patchers, checker copies).
5. Full Luna build + browser test (parent skill shipping checklist).

## 1.x kit → 2.0

| Old | New |
|---|---|
| `Assets/PlayableKit/**` | `Packages/com.kat.playablekit/**` |
| `PlayableKit.UI.PAUIControllerBase` | `PlayableKit.Flow.PAGameUIController` (derives from `PABaseUIController`; override `OnControllerAwake` / `OnControllerUpdate` / `OnOrientationChanged`, not `Awake`/`Update`) |
| `PlayableKit.UI.PACtaClickForwarder` | `PlayableKit.Flow.PACtaClickForwarder` |
| `PlayableKit.PAGameManagerBase` | `PlayableKit.Flow.PAGameManagerBase` |
| `PlayableKit.PAAudioManagerBase` | `PlayableKit.Audio.PAAudioManagerBase` |
| `PACameraShake`, `PAFreezeSpineRigAfterWarmup`, all `PAUI*` effects, `PAFallWindFx`, `PAFlashTrailRendererDriver`, `PAScratchSpriteRenderer`, `IWindFxReceiver`, `PAUITapHand`, `PABillboardToCamera` | `PlayableKit.Effects.*` |
| `PADoFadeAToB`, `PADoMoveAToB`, `PADoScaleAToB`, `PALunaTweenAnimation`, `PATweenButton`, `PATweenRotateButton`, `PAUiSlide`, `PAUiPageSlide` | `PlayableKit.Tweening.*` |
| `PAHoldButton`, `PADragScroll`, `PAUiDragRotate`, `IDragRotateTarget`, `IMoveInput` | `PlayableKit.Controls.*` |
| `PlayableKit.PlayableLog` | `PlayableKit.PALog` (`[LunaLog]` prefix → `[PA]`) |
| `PlayableKit.LunaAnalyticsSink` | `PlayableKit.Analytics.LunaAnalyticsSink` |
| `IOrientationSource`, `PAGameEventOrientationSource` | Removed: override `PABaseUIController.OnOrientationChanged` |
| `IStoreGateway`, `PABaseUIController.Store` | Removed: protected `GoToStore()` or `PALunaStoreGateway` |
| `PABaseUIController.Cta` (CtaPresenter property) | `PABaseUIController.CtaPanel`; `Cta()` is now the CTA button method on `PAGameUIController` |
| `ISteerInput` | Removed (use `IMoveInput`) |
| Menus `Tools/PA`, `Tools/Sp Unity`, `Tools/MyLunaHelper` | `Tools/PlayableKit/...` |
| `Assets/Reference/*` | Removed: `PASoundManager` + `Samples~/LunaConfig` |
| `TweenLite` in `Playable.Tweening` (older copies, e.g. mukbang) | `PlayableKit.Tweening.TweenLite` |

## Other projects' names → kit

| Old name (project-local) | Kit |
|---|---|
| `LunaManager` | `PALunaManager` (check behaviour: some old managers delay the store or end the game on CTA) |
| `ScreenOrientationChecker` | `PAScreenOrientationChecker` |
| `UiManager` | `PACtaOverlay` |
| `SoundEnifi`, `SoundManager`, Obby `AudioManager` | `PASoundManager` + `PASoundConfig` |
| `LunaStoreGateway` | `PALunaStoreGateway` |
| `BaseUIController` | `PABaseUIController` |
| `UIShine` | `PAUIShine` (needs `PA_UI_Shine_M`) |
| `DoScaleAToB` / `DoMoveAToB` / `DoFadeAToB` | `PADo*AToB` |
| `TweenButton` / `TweenRotateButton` | `PATweenButton` / `PATweenRotateButton` |
| `KeepRatio` | `PAKeepRatio` |
| `AutoPlayVideoOnEnable` | `PAAutoPlayVideoOnEnable` |
| `VideoLoopSequenceController` | `PAVideoLoopSequenceController` |
| `PlayworksVideoBootstrap` | `PAVideoBootstrap` |
| `PlayableVideoStepController` | `PAVideoStepController` |
| `VideoPlayOrientationSizer` | `PAVideoOrientationSizer` |
| `PlayableSpiderLunaPhysicsPatch`, `CodexLunaStage4AutoPatch` | `PALunaBuildPatcher` |
| `PlayableAdsCheckerWindow` | `PADependencyScannerPanel` |
| `SimpleJoystick` | `PAJoystick` |
| `CoinMoveFX` | `PAUICoinFly` |
| `ScratchSpriteRenderer` (DOTween) | `PAScratchSpriteRenderer` |
| `LunaFakeShadow.shader` | `PA_FakeShadow` |
| `Craft01CircleReveal.shader` | `PA_UI_CircleReveal` |
| `ResponsiveVideoBackdropBlur.shader` | `PA_UI_BackdropBlur` |
| `PA01_UIGlowOutline` | `PA_UI_GlowOutline` |

Not adopted into the kit (stay project-local): BikePlayable `LunaPreflightValidator`, `AudioManagerBase`
(PlayerPrefs + try/catch); GangterCity `LunaMapStandard*` shaders, `LunaTools`; StickiiPA `PAShoreFieldGenerator`,
`PA01_UIHole*`; SatisFix/scoop/squishy/mukbang `SoundManager` / `SoundConfig`; Fake-Video orientation setup window.
