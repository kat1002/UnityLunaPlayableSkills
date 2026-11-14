# Luna Constraints, Engine Quirks & Runtime Gotchas

Luna transpiles C# to JavaScript (`LunaTemp/stage3/`) and uses its own JS implementation of Unity and UGUI (`UnityEngine.UI.js`, `engine/luna/script1.js`). **Passing Unity Editor compilation and Play Mode does not guarantee the playable will work in Luna.**

---

## 1. Compile-Time & Serialization Constraints

1. **Missing API Overloads (Stage 3 `CS1501`)**:
   - `GetComponentInParent<T>(bool includeInactive)` **does not exist** in Luna. Walk `transform.parent` in a `while` loop and call `GetComponent<T>()` (which works on inactive objects).
   - Avoid newer Unity convenience overloads; prefer classic Unity 2019/2020 overload signatures.
   - **`AudioListener.pause` does not exist** (Stage 3 `CS0117`). Silence on Luna `OnPause` with `AudioListener.volume = 0f` and restore the saved volume on `OnResume` (don't force `1f`, or an in-game mute toggle gets undone). `AudioListener.volume` is supported.
2. **Generic `[Serializable]` Types Are Silently `null`**:
   - Luna 7.2 cannot deserialize generic serializable field types (e.g., `[SerializeField] private Oriented<RectTransform> m_Layout;` evaluates to `null` at runtime in the build with no error).
   - **Fix**: Always use concrete non-generic `[Serializable]` classes (e.g., `OrientedRect`, `PARectPreset`).
3. **Forbidden C# Syntax & APIs** (flagged by `PlayableAdsCheckerWindow` & `NoRuntimeResourceLoadTests`):
   - **No runtime `Resources.Load`**: Assign all prefabs, sprites, audio clips, and ScriptableObjects via Inspector references (`PAConfig`, `PASound`, etc.).
   - **No `PlayerPrefs` or `JsonUtility`**: Keep session state in memory (`PAWallet`, `LunaManager.Session`).
   - **No C# 8/9+ syntax**: Avoid target-typed `new()`, pattern matching (`is not`, `and`, `or`), index-from-end `^`, `Array.Empty<T>()`, and tuples.
   - **No `try/catch`, `goto`, `System.Reflection`, `UniTask`, `AsyncWaitForCompletion`, `RuntimeInitializeOnLoadMethod`, or `OnMouseDown`**.
   - **No lambdas / LINQ / per-frame allocations in hot paths**: Use plain `for` loops and named methods.
   - **No `GameObject.Find` or `FindObjectOfType` at runtime**: Wire references in the scene or Editor setup scripts.

---

## 2. Input & `EventSystem` Gotchas

### 2.1 Dead UI Clicks (`InputSystemUIInputModule` Missing in Luna)
- **Symptom**: Tap particles/FX triggered by `Input.GetMouseButtonDown(0)` work, and `EventSystem.RaycastAll` finds UI elements, but **no `Button.onClick`, `IPointerDownHandler`, or drag events ever fire** in the Luna build.
- **Root Cause**: Luna's `UnityEngine.UI.js` only includes `StandaloneInputModule` — it does **not** ship `InputSystemUIInputModule`. If the scene's `EventSystem` uses the new Input System module, Luna has no active input module.
- **Fix**:
  1. Guard any new Input System code with `#if ENABLE_INPUT_SYSTEM && !UNITY_LUNA`.
  2. At startup under `#if UNITY_LUNA`, ensure `StandaloneInputModule` exists on `EventSystem.current`:
     ```csharp
     #if UNITY_LUNA
     if (EventSystem.current != null && EventSystem.current.GetComponent<StandaloneInputModule>() == null)
     {
         EventSystem.current.gameObject.AddComponent<StandaloneInputModule>();
     }
     #endif
     ```

### 2.2 Lost Fast Taps (Polled Input Flag Clearing in `script1.js`)
- **Symptom**: Very fast taps/clicks (where pointer down and pointer up occur between two frames) are missed by code checking `Input.GetMouseButtonDown(0)` or `TouchPhase.Began`.
- **Root Cause**: In Luna's `engine/luna/script1.js`, `_onMouseUp` sets `Up = true`, `held = false`, and **clears `Down = false`** before `prerender`. Thus a sub-frame tap presents as `GetMouseButtonDown == false` and `GetMouseButtonUp == true`.
- **Fix**: When writing custom polled input (`Reconcile`), compare current held pointer IDs against the previous frame's held IDs:
  - Unknown held ID -> synthesize `Down`.
  - Vanished held ID -> synthesize `Up`.
  - `Up` without prior held ID -> synthesize `Down` + `Up`.

### 2.3 Decorative UI Blocking Raycasts
- Turn **off** `raycastTarget` on decorative `Image`, `TextMeshProUGUI`, tutorial hand sprites, and rainbow frames so they do not swallow taps/drags.

---

## 3. UGUI, Layout & Clipping Gotchas

### 3.1 `DrivenRectTransformTracker.Clear` & Layout Components
- **Warning / Bug**: `ScrollRect`, `Scrollbar`, and `Slider` trigger `"avoid UnityEngine.DrivenRectTransformTracker.Clear"` in Luna. Additionally, `HorizontalLayoutGroup`, `VerticalLayoutGroup`, `GridLayoutGroup`, `ContentSizeFitter`, and `AspectRatioFitter` evaluate lazily in Luna, causing panels to appear shifted (~360px off on iPad 3:4) or wrongly shrunk on first open.
- **Fix**:
  1. Run **`Tools/PA/Sanitize UI For Luna`** (`PALunaUiSanitizer`) on the active scene. It temporarily activates inactive hierarchy branches, forces a canvas rebuild, **bakes** child `RectTransform` anchors/offsets, removes `LayoutGroup` & `ContentSizeFitter`, converts `AspectRatioFitter` to `PAAspectFitter`, and converts `ScrollRect` to `PADragScroll`.
  2. Never measure layout-group child rects immediately after `SetActive(true)` on a never-shown panel; use deterministic authored sizes or per-orientation presets (`PALayout` / `PARectPreset`).
  3. Re-run custom layout/fitter logic in `OnEnable` when an inactive canvas becomes active (inactive canvases have zero/stale rect sizes).

### 3.2 `RectMask2D` Does NOT Clip in Luna — Use `Mask`
- **Root Cause**: In Luna's `UnityEngine.UI.js`, `RectMask2D.SetClipRect` is an unimplemented `TODO` no-op. Children inside a `RectMask2D` overflow visibly in the browser build.
- **Fix**:
  1. Replace `RectMask2D` with Unity's stencil **`Mask` + `Image`** component on the viewport object.
  2. Use a **plain rect (`Image.type = Simple` or no sprite)** on the `Mask` `Image` — a `Sliced` sprite creates a polygon stencil that clips corners weirdly.
  3. Because Luna lazily syncs `Image` vertex meshes, if you resize masked tiles or viewports at runtime, call `graphic.SetVerticesDirty()` on the tiles and viewport mask `Image` 1–2 frames after resizing (or toggle `Mask.enabled` off/on).

---

## 4. Coroutines, Tweens, Animator & Particles

### 4.1 Coroutine Nesting Bug (Silent Caller Termination)
- **Root Cause**: In Luna's `Coroutine#MoveNext`, when a nested `IEnumerator` finishes, Luna pops only one level and advances the parent once. If a nested `IEnumerator`'s **last** `yield return` is *another* bare `IEnumerator` (e.g., `yield return WaitWhilePaused();` at the end of `Helper()` yielded via `yield return Helper();`), both finish on the same step and **Luna marks the entire grandparent coroutine done**, silently halting the game flow with no exception.
- **Fix**:
  - Either wrap nested coroutines with `yield return StartCoroutine(Helper())`,
  - Or prefer `Update`-driven state machines (`PALiveEatController.Tick()`) and `DOVirtual.DelayedCall(duration, NamedMethod)`.

### 4.2 Tweens (`TweenLite` / `PATween`)
- Do **not** use DOTween `Sequence` or chained lambda `OnComplete` callbacks.
- Drive multi-segment animations from a single `DOVirtual.Float(0f, 1f, duration, OnTick)` callback using `PATween.Play` and `PATween.Seg(t, start, len, ease)`.
- If a project contains both `DG.Tweening` and `Playable.Tweening` (`TweenLite.cs`), **never** import both `using DG.Tweening;` and `using Playable.Tweening;` in the same file; alias one (`using PT = Playable.Tweening;`).

### 4.3 Animator Parameters: Use Strings, Never `StringToHash`
- Refactoring `Animator.SetBool("IsMove", val)` / `Animator.SetTrigger("Die")` to integer hashes (`Animator.StringToHash`) causes animations (especially triggers like `Die`) to silently fail in Luna builds. Always use the **string parameter overloads**.

### 4.4 Particle Systems
- **Unsupported in Luna**:
  - ParticleSystem **Trails** module (ribbon trails vanish in build) -> use a `TrailRenderer` component on a child GameObject instead.
  - ParticleSystem **Bursts** and sprite-sheet start frame -> use `rateOverTime` or `rateOverDistance`, or use custom pooled UI emitters (`EmojiImageParticleEmitter`).
- **Toggling `emission.rateOverTime` 0 -> N at runtime ("pull back" / inward-collapsing stars)**: Luna's rate emitter
  (`particle-system-1.js`) catches up `while (lastSpawn + 1/rate < playTime)` and spawns each particle pre-aged by the
  backlog. While rate is 0, `lastSpawn` freezes but `playTime` keeps running (only `play()` resets it; `isPlaying` stays
  true after play-on-awake), so turning the rate back on spawns up to maxParticles already-far-out particles whose offset
  shrinks each frame, which reads as a ring collapsing to the center. Editor looks fine.
  **Fix**: keep `rateOverTime` 0 in the scene and emit from Update with `ps.Emit(n)` plus a fractional accumulator
  (`acc += rate*dt; n=(int)acc; acc-=n`). Do not call `Play()` to reset it, because Luna's `play()` clears live particles.
  (Seen in StickiiPA PA02 `StickerWorldGame.Shower` / `StickerZipBag.SetStars`.)

---

## 5. Custom Shaders, Materials & Spine

### 5.0 TMP SDF atlas downscaled by luna.json -> blurry outlined/large text
- **Symptom**: One TMP text looks soft/wobbly in the build while other text from the same font looks fine.
- **Root Cause**: A `luna.json` texture override (e.g. `maxWidth/maxHeight: 512`) on the `*SDF.asset` halves the atlas
  (1024 -> 512), halving the effective sampling size and SDF spread, while the material keeps `_TextureWidth`/`_GradientScale`.
  Outlined materials (`OUTLINE_ON`) and big auto-sized text show it first; small no-outline text hides it.
- **The per-asset override cannot be changed**: on every export the plugin (decompiled `EditorScripts.dll`, 7.2.0)
  deletes the font asset's own texture override and recreates it from the texture default, or from the longest
  *folder* override whose name is a prefix of the asset path, with compression forced to `none`. A size typed on the
  font asset's own row is therefore reset to the default (512) on each build.
- **Fix**: Add a *folder* override (e.g. `"name": "Assets/Fonts/Baloo/"`, `maxWidth/maxHeight: 1024`) or raise the
  texture default; verify with `LunaTemp/stage2/assets/textures/<id>.png` dimensions. If the size budget is tight,
  regenerate a smaller ASCII-only atlas instead of letting Luna downscale it.

### 5.1 `Sprite.textureRect` vs Downscaled `Texture.width / height`
- **Root Cause**: When Luna downscales a texture (e.g., from `950x660` to `512x356` per `luna.json`), `sprite.textureRect` still returns **native pixels** (`950x660`), whereas `sprite.texture.width/height` returns the **downscaled dimensions** (`512x356`). Computing UV bounds as `textureRect / textureSize` yields values `> 1.0` (`1.855`), breaking custom shaders!
- **Fix**:
  - Never access `sprite.texture` at runtime if avoidable — bake sprite sheet UVs in the Editor (`EmojiSheetLayoutBaker`).
  - If computing UV rect at runtime, check if `uvRect.xMax > 1.01f || uvRect.yMax > 1.01f` and fall back to `(0, 0, 1, 1)`. Aspect ratio from `textureRect.width / textureRect.height` remains accurate.

### 5.1b UV-Based Effects on `Sliced` Images
- **Symptom**: a sweep/gradient shader (e.g. `PA_UI_Shine`) crawls over a 9-sliced button's ends, then jumps across the middle in one frame and bends.
- **Root Cause**: Sliced UVs are not proportional to on-screen position: borders keep their pixel size while the centre column of the sprite stretches (76px sprite with 37px borders on a 700px button → ~97% of the width maps to ~2.6% of UV range).
- **Fix**: undo the slicing in the shader with per-axis border fractions (sprite-UV borders vs drawn-rect borders) set from C# (`PAUIShine.RefreshSlice`); zeros = identity for `Simple`. Uniforms only, so Luna-safe (avoid extra Canvas shader channels / `PositionAsUV1`, whose Luna support is unverified).

### 5.2 `Material.SetVector` Passes by Reference in Luna
- Luna's JS `setVectorParameter(name, e.data)` caches and compares vector references. Mutating a long-lived `Vector4` field in place and passing it back to `mat.SetVector("_Param", m_CachedVec)` can be skipped by Luna's dirty check.
- **Fix**: Always pass a fresh `new Vector4(x, y, z, w)` to `Material.SetVector`.

### 5.2b UI Vertex Space Differs (Position-Based UI Shader Effects Land in the Wrong Place)
- **Symptom**: a UI shader effect that derives position from `v.vertex` (or `mul(unity_ObjectToWorld, v.vertex)`), e.g. a ripple centre or a hole shadow keyed to a map, is correct in the Game view but shifted/scaled in the Luna build, and wrong again in the Scene view.
- **Root Cause**: UI vertex space is not consistent: Unity's Game view batches into canvas space, the Scene view draws a Screen Space Overlay canvas with a different matrix, and Luna draws each graphic with its own transform.
- **Fix**: derive positions from the **sprite UV**, with a per-material affine set from C# (`mapPx = uv * _UVToMapPx.xy + _UVToMapPx.zw`, computed from `RectTransform.GetWorldCorners` on the CPU, written only when it changes, as a fresh `Vector4`). Keep `v.vertex` only for `_ClipRect` clipping. Assumes Simple, unrotated sprites whose mesh spans `textureRect` (see 5.1 for downscaled textures).

### 5.2c Opaque Textures Are Exported as JPEG (Breaks Data Textures)
- **Symptom**: a shader reading a data texture (distance field / SDF / lookup / masks packed in RGB) looks right in the Editor but misshapen in the build (e.g. an inner hole shadow inset with a hard edge). `LunaTemp/stage2/assets/textures/<id>.json` shows `"ext": ".jpeg"`.
- **Root Cause**: Luna converts any texture whose **imported pixels are fully opaque** to lossy JPEG, ignoring the `luna.json` rule (`format: "png32"`, `compression: "none"`). It reads the texture as imported for the **active build target** (often Standalone), so a WebGL platform override does not help. An explicit `RGB24` default format drops alpha even if the PNG has it. JPEG chroma subsampling then corrupts each channel independently.
- **Fix** (all three):
  1. Bake a non-opaque alpha into the source PNG (e.g. alpha = 254 everywhere; harmless when the shader reads only `.rgb`). Keep `alphaIsTransparency` off.
  2. Set the importer **default** format to `RGBA32`, uncompressed (sRGB off for data).
  3. Set the texture's `luna.json` rule to `"compression": "none"` (`quantSize` palette-quantizes smooth gradients).
- **Verify** after build: `ext` is `.png` and `LunaTemp/stage2/assets/textures/<id>.png` exists.
- **Hole/shore distance fields**: generate with **`Tools/PlayableKit/Shore Field Generator`** (`PAShoreFieldGenerator`), which applies steps 1-2 automatically (alpha 254, RGBA32, linear) and writes `_ShoreTex` / `_PairSplitY` / `_MapSize` to the chosen materials; only the `luna.json` rule (step 3) is manual.

### 5.3 Spine & Render Pipeline Shaders
- **Spine Data Format**: Luna requires JSON Spine exports (`.json` assigned to `skeletonJSON`). Binary `.skel.bytes` fails in Luna — verify with **`Tools/MyLunaHelper/Spine Binary Inspector`**.
- **Single Spine Runtime**: Never keep both the UPM package `com.esotericsoftware.spine.spine-csharp` and local source `Assets/Libs/Spine/` in the same project (causes GUID conflicts).
- **Render Pipeline & Shaders**:
  - **URP 2D Renderer fails export (Stage 1 `LP1025`, `ExportCollectionException: UrpAssetsCollection`, `Member 'Renderer2DData.copyDepthMode' not found`)**: Luna 7.2 reflects 2D-renderer fields that don't exist in URP 17 (Unity 6). Projects made from the URP 2D template hit this. **Fix**: create a `UniversalRendererData` asset (assign URP's default `PostProcessData`), make it the only entry in the URP asset's `m_RendererDataList`, and delete the template's leftover `Global Light 2D`. Safe when the playable is UGUI/unlit (no `Light2D` lighting, no `Universal2D`-only shader passes).
  - Check active pipeline via `GraphicsSettings.currentRenderPipeline` before changing materials (do not blindly trust docs if a project uses Built-in `Standard`).
  - For Luna compatibility, avoid `URP/Lit` and `Shader Graph` on scene renderers; use `Sprites/Default`, `UI/Default`, `Unlit/Texture`, or custom GLSL-clean shaders like `PlayableKit/UI_Shine`, and register required shader variants in `Assets/SVC_Luna.asset`.

---

## 6. Luna Engine JS Bugs Patched Post-Build (`PALunaBuildPatcher`)
Some Luna runtime bugs live in the generated JS, not in C#. `PALunaBuildPatcher` (package `com.kat.playablekit`,
`[InitializeOnLoad]`) polls `LunaTemp` every 5 s and re-applies string patches whenever Luna rewrites a file; a marker
string makes it idempotent. If a needle stops matching after a Luna upgrade the patch silently stops applying, so
re-check after changing Luna versions (`Tools/PlayableKit/Luna/Apply Build Patches Now` reports the count).
- **`physics3d-0.js`: `Error: Not implemented for the mesh shape`**: a Rigidbody with a `MeshCollider` throws when
  Luna computes inertia. The patch adds a box-approximation inertia from the mesh bounds. Without the patcher, use
  primitive colliders on dynamic bodies.
- **`UnityEngine.js`: `VideoPlayer.isLooping` not honoured**: the setter never reaches the HTML `<video>`, the element
  is created without `loop`/`autoplay`, and the end event replays without resetting `currentTime`. The patch forwards
  `loop`, sets `autoplay`/`loop` on creation, and only replays manually when native loop is off.
