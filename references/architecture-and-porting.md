# Playable Ad Architecture & Porting

Kit-agnostic patterns proven in company PAs. For PlayableKit's ready-made versions (`PAGameManagerBase`,
`PABaseUIController`, `PALunaManager`…) use the `unity-luna-playablekit` sub-skill.

---

## 1. Runtime architecture

1. **Self-contained, flat, no main-game singletons.** A PA scene's script closure must not reach full-game services
   (save systems, IAP, Firebase, scene managers). Guard unavoidable references with `#if UNITY_LUNA` /
   `#if !UNITY_LUNA` and exclude the folders in `luna.json`.
2. **One coordinator per mode.** A small state machine (`Intro → Playing → StoreOpened/Comeback → Ended`) owns flow.
   One panel/root component holds all serialized scene references and constructs plain C# helpers (spawner, feed
   zone, HUD, tutorial hand, FX) in `Awake()`, ticking them from its own `Update()`. Helpers: no `MonoBehaviour`,
   no singletons. (mukbang: `PAGameFlow` → `PALiveGame`, `PALivePanel` + `PALiveFoodSpawner`, `PALiveHud`…)
3. **Two Luna touch points only**: the lifecycle manager (`GameStarted`, `GameEnded`, `InstallFullGame`, timers)
   and, if needed, the coordinator reacting to `OnPause`/`OnResume`. Nothing else calls `Luna.Unity.*`.
4. **Pool from inactive scene templates.** Every runtime-spawned object (chat lines, FX, emoji particles) is cloned
   from an inactive instance already in the scene; never load prefabs at runtime.
5. **Update-driven timing** over deep coroutine nesting; clamp `dt` to 0.25 s.
6. **Bake in the Editor** what Luna can't do at runtime: combined meshes, sprite-sheet UV rects, layout rects,
   distance fields.

---

## 2. Scene hierarchy & dual orientation

Example (mukbang, project-local classes):

```text
_____________                 separator
  LunaManager                 [lifecycle manager, idle timeout]
  Canvas_CTA_Ver              [Canvas order 999, 1080x1920 match 0.5, safe area] > Btn (GoToStore)
Ver                           [Canvas order 500, CanvasScaler 1080x1920 match 0.5, mode coordinator]
  RootLayout
    LiveController            [panel: all scene refs]
      BG/Content              [aspect fitter]
      Game                    characters, slots, emitters, inactive FX templates
      HideableUI              [CanvasGroup] player info, tutorial hand, chat, title
Canvas_CTA_Hor                [Canvas order 999, 1920x1080 match 1] > safe area > Btn (GoToStore)
Sound                         [sound manager] > Music, Sfx0-2 (AudioSources)
EventSystem                   [EventSystem + StandaloneInputModule in Luna]
Canvas                        background, order 0 > BG [aspect fitter]
Main Camera
```

Orientation options:
- **Single gameplay canvas + per-orientation rect presets** applied on orientation change, toggling the portrait /
  landscape CTA canvases.
- **Two canvases** (Ver / Hor) toggled by an orientation checker. Capture rest `anchoredPosition` in `Awake()` and
  re-run layout in `OnEnable()` (inactive canvases have stale rects).

Landscape test: `Screen.width > Screen.height`, polled every frame (no reliable resize event in Luna). Serialized
per-orientation fields must use **non-generic** wrapper types ([constraints §1](./constraints-and-gotchas.md#1-c-language--transpiler)).

---

## 3. Porting a PA to a new Unity project

1. **Unity & packages**: Unity 2022.3 LTS (company standard 2022.3.62f2) unless the project is on Unity 6.
   Reference `com.unity.playworks.upp` (local `7.2.0/scripts`), uGUI, TextMeshPro, 2D Sprite, and **one** Spine
   runtime. Install MSBuild 17.14 + .NET 4.7 Developer Pack.
2. **Copy the PA code closure as-is**: playable scripts (with their `Editor/`), effects, lifecycle manager, tween
   helper, checker/sanitizer editor tools, PA prefabs, `Assets/csc.rsp`, `Assets/SVC_Luna.asset`, the template
   scene(s). (mukbang's exact list: its `Doc/PA_Architecture_Porting.md`.)
3. **Untangle main-game types**: extract enums/base classes the PA uses (e.g. mukbang's `PALiveFood` base
   `BaseSMLiveItem`, `SMItemSO`, `FoodTaste`, `CharacterState`) without pulling in services (`DataServices`,
   `SoundConfig`, `UIHeader`).
4. **`luna.json`**: copy, then reset `projectId`, `applicationID`, `creativeName`, `pgServer`; enable only the new
   scene; clear `unusedModules` / `unusedClasses` / `unusedMethods`; keep excludes and texture overrides.
5. **Verify & build**: run sanitizer/checkers, stop the Luna preview server, full build, test portrait + landscape,
   CTA, store return, idle timeout, end timer in a browser; then Release.

---

## 4. Debugging a Luna-only bug

1. Reproduce in a **Develop** build; read the browser console and `LunaTemp/luna.log`.
2. Confirm your script reached `LunaTemp/stage3/processed-scripts/` and the build isn't stale (timestamps).
3. Read Luna's JS implementation of the API (`stage3/engine/unity/bin/UnityEngine*.js`, `engine/luna/script1.js`,
   or a vendored `Doc/LunaSDK/`) instead of guessing.
4. If the bug only shows in Release, it is stripping: replace the API (e.g. manual inverse transform).
5. Record the trap with symptom, cause, fix and file:line in the team handbook.
