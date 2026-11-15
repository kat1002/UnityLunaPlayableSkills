---
name: unity-luna-playworks
description: >-
  Build, debug, optimize and ship Unity playable ads with Unity Playworks / Luna (6.4, 7.2). Use when writing
  Luna-safe C# or uGUI, wiring Luna.Unity LifeCycle / Playable.InstallFullGame / Analytics / LunaPlaygroundField,
  configuring luna.json or the Playworks plugin, cutting build size under 5 MB, fixing Luna-only build or browser
  bugs (stage 1-4 errors, "X is not a function", black screen, silent audio, dead buttons, null fields), or taking a
  PA concept from brief to deliverable. Works with or without PlayableKit.
---

# Unity Luna Playworks (Playable Ads)

Luna does **not** use the Unity WebGL player. It decompiles C#, recompiles it to JavaScript (`LunaTemp/stage3/`),
strips everything unreferenced, and re-exports scenes/assets with its own JS Unity/uGUI implementation, bundled into
one HTML5 ad (`LunaTemp/stage4/`). **Editor compile and Play Mode prove nothing.** Every non-trivial change needs a
Luna build tested in a browser.

**PlayableKit**: if the project contains the `com.kat.playablekit` package (`Packages/manifest.json`) or a vendored
copy (`Assets/**/PlayableKit/`, `PALunaManager`, `UIShine`), also read the
[`unity-luna-playablekit` sub-skill](./unity-luna-playablekit/SKILL.md). This skill never requires the kit.

---

## 1. Hard rules

1. **Test in Luna builds.** Develop build after every non-trivial change; **Release (minified)** build before
   shipping: stripped-API bugs (`… is not a function`) appear only there.
2. **Full build after schema changes.** "Recompile" rebuilds code only and reuses the old stage-1 scene export.
   After adding/renaming/retyping any serialized field, component or `[LunaPlaygroundField]`, run a full build.
3. **`luna.json` is cached per domain reload.** After editing it on disk (hand, script, agent), restart Unity or
   force a domain reload before building; re-check the file after the build.
4. **One enabled scene** per creative (`unity.scenes`, `startupScene: 0`), fresh `creativeName`, IDs not shared with
   another game.
5. **Never hand-edit `LunaTemp/`** except the two automated post-build patches (physics3d mesh inertia, video loop);
   never commit it. Never touch SDK `pipeline/.auth_*` files.
6. **Stop the Luna preview web server (port 8002) before rebuilding**, or stage 4 fails with `EBUSY … bridge.js`.
7. **Never hand-edit `.unity` scene YAML**; change scenes via the Editor/Editor scripts, with Play Mode stopped.
8. **Never call `SpriteAtlasUtility.PackAtlases` from scripts / MCP `execute_code`**: native crash in Unity
   2022.3.62. Creating/configuring `.spriteatlas` assets is fine; pack via Inspector or Play/Build.
9. **CTA: `Luna.Unity.Playable.InstallFullGame()` synchronously inside the click handler.** No delay.

---

## 2. Do / Don't cheat sheet

