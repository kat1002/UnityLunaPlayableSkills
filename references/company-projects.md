# Company PA Projects, Tools & Shared Assets

Facts checked against the repos on 2026-11-14 (PA Handbook). Each project is its own git repo. Paths below are
relative to the projects root (`D:/Phuc/Projects/` on the reference machine). The Luna package is a machine-local
`file:` path in every project: re-point it after cloning. Full per-project notes: the team's **PA Handbook**
(`_PA_Handbook/projects/<Project>.md`, generic rules in `02_Luna_Rules_And_Pitfalls.md`).

## Which project do I open?

- Same genre/mechanic as an existing PA → that project; copy its closest scene.
- New mechanic, clean base → PlayableKit `Template Scene` (`unity-luna-playablekit` sub-skill).
- Video-driven fake gameplay → Playable-Ads-Fake-Video or VideoPlayer.

## Projects

| Project | Unity root | Unity | Pipeline | Luna | What | Kit layout |
|---|---|---|---|---|---|---|
| BikePlayable | `BikePlayable/` | 2022.3.62f3 | Built-in | 7.2.0 | Bike runner from primitives | own `_Template` (DOTween) |
| CraftPlayable | `CraftPlayable/` | 6000.0.79f1 | URP 17.0.4 | 7.2.0 | Craft/voxel (`Craft_PA_04`) | vendored PlayableKit 1.x |
| GangterCity | `GangterCity/` | 2022.3.62f2 | Built-in (its CLAUDE.md wrongly says URP 14) | 7.2.0 | Rope/vehicle/shooter (`Rope_PA_06`) | vendored 1.x (`_Template/`) |
| Obby-Playable-Ads | `Obby-Playable-Ads/` | 2022.3.62f3 | URP 14.0.12 | 7.2.0 | Obby (`Obby_PA_02`) | vendored 1.x (`Assets/Models/`) |
| Playable-Ads-Fake-Video | `Playable-Ads-Fake-Video/` | 2022.3.62f3 | Built-in | 7.2.0 | Fake gameplay with video | older renamed copies |
| StickiiPA | `StickiiPA/` | 6000.0.79f1 | URP 17.0.4 | 7.2.0 | Stickii (`Stickii_PA_04`) | vendored 1.x (newest fixes) |
| PlayableKit | `PlayableKit/` | 2022.3.25f1 | Built-in | 7.2.0 | Toolkit package + template (not an ad) | package 2.0.0 source |
| VideoPlayer | `VideoPlayer/` | 2022.3.25f1 | Built-in | 7.2.0 | Video loop / phone fake | none |
| Home-Clean-Playable | `Home-Clean-Playable/SatisFix/` | 2022.3.25f1 | URP 17.2.0 | 7.2.0 | Home-cleaning ASMR (deliverables in `GoogleAds/`) | none |
| perfectcleaning_pa_unity-lunar_sdk | `…/SatisFix/` | 2022.3.25f1 | URP 14.0.11 | 7.2.0 | Older SatisFix fork (no `.git`) | none |
| mukbang_pa | `mukbang_pa/IdolMukbang/` | 2022.3.62f2 | URP package, no pipeline asset set | 7.2.0 | Mukbang / supermarket / blind box | vendored copy |
| scoop_pa | `scoop_pa/ScoopAndPack/` | 2022.3.62f2 (docs saying 6000.2.7f2 are stale) | Built-in | **6.4.0** (`uk.lunalabs.luna`) | Scoop & Pack, croc DIY | vendored copy |
| squishy_pa | `squishy_pa/SquishyDumpling/` | 2022.3.62f2 | Built-in | 7.2.0 | Squishy dumpling / scoop mukbang | vendored copy |

Known config debt: `applicationID` is `0` in VideoPlayer and perfectcleaning; mukbang/scoop/squishy share one
`projectId`, Home-Clean/perfectcleaning share another. `creativeName` often doesn't match the scene. Fix these when
you start a new creative there.

## Editor tools by project

Tools are **not** in every project. Grep for the class / `MenuItem(` string before telling someone to run a menu.

| Tool | Menu | Where |
|---|---|---|
| Playable Ads Checker (`PlayableAdsCheckerWindow`) | `Tools/Sp Unity/Playable Ads Checker` | Home-Clean, perfectcleaning (`Assets/Editor/`), mukbang (`MyLunaUtils/Editor/`), `Package/` copy |
| Luna Compliance (`MyLunaComplianceWindow`), Spine Binary Inspector, Find Asset By ID, Force Remove All AssetBundles | `Tools/MyLunaHelper/...` | Home-Clean, perfectcleaning, mukbang, scoop, StickiiPA |
| Sanitize UI For Luna (`PALunaUiSanitizer`, legacy) | `Tools/PA/Sanitize UI For Luna` | mukbang, StickiiPA |
| PlayableKit tools (Control Center, sanitizer, checker, compliance, build patcher…) | `Tools/PlayableKit/...` | PlayableKit package, StickiiPA: see sub-skill |
| `PAShoreFieldGenerator` (hole distance field, alpha-254 PNG) | `Tools/PlayableKit/Shore Field Generator` | StickiiPA only |
| `LunaPreflightValidator` (scene readiness, refs, Luna config) | `BikePlayable/Preflight Validation` | BikePlayable |
| Prepare for Luna (pin `Unlit/Color`, build settings), `RoadSegmentBaker` | `Tools/Playable/Prepare For Luna (shader + build settings)` | BikePlayable |
| `LunaTools` (reflection bridge into `EditorScripts.dll`, re-read / force-save `luna.json`) | `Tools/Luna/Refresh From Disk`, `Apply To luna.json`, `Diagnose Bridge` | GangterCity `Assets/Editor/LunaTools/` |
| `PlayableSpiderLunaPhysicsPatch` (physics3d mesh inertia, auto every 5 s) | automatic | GangterCity, CraftPlayable |
| `CodexLunaStage4AutoPatch` (video loop) | automatic | VideoPlayer |
| Build Selector / profiles (`PlayableProfiles.json`, forces `luna.json` reload) | project menu | squishy (`Tools/Playworks/README.md`) |
| `NoRuntimeResourceLoadTests` (EditMode test) | Test Runner | mukbang |
| Video optimizer / backdrop capture / orientation setup (FFmpeg) | project windows | Playable-Ads-Fake-Video |

