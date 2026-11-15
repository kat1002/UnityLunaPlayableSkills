# PlayableKit Editor Tools & `PALunaBuildPatcher`

All menus live under `Tools/PlayableKit/` in the 2.0 package. 1.x vendored copies use `Tools/PA/...`,
`Tools/Sp Unity/...`, `Tools/MyLunaHelper/...` ([migration.md](./migration.md)).

| Menu | What it does |
|---|---|
| `PlayableKit Control Center` (Ctrl+Shift+K) | Hub for the tools below |
| `Sanitize UI For Luna` (`PALunaUiSanitizer`) | Temporarily activates inactive branches, forces a canvas rebuild, bakes child rects, destroys `LayoutGroup` / `ContentSizeFitter`, swaps `AspectRatioFitter`→`PAAspectFitter`, `ScrollRect`→`PADragScroll`. Undoable, never saves, safe twice |
| `Playable Ads Checker` (`PADependencyScannerPanel`) | Missing scripts, who-references-what, null Sprite/Mesh/Material/Spine data, AssetBundles, textures, audio, scripts used in scene/prefab/SO, code rules. Press the runtime-dependency scan first |
| `Luna Compliance` (`PALunaCompliancePanel`) | Unsaved scene (Error), scene not in Build Settings (Warning), Spine `.skel.bytes`, shader names with URP/Lit/Shader Graph, DOTween Pro folders, dangerous API (`GetComponent("string")`, `OverlapCollider`, `System.Reflection`), URP shader assets in materials |
| `Spine Binary Inspector` | Finds binary Spine skeletons |
| `Audio Import Optimizer` (`PAAudioImportOptimizer`) | WebGL SFX / Music import presets on selected clips (`ApplyPreset(assetPath, sfx)` public). Whether Luna honours Unity's WebGL audio overrides is unverified |
| `Video/Video Optimizer` | Batch H.264 Baseline, yuv420p, no B-frames, two-pass under a size budget, keeps aspect without bars, never upscales or duplicates frames, optional MP3, ffprobe verification, auto re-encode when over budget, imports results |
| `Video/Backdrop Capture` | Bakes one video frame into a cover-cropped, blurred (optionally darkened) JPG sprite for behind letterboxed video |
| `Luna/Apply Build Patches Now` | Forces `PALunaBuildPatcher`; Console reports how many files were patched |
| `Export PlayableKit.unitypackage` | Exports the package folder (`Samples~` excluded) |

Code rules the checker flags: `PlayerPrefs.`, `Resources.Load`, `RuntimeInitializeOnLoadMethod`, `void OnMouseDown`,
`OdinInspector`, `using Firebase`, `using com.adjust`, `UniTask`, `JsonUtility.`, `goto `, `= new();`, `[^n]`,
`.FindAll(`, `is … or/and`, `is not`, `GameObject.Find`, `FindObjectOfType`, lambda `.Find(`,
`Unity.VisualScripting`, `Array.Empty<`, `AsyncWaitForCompletion`. Skips `Packages/`, `/Editor/`, DOTween, Spine,
Luna, TextMesh Pro.

Video tools need **FFmpeg + FFprobe** (auto-detected from PATH, winget, scoop, choco, Homebrew; saved path kept).

## `PALunaBuildPatcher`

`[InitializeOnLoad]` Editor script that watches `LunaTemp/` every few seconds (never during Play Mode or
compilation) and string-patches Luna 7.2's generated JS whenever Luna rewrites it:

- **`physics3d-0.js`**: mesh-collider inertia crash (`Not implemented for the mesh shape`) → box-approximation
  inertia from mesh bounds.
- **`UnityEngine.js`**: `VideoPlayer.isLooping` ignored → forwards `loop`, sets `autoplay`/`loop` on element
  creation, replays manually only when native loop is off (three small patches).

Idempotent (marker string). Does **nothing** when the Luna 7.2 strings don't match (other Luna version, or Luna
fixed it): it never guesses. After a Luna upgrade, run `Apply Build Patches Now` and check the patched-file count;
"0 patched" on a fresh build means the needles no longer match.

Replaces the per-project patchers `PlayableSpiderLunaPhysicsPatch` (GangterCity, Craft) and
`CodexLunaStage4AutoPatch` (VideoPlayer): don't run both.

## Not in the package

- `PAShoreFieldGenerator` (`Tools/PlayableKit/Shore Field Generator`): StickiiPA only (rejected as project-specific).
- GangterCity `LunaTools` (luna.json reflection bridge), BikePlayable preflight validator: project-local.
- Playable Orientation Setup (Fake-Video): not ported (depends on `AspectRatioFitter` and object names).