| Area | ❌ Breaks in Luna | ✅ Do instead |
|---|---|---|
| C# syntax | `new()`, `is not`, `is A or B`, `^1`, `= default`, value tuples, `async`, `goto`, destructors | Plain C# 7 |
| C# APIs | `PlayerPrefs`, `JsonUtility`, runtime `Resources.Load`, `System.Reflection`, `try/catch` logic, `RuntimeInitializeOnLoadMethod`, `OnMouseDown`, `Application.OpenURL`, UniTask, threads | In-memory state, serialized refs, scene objects + `Awake`, uGUI events / own raycast, `InstallFullGame()` |
| Hot paths | LINQ, lambdas/closures, `GameObject.Find`, `FindObjectOfType` | `for` loops, named methods, wired references |
| Unity null | `?.` / `??` on Unity objects | `== null` |
| Serialization | Generic `[Serializable]` fields (`Oriented<T>`) → silently null | Concrete non-generic wrapper classes |
| UnityEvent-only methods | Stripped → button dead | `[Preserve]` + `public` |
| Missing overloads | `GetComponentInParent<T>(true)`, `ParticleSystem.Play(true)`, `AudioListener.pause` | Walk `transform.parent`; no-arg overloads; `AudioListener.volume = 0` and restore saved value |
| Minified strip | `Transform.InverseTransformPoint`, `worldToLocalMatrix` | `Quaternion.Inverse(rot) * (p - pos)` / `lossyScale` |
| Instantiate | Prefab assets not in scene; `Instantiate(component)` | Clone inactive scene templates; instantiate `.gameObject` |
| Input | `InputSystemUIInputModule` (absent → dead UI) | `StandaloneInputModule` under `#if UNITY_LUNA`; reconcile held pointers per frame for fast taps |
| UI layout | `LayoutGroup`, `ContentSizeFitter`, `AspectRatioFitter`, `ScrollRect`, `Scrollbar`, `Slider` | Bake rects, own fitter / drag-scroll, per-orientation presets |
| UI hiding | `graphic.enabled = false`; `SetActive` on children inside `OnDisable` | `gameObject.SetActive(false)`; reset children on next show |
| Clipping | `RectMask2D` (no-op) | Avoid clipping, or stencil `Mask` + Simple image (build-test) |
| UI effects | uGUI `Outline`/`Shadow`, particles on Overlay canvas | Offset text copies; Screen Space – Camera canvas |
| TMP | Dynamic atlas, newer TMP shaders | Static atlas, 3.0.7 shaders, `Mobile/Distance Field` |
| Shaders | URP Lit, Shader Graph, `#pragma target` > 3.0, URP `Sprite-Unlit-Default`, Texture2DArray | `Sprites/Default`, `UI/Default`, `Unlit/*`, simple custom shaders |
| URP | Nothing referencing URP → black screen; URP 2D Renderer on Unity 6 → `LP1025` | Keep-alive `typeof(UniversalRenderPipelineAsset)`; `UniversalRendererData` |
| Materials | `MaterialPropertyBlock` on sprites; mutating a cached `Vector4` for `SetVector` | Material instances; `new Vector4(...)` each call |
| Data textures | Fully opaque PNG / `RGB24` → exported as JPEG | Alpha 254, `RGBA32`, `compression: none` |
| Particles | Trails module, bursts, sprite-sheet start frame, `Play()` on activation frame, toggling rate 0→N | `TrailRenderer`, `rateOverTime`, wait a frame then `Stop/Clear/Play`, `Emit(n)` from Update |
| Animator | `SetTrigger(hash)`, humanoid, 2 anims at once, private event methods | String overloads, Generic rigs, public event methods |
| Coroutines | Nested `IEnumerator` whose last yield is another bare `IEnumerator` → parent dies | `yield return StartCoroutine(...)` or Update-driven |
| Tweens | DOTween `Sequence`, chained lambda callbacks | One `DOVirtual.Float` with a named tick method, or own Update tween |
| Physics | Rigidbody + MeshCollider, NavMesh, `CombineMeshes` at runtime | Primitive colliders (or patch), own pathing, bake in Editor |
| Audio | Compressed In Memory / Streaming → silent | **Decompress On Load**, mono 64 kbps |
| Video | `Play()` once, `isLooping`, unmuted autoplay | Retry `Play()` until playing, loop patch, muted video + separate AudioSource |
| Spine | `.skel.bytes`, two runtimes | JSON export, one runtime |
| Packages | Odin, Firebase, Adjust, ad SDKs, Addressables, Timeline, Visual Scripting | Remove / exclude / guard `#if !UNITY_LUNA` |

Details, causes and project evidence: [constraints-and-gotchas.md](./references/constraints-and-gotchas.md).