What the Playable Ads Checker flags (rules worth grepping for by hand): `PlayerPrefs.`, `Resources.Load`,
`RuntimeInitializeOnLoadMethod`, `void OnMouseDown`, `OdinInspector`, `using Firebase`, `using com.adjust`,
`UniTask`, `JsonUtility.`, `goto `, `= new();`, `[^n]`, `.FindAll(`, `is … or/and`, `is not`, `GameObject.Find`,
`FindObjectOfType`, lambda `.Find(`, `Unity.VisualScripting`, `Array.Empty<`, `AsyncWaitForCompletion`; plus
missing scripts, null Sprite/Mesh/Material/Spine refs, AssetBundle names, heavy textures/audio. Press "Quét runtime
dependencies" first (collects only runtime-used assets).

Luna Compliance flags: unsaved scene / scene not in Build Settings, Spine `.skel.bytes`, shader names containing
URP/Lit/Shader Graph, DOTween Pro folders, "dangerous API" (`GetComponent("string")`, `OverlapCollider`,
`System.Reflection`).

## Per-project traps worth knowing

| Project | Trap |
|---|---|
| BikePlayable | Manager objects (`World`, `Player`, `Traffic`, `GameManager`) must sit at origin; `LunaManager.safetyTimeout` 45 s is the only hard cap (keep `surviveDuration` 25 s below it); template needs DOTween at `Assets/Plugins/Demigiant/DOTween/` |
| CraftPlayable | Use `Craft04Input` poller for drag/joystick; `Craft01ScrollGrid` instead of ScrollRect + GridLayout |
| GangterCity | Animator strings via central `Anim` / `AnimHash` wrapper (`Anim.Trigger("Player", AnimHash.Shoot)`); ragdoll bones `Interpolate None` + `Discrete`; never PolyFew BatchFew |
| Obby | `LunaManager.SetEndTime` ignored after `StartGame` (call it first); store via debounced `StoreOpener.TryOpen`; `SoundManagerRP` uses `PlayerPrefs` (`playerPrefs: true`): don't copy |
| Fake-Video | Keep the `Play()` retry loop; video audio via separate `AudioSource`; don't rename `Main Video` / `Backdrop` / `Action` (looked up by name); mid-video store open must not latch `storeOpened` or send `GameEnded` |
| Home-Clean / perfectcleaning | `s_KeepUrpAssembly` in `LunaManager`; `[Preserve]` on level-select handlers; start level transition before video teardown; scratch textures need Read/Write; `ScratchObject` needs `m_ListScratchSprite` |
| mukbang | One Spine runtime; tray textures standalone, uncompressed; never call `SpriteAtlasUtility.PackAtlases` from scripts/MCP (native crash on 2022.3.62); animation `disgurt` is spelled that way in the skeleton |
| scoop | PA scenes reach main-game services: keep `#if UNITY_LUNA` early returns in `ReturnToGameScene()`; only `PA_GiftBoxMachine` has `LunaManager` wired; repo tracks `LunaTemp` (don't add more) |
| squishy | DOTween.js manifest + TMP.js manifest fixes live in the vendored SDK; Build Selector resets `creativeName` to the profile name: set your dated name **after** Prepare |
| VideoPlayer | Its old `LunaManager` is an anti-pattern (GameEnded at start, delayed store); `Codex*` scripts have hardcoded machine paths |

## Shared assets (`Package/` folder)

| Item | Use |
|---|---|
| `Notes_PlayableAds.xlsx` | Team's Vietnamese "Các lưu ý khi làm" cheat sheet (copies in several `Doc/` folders); rules merged into this skill |
| `PlayableAdsCheckerWindow.cs` | Drop-in checker (above) |
| `RenameTool.html` | Renames per-network deliverables (`GGA_`, `Minter_`, `FB_`, `Tiktok_`, `Applovin_`) from a zip with one folder per network |
| `Shader3d/LunaMapStandard{Opaque,Fade,Transparent}.shader` | Standard-lookalike shaders that survive Luna (heavy, not kit-verified) |
| Green-screen video folder | `ChromaKeyGreenVideo.shader`, `PlayworksUnlitTexture.shader`, `PlayworksVideoBootstrap.cs` (muted play-when-ready with retry) |
| `FakeSkyBox/` | Cheap fake sky mesh + texture |
| Third-party `.unitypackage`s | DOTween Pro 1.0.310 (team-recommended version; Pro folders flagged by compliance), Odin (editor only), UIParticleSystem (avoid), others unverified |

## PA_Assets_Extractor

Node 22+ tool (zero deps) that pulls images/audio out of competitor playable HTML (Luna, Cocos, Pixi/Phaser,
PlayCanvas) for **reference research only: never ship extracted assets**.

```bash
npm start                                   # UI at http://127.0.0.1:5178
node extract.js ad.html                     # → out/ad/{audio,textures,other,manifest.json}
node extract.js https://cdn.example.com/ad.html --out out/ad --verbose
```

Options: `--out`, `--no-fetch`, `--audio-only` / `--textures-only`, `--depth N`, `--timeout MS`. Runtime-computed
URLs, encrypted payloads and expiring CDN URLs are not extracted.
