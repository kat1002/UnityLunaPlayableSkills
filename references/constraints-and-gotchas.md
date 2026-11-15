# Luna Constraints, Engine Quirks & Runtime Gotchas

Luna does **not** ship the Unity WebGL player. It decompiles your C#, recompiles it to JavaScript with its own
compiler (`LunaTemp/stage3/`), strips everything not referenced, and re-exports scenes and assets with its own
JS implementation of Unity and uGUI (`UnityEngine.js`, `UnityEngine.UI.js`, `engine/luna/script1.js`). Unity WebGL
player settings do not matter. **Editor compile + Play Mode prove nothing**: test every change in a Luna build.

How to read: **Symptom** = what you notice; **Fix** = what to do. "Seen in" names the company project where the rule
was found or proven (see [company-projects.md](./company-projects.md)). Rows marked *(unverified)* are reported but
not proven against Luna source: test in a build before depending on them.

Contents: [1 C#](#1-c-language--transpiler) · [2 Unity APIs](#2-unity-apis-stripped-missing-or-partial) ·
[3 Input](#3-input--eventsystem) · [4 uGUI/TMP](#4-ugui-layout-clipping-tmp) ·
[5 Rendering](#5-rendering-shaders-materials-textures) · [6 Particles](#6-particles) ·
[7 Animation/Tweens/Coroutines](#7-animation-tweens-coroutines) · [8 Physics](#8-physics) · [9 Audio](#9-audio) ·
[10 Video](#10-video) · [11 Spine](#11-spine) · [12 Packages](#12-packages-that-break-luna) ·
[13 Generated-JS bugs](#13-luna-engine-js-bugs-that-need-a-post-build-patch) · [14 Symptom index](#14-symptom-index)

---

## 1. C# language & transpiler

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| Stage 3 compile error, or silent misbehaviour, though Editor is fine | C# newer than ~7.3: target-typed `new()`, `is not`, `x is A or B`, `arr[^1]`, `= default` literal, `async/await`, value tuples | Plain C# 7: `new List<T>()`, `!(x is T)`, `arr[arr.Length - 1]`, explicit `default(T)` | squishy, team notes |
| Stage 3 `LP2007 Unexpected symbol '{'` | Value tuples (`(string A, bool B)[]`, tuple dictionary keys) | Small class or nested dictionaries | squishy |
| Not supported at all | Destructors, `goto`, `OnGUI`, threads, file IO, `System.Reflection`, `DllImport`, `RuntimeInitializeOnLoadMethod`, `JsonUtility`, `System.IO.Compression.DeflateStream`, `Application.OpenURL` (store) | Remove; scene object + `Awake` instead of `RuntimeInitializeOnLoadMethod`; ship data uncompressed or inline an inflate routine; open the store only with `InstallFullGame()` | squishy, team notes, Ads Checker |
| Generic `List<UnityEngine.Object>`-style collections lose their element type in the generated JS | Transpiler | Keep small wrapper `Entry` objects in the list; avoid generics beyond `List`/`Dictionary` in runtime code (Editor code is exempt) | squishy, BikePlayable |
| Logic depending on exceptions misbehaves | `try/catch` not reliable for control flow | Never depend on `try/catch`; only as a last-resort guard around partially supported APIs (video teardown) | team notes |
| `PlayerPrefs` / save data | No persistence in an ad | Keep session state in memory | all |
| Runtime `Resources.Load` fails / bloats | Runtime resource loading unsupported | Serialized references or a ScriptableObject catalog | all |
| Slow frames or transpile bugs | LINQ, lambdas, closures, per-frame allocation | Plain `for` loops and named methods; no LINQ/lambdas in `Update` | PlayableKit, Obby |
| `GameObject.Find`, `FindObjectOfType`, `.FindAll(`, lambda `.Find(`, `Array.Empty<T>()` flagged | Slow / missing in Luna | Wire references in the scene; `new T[0]` cached. A one-time `Awake` fallback lookup is tolerated (BikePlayable), never in hot paths | Ads Checker |
| `?.` / `??` on a Unity object wrong | Unity "fake null" is not a JS null | `== null` / `!= null` | PlayableKit |
| `CS0104` ambiguous `Event` (`Spine.Event` vs `UnityEngine.Event`) only in Luna | Stage 3 recompiles decompiled C# and drops namespace qualifiers | Don't declare `Spine.Event` params/locals in a file that uses `UnityEngine`; poll the timeline instead | squishy |
| `CS0117` / `CS7036 … _002Ector` | `base(...)` ctor argument needing a temporary decompiles to an invalid ctor call | Move the expression into a static helper | squishy |
| Field `Enabled`, class `Notification` misbehave | Name collides with JS / Luna internals | Rename | squishy |
| Stub services (save, network, IAP, ads, tracking) report success | Fake success hides missing features | Stubs do nothing or fail visibly; never report success | squishy |
| Field set in Editor is **null** in the build, no error | Luna 7.2 silently drops serialized fields whose type is **generic** (`[SerializeField] Oriented<T>`) | Concrete non-generic `[Serializable]` wrapper types. Verify `Deserializers["<Type>"]` in `LunaTemp/stage4/develop/js/deserializers.js` | GangterCity, Craft |
| Scene fails to deserialize in the browser | Serialized field hiding `Component.transform` (e.g. a field named `transform`) | Rename (`m_CachedTransform`) | squishy |
| Stage 3 `CS1501` | Newer overloads missing, e.g. `GetComponentInParent<T>(bool includeInactive)` | Walk `transform.parent` and call `GetComponent<T>()`; prefer Unity 2019/2020 signatures | StickiiPA |
| `CS0117 AudioListener.pause` | Not in Luna | See §9 | all |
| Main-game types missing at stage 3 (`MySceneManager`, IAP, Firebase…) | PA scene's script closure pulls in full-game services | `#if UNITY_LUNA` / `#if !UNITY_LUNA` guards; exclude folders in `luna.json` `unity.scripts.excludes` | scoop |
| Editor code leaks into the build | `UnityEditor` usage in runtime scripts | Keep editor code in `Editor/` folders (excluded by `**/Editor/`) or under `#if UNITY_EDITOR` | all |

Defines: Luna builds define **`UNITY_LUNA`**. Guard Luna-only API calls with `#if UNITY_LUNA` so the Editor stays a
no-op, and guard new-Input-System code with `#if ENABLE_INPUT_SYSTEM && !UNITY_LUNA`.

---

## 2. Unity APIs: stripped, missing or partial

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| Button works in Editor, does nothing in build | Methods whose **only** reference is a serialized `UnityEvent` (`Button.onClick`) are stripped | `[UnityEngine.Scripting.Preserve]` + `public` on the handler; also call it from code if possible | Home-Clean |
| `… is not a function` on `Transform.InverseTransformPoint` / `worldToLocalMatrix` (**minified build only**) | Luna 7.2.0 Release builds strip them | `Quaternion.Inverse(t.rotation) * (world - t.position)`, then divide componentwise by `t.lossyScale` | squishy |
| `… is not a function`, C# is correct | **Stale build** | Rebuild; if it persists delete `LunaTemp/` and do a full build | mukbang |
| Renamed/retyped serialized field shows old data or null after "Recompile" | Recompile rebuilds code only (stage 3) and reuses the old stage-1 scene export | **Full** Luna build; confirm `LunaTemp/stage1` timestamp moved | GangterCity |
| `Mesh.CombineMeshes` does nothing / crashes | Not implemented | Combine in the Editor, ship the baked mesh/prefab | BikePlayable |
| `Sprites.DataUtility.GetOuterUV` missing | Not implemented *(unverified)* | UV rect from `sprite.textureRect` / texture size (see §5.2) | Craft |
| `Color.HSVToRGB` missing | Not implemented *(unverified)* | Own HSV→RGB helper | Craft |
| Runtime `sprite.texture` pixel work fails | No runtime texture access | Bake data in the Editor | mukbang |
| `string.Format` with floats prints wrong | Unreliable | Math + string concatenation | mukbang |
| `Instantiate(particleSystem)` fails | Luna clones only `GameObject` / `Transform` / `MonoBehaviour` | `Instantiate(ps.gameObject, parent)` then `GetComponent<ParticleSystem>()` | GangterCity |
| Sprite/Image read from a prefab **asset** that was never instantiated is unresolved | Luna resolves prefab references lazily | Instantiate the prefab once (or keep a scene template), then read the sprite | squishy |
| `Renderer.bounds` stale after activation / pose change | Bounds lag in Luna | Measure from data (`Skeleton.GetBounds`, sprite size × scale) | squishy |
| Feature silent in the build, fine in Play Mode | Helpers live on objects tagged `EditorOnly`, which are not exported | Never put runtime helpers on `EditorOnly` objects | squishy |
| Ported game code won't compile | `ReadPixels` and similar GPU-readback APIs | Don't port code that relies on them; bake in the Editor | squishy |
| Instantiated object missing / "prefab not found" | Prefab not present in the scene | Keep an **inactive template instance** in the scene and clone it | team notes, mukbang |
| `ParticleSystem.Clear(true)` / `Play(true)` not found | Only no-arg overloads | `ps.Clear(); ps.Play();` | Home-Clean |
| Ghost child keeps rendering after the parent is hidden | `SetActive` on children **during** parent deactivation (`OnDisable`) corrupts Luna's active state; child `OnDisable` is unreliable | In `OnDisable` only stop tweens; reset children on the next show. Stop effects explicitly **before** hiding the parent | Home-Clean, squishy |
| Hidden UI still draws (ghost/stale mesh) | `graphic.enabled = false`: Luna's `Graphic.OnDisable` never clears the `CanvasRenderer` | `graphic.gameObject.SetActive(false)` (guard with `activeSelf` if per-frame) | StickiiPA |
| Scene-load code fails | One scene per creative | Restart / end card inside the same scene; early-return scene loads under `UNITY_LUNA` | squishy, scoop |
| `Physics.OverlapSphere` / zero-length casts miss start overlaps | Only direct intersection tests proven | `Physics.OverlapSphereNonAlloc` | GangterCity |
| Video API throws on teardown | `UnityEngine.Video` partial | See §10 | Home-Clean |
| URP project: **black screen**, JS spam `reading 'Disabled'` (`LightRenderingMode`) | Luna ships only assemblies your scripts reference; nothing referenced URP so `UnityEngine.UniversalRenderPipeline.js` was dropped | Reference URP types from an always-compiled script (see §5.4) | Home-Clean |

---

## 3. Input & EventSystem

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| Polled taps/FX work but **no** `Button.onClick` / pointer / drag events fire | Luna's `UnityEngine.UI.js` ships only `StandaloneInputModule`, not `InputSystemUIInputModule` | Guard Input System code; add `StandaloneInputModule` under `UNITY_LUNA` (snippet below) | Obby, StickiiPA |
| Very fast taps missed by `GetMouseButtonDown` / `TouchPhase.Began` | `script1.js` `_onMouseUp` clears `Down` before the frame, so a sub-frame tap reads Down=false, Up=true | Reconcile held pointer IDs per frame: unknown held ID → synth Down; vanished ID → synth Up; Up without prior hold → synth Down+Up | Craft, StickiiPA |
| Drag drops Began/Ended or jumps | Luna drops phases or flips touch ↔ emulated-mouse IDs mid-drag | Same per-frame pointer reconciliation | Craft |
| `OnMouseDown/Up/Drag` never fire | Not supported | uGUI events or own raycast from pointer position | team notes |
| Taps swallowed | Decorative `Image`/`TMP`/hand sprites have `raycastTarget` on | Turn `raycastTarget` off on decorations | StickiiPA |
| Serialized buttons do nothing, controllers never initialize, scene still renders | The script is in `unity.scripts.excludes`: excluded scripts become empty JS objects | Check excludes first; excluding a library means excluding every script that uses it too | squishy |
| CTA not tappable in one orientation | Tap zone does not cover the full screen there | Check both orientations in the Luna preview | PlayableKit |

```csharp
#if UNITY_LUNA
using UnityEngine.EventSystems;
// Once at startup (Awake of your Luna/lifecycle manager):
if (EventSystem.current != null && EventSystem.current.GetComponent<StandaloneInputModule>() == null)
    EventSystem.current.gameObject.AddComponent<StandaloneInputModule>();
#endif
```

---

## 4. uGUI layout, clipping, TMP

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| `avoid UnityEngine.DrivenRectTransformTracker.Clear` warnings; panels shifted (~360 px on iPad 3:4) or shrunk on first open | `LayoutGroup` (H/V/Grid), `ContentSizeFitter`, `AspectRatioFitter`, `ScrollRect`, `Scrollbar`, `Slider` drive rects through `DrivenRectTransformTracker` and evaluate lazily | **Bake** the rects (force a canvas rebuild with inactive branches activated, copy anchors/offsets) and remove the drivers; replace `ScrollRect` with own drag-scroll, `AspectRatioFitter` with own fitter that sets size in `OnEnable`/on resize. A project sanitizer may exist ([company-projects.md](./company-projects.md#editor-tools-by-project)) | mukbang, StickiiPA, Craft |
| Layout wrong when a canvas becomes active | Inactive canvases have zero/stale rect sizes | Re-run custom layout in `OnEnable`; never measure layout children right after first `SetActive(true)`; prefer authored per-orientation presets | mukbang |
| Content overflows a clipping viewport | `RectMask2D.SetClipRect` is an unimplemented no-op in Luna's `UnityEngine.UI.js` | Stencil `Mask` + `Image` with a **Simple** (non-Sliced) sprite or no sprite; after runtime resize call `SetVerticesDirty()` on tiles and the mask image 1–2 frames later (or toggle `Mask.enabled`). Note: PlayableKit docs list `Mask` as "rewrite"; prefer avoiding clipping (sprite/shader reveals) and build-test any `Mask` you keep | StickiiPA |
| UI particles invisible | Particles don't render on Screen Space – Overlay canvases | Screen Space – Camera canvas or world-space particles | PlayableKit |
| Layout breaks in the other orientation | Ads run portrait **and** landscape | Author both; switch presets on orientation change; test by rotating while playing | team notes |
| TMP never renders; every TMP reference null | `bin/TextMeshPro.js` missing from the Luna compiler manifest | Ensure the manifest lists `bin/TextMeshPro.js` | squishy |
| uGUI `Outline` / `Shadow` effects draw nothing | Not implemented in Luna | Offset child copies of the text (or a TMP outline material) | squishy |
| Legacy `UI.Text` invisible | No real font assigned | Assign a `.ttf` font asset | BikePlayable |
| TMP translucent blocks / invisible | TMP shaders newer than the package (read SDF scale from `texcoord0.w` not `texcoord1.y`) | 3.0.7 Essential Resources shaders; re-import `Assets/TextMesh Pro` after package changes; `TextMeshPro/Mobile/Distance Field` | squishy |
| Dynamic TMP atlas shows no glyphs | Dynamic atlases are not populated in Luna | Add glyphs in the Editor, set atlas **Static** | squishy |
| One TMP text soft/wobbly, others fine | A `luna.json` texture override halves the SDF atlas (1024→512) while the material keeps `_TextureWidth`/`_GradientScale`; outlined and big text show it first | Folder override (see [plugin-ui-and-luna-json.md §3.3](./plugin-ui-and-luna-json.md#33-font-sdf-atlas-overrides-get-regenerated)) or a smaller ASCII-only atlas | StickiiPA |
| White boxes around Spine/PMA sprites, soft UI | Default texture rule caps 512 and quantizes | Explicit `compression: none` override with larger `maxWidth/maxHeight` for those textures | squishy |
| Atlas sprites outlined / blurry / dark fringe | Luna **halves** sprite atlases | Keep those sprites standalone; source PNGs uncompressed, no mips; reference sprites explicitly, never folder `DefaultAsset`s (pulls whole folders in) | mukbang |
| Tiled sprites wrong | Atlas padding | Full Rect mesh; keep tiling sprites out of atlases or add padding | mukbang |

---

## 5. Rendering, shaders, materials, textures

### 5.1 Table

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| Whole frame (UI too) vanishes; JS `this.commitFunction[t.dataType] is not a function` | A material samples a `Texture2DArray` (e.g. PolyFew **BatchFew**) | Never ship texture-array materials | GangterCity |
| Shader pink / missing | Shader stripped because nothing references it (e.g. `Shader.Find` at runtime) | Pin it: Always Included Shaders or a referenced material | BikePlayable |
| SpriteRenderer `Failed to compile shader program` | URP `2D/Sprite-Unlit-Default` skinning variant uses an `std430` SSBO | Built-in `Sprites/Default` material | StickiiPA |
| URP Lit / Shader Graph wrong or broken | Not supported reliably | `Sprites/Default`, `UI/Default`, `Unlit/*`, Mobile shaders, or hand-written GLSL-clean shaders; no `#pragma target` > 3.0 | squishy, PlayableKit |
| Sprite with custom shader stops drawing after sprite swap | Luna binds a SpriteRenderer's texture via its property block; `Get/SetPropertyBlock` pins a stale texture | Per-renderer **material instances**, not `MaterialPropertyBlock` | Home-Clean |
| Shader uniform does not update | Luna's `Material.SetVector` forwards the Vector4 **by reference** and dirty-checks it | Pass a fresh `new Vector4(...)` every call | Craft |
| Realtime shadows / lights expensive | Luna defaults | `luna.json` realtime shadows off; fake blob shadows (unlit ellipse decal) | Home-Clean, GangterCity |
| Shader variant missing (e.g. `Particles/Standard Unlit`, `Mobile/Particles/Alpha Blended`) | Variant not collected | Add to the Luna shader-variant collection (`Assets/SVC_Luna.asset`) | mukbang, squishy |
| Transparent renderer order ignores `sortingOrder` / depth changes | Luna draws transparent renderers in **render-queue** order | Give the renderer its own material with an explicit `renderQueue` (e.g. 3100), then reactivate it | squishy |
| Scrolling texture via Repeat wrap breaks | Non-power-of-two textures can't Repeat on WebGL1 | Scroll moving Image copies instead of UV/`_Time` scrolling, or use POT textures | squishy |

Texture **size and compression** are set by `luna.json` `assets.rules`, not Unity importers; the importer only
decides alpha/format, which controls PNG vs JPEG export (§5.5).

### 5.2 `Sprite.textureRect` vs downscaled texture size
When `luna.json` downscales a texture (950×660 → 512×356), `sprite.textureRect` still returns **native pixels**
while `sprite.texture.width/height` returns the downscaled size, so `textureRect / textureSize` > 1 breaks custom UV
shaders. Bake sprite-sheet UV rects in the Editor; if computed at runtime, fall back to `(0,0,1,1)` when
`xMax > 1.01f || yMax > 1.01f`. Aspect from `textureRect.width / textureRect.height` stays correct.

### 5.3 UV effects on Sliced images; position-based UI shaders
- Sweep/gradient shaders reading raw UVs on **Sliced** images crawl over the ends then jump across the middle:
  borders keep pixel size while the centre stretches. Undo slicing in the shader with per-axis border fractions
  (sprite-UV borders vs drawn-rect borders) set from C#; zeros = identity for Simple images. Uniforms only (Luna
  support for extra canvas shader channels / `PositionAsUV1` is unverified).
- UI shader effects that derive position from `v.vertex` / `unity_ObjectToWorld` land differently in Game view,
  Scene view and Luna (each batches in a different space). Derive positions from **sprite UV** with a per-material
  affine computed on the CPU from `RectTransform.GetWorldCorners` (written only on change, as a fresh `Vector4`).
  Keep `v.vertex` only for `_ClipRect`.

### 5.4 URP projects
Five company projects ship URP (Home-Clean, Craft, Stickii, Obby, mukbang); others are Built-in. URP is possible but
needs a per-project check:
1. Keep the URP assembly alive from an always-compiled script:
   ```csharp
   using UnityEngine.Rendering; using UnityEngine.Rendering.Universal;
   static readonly System.Type[] s_KeepUrpAssembly = { typeof(UniversalRenderPipelineAsset), typeof(LightRenderingMode) };
   ```
2. **URP 2D Renderer** on Unity 6 / URP 17 fails Stage 1 `LP1025` (`ExportCollectionException: UrpAssetsCollection`,
   `Renderer2DData.copyDepthMode not found`): create a `UniversalRendererData` (assign URP's default
   `PostProcessData`), make it the only entry of the URP asset's renderer list, delete the template's
   `Global Light 2D`. Safe when the ad is UGUI/unlit.
3. Read `GraphicsSettings.currentRenderPipeline` (and quality-level assets) before changing materials; don't trust
   docs that say "URP" (GangterCity's notes did, the project is Built-in).
4. Preview every URP shader/feature in a Luna build; no Unity 6 + URP workaround beyond the above is documented.

### 5.5 Opaque textures are exported as JPEG (breaks data textures)
Symptom: a shader reading a data texture (SDF / lookup / packed masks) is misshapen in the build;
`LunaTemp/stage2/assets/textures/<id>.json` shows `"ext": ".jpeg"`. Luna converts any texture whose **imported**
pixels are fully opaque to lossy JPEG, ignoring the `luna.json` rule, reading the import for the **active build
target** (often Standalone, so a WebGL override doesn't help); an explicit `RGB24` default format drops alpha too.
Fix all three: bake alpha 254 into the PNG (`alphaIsTransparency` off); importer **default** format `RGBA32`
uncompressed (sRGB off for data); `luna.json` rule `"compression": "none"`. Verify `ext: ".png"` after the build.

---

## 6. Particles

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| System just activated / restarted with `Play()` shows nothing | Luna doesn't start it that frame | Wait one frame, then `Stop(); Clear(); Play();` on every system | GangterCity |
| Ribbon trails vanish | ParticleSystem **Trails** module unsupported | `TrailRenderer` on a child object | StickiiPA |
| Bursts / sprite-sheet start frame vanish | Unsupported | `rateOverTime` / `rateOverDistance`, or pooled UI emitters | StickiiPA |
| Turning `rateOverTime` 0 → N spawns a ring that collapses inward | `particle-system-1.js` catches up `while (lastSpawn + 1/rate < playTime)`; `lastSpawn` freezes at rate 0 while `playTime` runs, so the backlog spawns pre-aged | Keep `rateOverTime` 0 in the scene and emit from `Update` with `ps.Emit(n)` + fractional accumulator (`acc += rate*dt; n=(int)acc; acc-=n`). Don't `Play()` to reset (it clears live particles) | StickiiPA |
| Wrong facing | Render mode | Billboard | team notes |
| Per-shot `Instantiate` is slow | — | Pool particle objects | GangterCity |
| Invisible on UI | Overlay canvas | §4 | PlayableKit |

---

## 7. Animation, tweens, coroutines

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| Animation (e.g. Die) never plays after `SetTrigger(hash)` | Luna Animator only proven with **string** names | `SetBool("IsMove", v)`, `SetTrigger("Die")`; never `Animator.StringToHash` overloads | Obby |
| Two animations don't blend | Can't play two at once | One at a time via Animator parameters | team notes |
| Animation events don't fire | Target methods must be **public** | Make them public | team notes |
| Humanoid rig doesn't animate | No humanoid avatars | Generic rigs (convert clips) | team notes |
| Coroutine flow silently stops after a nested helper | Luna `Coroutine#MoveNext` pops one level per step: if a nested `IEnumerator`'s **last** `yield return` is another bare `IEnumerator`, both finish together and the whole parent coroutine is marked done | `yield return StartCoroutine(Helper())`, or `Update`-driven state machines/timers. Team notes: limit coroutines | Craft |
| DOTween `Sequence` / chained `OnComplete` lambdas flaky | Closures + sequence bookkeeping | One `DOVirtual.Float(0,1,dur,OnTick)` with a named method computing all segments, or an own `Update`-driven tween | mukbang |
| DOTween breaks at startup: `Cannot read properties of undefined (reading 'OutBack')` | Playworks 7.2.0 gates DOTween (≤1.2.705); a newer DOTween means `DOTween.js` is not in the compiler manifest | In the SDK install the plugin actually builds with (path in `LunaTemp/luna.log`), list `DOTween.js` in `pipeline/templates/LunaCompiler/manifest.json` and remove the `"require": "DOTween"` gate; re-apply after any SDK replacement. Team notes pin DOTween **Pro 1.0.310** | squishy |
| `DG.Tweening` and another tween lib clash | Same extension names | Never `using` both in one file; alias one (`using PT = My.Tweening;`) | mukbang |

---

## 8. Physics

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| `physics3d-0.js`: `Error: Not implemented for the mesh shape` | Rigidbody + `MeshCollider` inertia not implemented | Primitive colliders on dynamic bodies, or post-build patch (§13) | GangterCity |
| Mesh collider scale odd | — | Limit scale, prefer primitives | team notes |
| Child Collider2D under a moving visual stays at old position (even disabled) | Child collider not updated | Manual distance test instead | team notes |
| NavMesh agents don't work | No NavMesh | Own pathing / transform movement | team notes |
| Ragdoll breaks | Bone bodies must be `Interpolate = None`, `Collision Detection = Discrete` | Author in scene, don't change in code | GangterCity |
| Physics-heavy scene diverges / slow | WebGL determinism + cost | Move transforms; don't lean on physics | GangterCity |

---

## 9. Audio

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| **Silent audio**; engine says playing (−91 dB) | Load Type **Compressed In Memory** (with/without Preload) exports as silence | **Decompress On Load** on every clip; never Streaming | squishy |
| `CS0117 AudioListener.pause` | Not in Luna | On pause save `AudioListener.volume` and set `0f`; restore the **saved** value on resume (don't force `1f`, it undoes the player's mute) | all |
| 3D attenuation does nothing | `AudioSource.spatialBlend` stubbed *(reported; no Luna-source proof)* | Don't set `spatialBlend`; set volumes by hand | GangterCity |
| Audio too big | — | Mono 64 kbps (`luna.json` sound rule); BGM 8–10 s looped; trim tails | mukbang, team notes |
| BGM doesn't start | Browser blocks audio autoplay before a user gesture | (Re)start the music loop on the first touch | squishy |

---

## 10. Video

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| `VideoPlayer.Play()` does nothing the first time | Luna creates the `<video>` element asynchronously; `Play()` is a no-op until it exists | Re-call `Play()` each frame until the player reports playing | Fake-Video |
| Browser blocks autoplay | Direct-audio default unmutes the element before first play | Keep the video element muted; play audio through a separate `AudioSource` | Fake-Video |
| Video settings set from script ignored | Some VideoPlayer/render config is read-only at compile time | Configure in the scene (serialized) | Fake-Video |
| Loop clip plays once | `UnityEngine.js` ignores `VideoPlayer.isLooping` | Post-build patch (§13) or manual restart on end | VideoPlayer |
| Loop seek `player.time = 0` throws | Rejected before the player settles | Guard and retry next frame | VideoPlayer |
| Level never starts after video teardown | `UnityEngine.Video` partial; teardown throws | Start the level transition **before** touching video teardown | Home-Clean |
| Won't play on some devices / too large | Luna does **no** transcoding; `.mp4` ships byte-for-byte | H.264 **Baseline**, yuv420p, no B-frames, small resolution (e.g. 540×960, 24 fps, CRF ~20) via ffmpeg | VideoPlayer |

---

## 11. Spine

| Symptom | Cause | Fix | Seen in |
|---|---|---|---|
| Skeleton missing | Binary `.skel` / `.skel.bytes` unsupported | Export **JSON** (`skeletonJSON`) | squishy, team notes |
| Stage 3 `CS1061/CS0246 maskMaterials` in `SkeletonRenderer.cs` | Luna compiles without `BUILT_IN_SPRITE_MASK_COMPONENT` | Keep that code wrapped in `#if BUILT_IN_SPRITE_MASK_COMPONENT` when upgrading Spine | squishy |
| Duplicate GUID / assembly conflicts | Two Spine runtimes (UPM package **and** `Assets/Libs/Spine`) | Exactly one runtime | mukbang |
| White attachment boxes | Texture rule quantizes PMA atlases | `compression: none` override | squishy |
| Spine 4.2 behaves oddly | Luna 7.2.0 ships typemaps for Spine 4.0/4.1 | Test every animation path in a browser build | squishy |
| Multi-texture `SkeletonGraphic` shows corrupt UVs | One canvas renderer for several materials | Enable `allowMultipleCanvasRenderers` | squishy |
| Sprite-attachment conversion warns `Sprite.packed` | Unsupported in Luna | Use atlas-region attachments from the Spine export | squishy |
| Spine materials | URP / Shader Graph shaders unsupported | Built-in Spine shaders or `Sprites/Default` / `UI/Default` / `Unlit/Texture` | — |

---

## 12. Packages that break Luna

| Package / feature | Verdict |
|---|---|
| UniTask, async/await, threads | Banned |
| Odin Inspector | Not in runtime code; exclude `**/Sirenix/` |
| Firebase, Adjust, AppsFlyer, IAP, ad SDKs, notifications | Remove or guard with `#if !UNITY_LUNA` + exclude |
| UI Effect / UIShiny / UIParticle | Avoid |
| Timeline, NavMesh, humanoid, Shader Graph, Visual Scripting, XR, Terrain | Don't depend on them (`LP1032` for unsupported packages) |
| Addressables / AssetBundles | Unusable; clear AssetBundle names or the build errors |
| DOTween **Pro** folders (`Assets/Plugins/Demigiant/DOTweenPro`, `DOTween.XML`) | Conflicting: team notes recommend Pro 1.0.310, compliance tools flag Pro folders. Plain DOTween works with the manifest fix in §7. Test in a build |
| Managed DLLs without source | Not transpiled; exclude and stub |
| PolyFew BatchFew / texture arrays | Never (§5.1) |

---

## 13. Luna engine JS bugs that need a post-build patch

Some bugs live in the generated JS under `LunaTemp/`, not in C#. Do **not** hand-edit `LunaTemp/` for your own bugs;
these two are the only accepted patches and should be applied by an Editor script after **every** build (Luna
rewrites the files). A patcher must string-match the Luna 7.2 code, be idempotent (marker string), and report "no
match" instead of guessing after a Luna upgrade.

- **`physics3d-0.js`**: `Not implemented for the mesh shape` when a Rigidbody has a `MeshCollider`. Patch adds a
  box-approximation inertia from mesh bounds. Without a patch: primitive colliders on dynamic bodies.
- **`UnityEngine.js`**: `VideoPlayer.isLooping` ignored. Patch forwards `loop`, sets `autoplay`/`loop` on element
  creation, and only replays manually (resetting `currentTime`) when native loop is off.

Existing implementations: PlayableKit's `PALunaBuildPatcher` (see the `unity-luna-playablekit` sub-skill), GangterCity
`PlayableSpiderLunaPhysicsPatch`, VideoPlayer `CodexLunaStage4AutoPatch`.

---

## 14. Symptom index

| You see | Go to |
|---|---|
| Black screen | §2 / §5.4 (URP assembly stripped), §5.1 (Texture2DArray, shader), stale build (§2) |
| `X is not a function` | §2 (stripped API → test a Release build; stale build) |
| Button does nothing | §2 (`[Preserve]`), §3 (input module, raycastTarget, tap zones) |
| No sound | §9 (Decompress On Load) |
| TMP missing / blocks / blurry | §4 |
| White boxes around sprites | §4, §11 |
| Field null in build only | §1 (generic Serializable), §2 (Recompile reused stage 1) |
| Build contains old behaviour | §2 stale build; [plugin-ui-and-luna-json.md §4](./plugin-ui-and-luna-json.md#4-toolchain-gotchas) (`luna.json` cached per domain reload) |
| Coroutine flow stops | §7 |
| Effect never plays / ghost UI | §6, §2 (OnDisable / `graphic.enabled`) |
| Video doesn't start / loop | §10, §13 |
| Crash with mesh colliders | §8, §13 |
| Store doesn't open / opens twice / game stuck after store | [luna-api-and-lifecycle.md](./luna-api-and-lifecycle.md) |