---

## 3. Workflows

### A. New concept → shipped creative
Copy the closest existing scene (never start from scratch) → name scene + `creativeName`
(`<Game>_<Theme>_PA_<NN>_<ddMMyy>_<Author>`) → `luna.json` (one scene, fresh IDs) → wire lifecycle, end timer,
idle store rule, tutorial hand, audio, both orientations → run the project's checkers → Develop build + browser
test → Release build + retest → zip with exactly one `.html`.
Full checklist: [shipping-checklist.md](./references/shipping-checklist.md).

### B. Lifecycle, CTA & end card
- `GameStarted()` once on first touch; `GameEnded()` at most once at the final end; `InstallFullGame()`
  synchronously in the click with a 0.25 s repeat guard; mid-game CTA does not end the game.
- Company timing rule: after the **last touch**, open the store in **> 5 s and < 7 s**; end timer 25–40 s; a safety
  timeout; full-screen CTA fallback on the end card (auto store opens can be blocked).
- Pause via `AudioListener.volume`; never gate resume with `Time.unscaledTime`; clamp `dt` to 0.25 s.
- Kit-free reference implementation: [luna-api-and-lifecycle.md](./references/luna-api-and-lifecycle.md).

### C. `luna.json`, build & size
One scene; exclude `**/Editor/`, SDKs, unused modes (Luna transpiles **every** non-excluded script); force-include
`TMP Settings.asset`; textures ≤ 512 `png32` `quantSize` 55–65 with explicit overrides for atlases/backgrounds/data
textures; audio mono 64 kbps; measure `LunaTemp/stage4/create-hub/*_build.zip` ≤ 5 MB (verify the network spec).
Stage-by-stage debugging and toolchain errors: [plugin-ui-and-luna-json.md](./references/plugin-ui-and-luna-json.md).

### D. Debugging a Luna-only bug
Develop build → browser console + `LunaTemp/luna.log` → confirm the script reached `stage3/processed-scripts/` and
the build isn't stale → read Luna's JS for the API (`stage3/engine/…`) → Release-only? it's stripping.
Symptom index: [constraints §14](./references/constraints-and-gotchas.md#14-symptom-index).

### E. Pre-build checks
Which checker/sanitizer exists differs per project: grep the `MenuItem(` string first.
[company-projects.md](./references/company-projects.md#editor-tools-by-project) lists them and what they flag; the
grep fallback list is in [shipping-checklist.md §4](./references/shipping-checklist.md#4-before-build).

---

## 4. References (read on demand)

| File | Read when |
|---|---|
| [constraints-and-gotchas.md](./references/constraints-and-gotchas.md) | Writing/porting runtime code, UI, shaders, particles, audio, video, Spine; any Luna-only bug |
| [luna-api-and-lifecycle.md](./references/luna-api-and-lifecycle.md) | `Luna.Unity.*` calls, CTA/store, end card, idle/timers, pause/store return, Playground fields |
| [plugin-ui-and-luna-json.md](./references/plugin-ui-and-luna-json.md) | Install/versions, `luna.json`, build stages, toolchain errors, size budget |
| [shipping-checklist.md](./references/shipping-checklist.md) | Starting a concept, naming, pre-build/test/release checklist, deliverable zip |
| [architecture-and-porting.md](./references/architecture-and-porting.md) | Structuring PA code/scenes, dual orientation, porting to a new Unity project |
| [company-projects.md](./references/company-projects.md) | Which project/tool to use, per-project traps, shared assets, PA_Assets_Extractor |
| [unity-luna-playablekit/SKILL.md](./unity-luna-playablekit/SKILL.md) | Project uses PlayableKit (package or vendored 1.x copy) |

New Luna trap found? Add it to the team handbook (`02_Luna_Rules_And_Pitfalls.md`) and to
`references/constraints-and-gotchas.md` with symptom, cause, fix and the file:line of the fix.
