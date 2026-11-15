# Luna Plugin UI, `luna.json`, Build Pipeline & Size Budget

Official docs: [Unity Playworks Plugin UI](https://docs.lunax.app/docs/playable/plugin-ui).
Plugin window: **Window > Unity Playworks** (some installs: `Tools/Unity Playworks Plugin`).

---

## 1. Versions & installation

| Item | Value |
|---|---|
| Playworks 7.2.0 | package `com.unity.playworks.upp` (all company projects but one) |
| Luna 6.4.0 | package `uk.lunalabs.luna` (scoop_pa) |
| Unity | Company requirement docs: **use 2022.3.x** ("Luna doesn't support Unity 6000 yet"). In practice 2022.3.25f1/62f2/62f3 everywhere and Unity 6 (6000.0.79f1) in CraftPlayable and StickiiPA, which needed the URP-2D-renderer fix ([constraints §5.4](./constraints-and-gotchas.md#54-urp-projects)). New projects: 2022.3 |
| Upgrading 6.x → 7.x | Remove `uk.lunalabs.luna` before adding the local `com.unity.playworks.upp`; keep the old SDK folder for rollback |
| Package reference | A **machine-local `file:` path** in `Packages/manifest.json` (e.g. `file:../../../../7.2.0/scripts`). Re-point it to your local `…/7.2.0/scripts` after cloning |
| Windows compiler | VS 2022 Build Tools (**MSBuild 17.14**) + **.NET Framework 4.7 Developer Pack**; path in `luna.json` `unity.scripts.msbuildWin64` |
| SDK copies | Some repos vendor the SDK under `Doc/LunaSDK/<ver>/` (`engine/`, `pipeline/`, `scripts/`, `tools/`): read engine JS there when root-causing runtime bugs |
| Credentials | `pipeline/.auth_*` files hold login tokens: never read, edit, or commit them |

---

## 2. Build stages (`LunaTemp/`)

| Stage | Contents | Debug use |
|---|---|---|
| `LunaTemp/luna.log` | Full compiler / export / bundler log, **including the SDK install path actually used** | First file to read on any failure |
| `stage1/` | Exported scenes (`assets/scenes/0.json`), prefabs, GLSL ES 3.00 shaders | Did serialized values / shader props export? Timestamp moves only on a **full** build |
| `stage2/` | Asset export (`assets/textures/<id>.json` + image) | Texture format/size actually shipped (`ext`, dimensions) |
| `stage3/` | C#→JS (`processed-scripts/`, `engine/unity/bin/UnityEngine.UI.js`, `engine/luna/script1.js`) | Was your script copied? How does Luna implement an API? |
| `stage4/develop/` | Develop build (`index.html`, `js/deserializers.js`, `engine/unity/bin/bridge.js`) served on `http://localhost:8002` | Check timestamps: don't test a stale build |
| `stage4/create-hub/*_build.zip` | Production bundle for Playworks / networks | Measure size here |

- **Develop** build: full engine + debug info; fast; does **not** reproduce minifier-stripping bugs.
- **Release** (minified) build: the deliverable; "X is not a function" stripping bugs appear only here.
- **Recompile** = stage 3 only, reuses the old stage-1 scene export. After adding/renaming/retyping any serialized
  field, component, or `[LunaPlaygroundField]`, run a **full** build.
- `LunaTemp/` and `LunaTemp.stale-*` belong in `.gitignore`. Never commit or hand-edit them (except the two
  documented post-build patches, [constraints §13](./constraints-and-gotchas.md#13-luna-engine-js-bugs-that-need-a-post-build-patch)).

---

## 3. `luna.json` (project root)

The plugin UI tabs (`Settings`, `Assets`, `Code`, `Build & Upload`, `Size Breakdown`) all read/write `luna.json`.

### 3.1 Settings
- **Exactly one enabled scene**: `unity.scenes` = your scene only, `startupScene: 0` (index inside that list);
  everything else in `unity.disabledScenes`. If `unity.scenes` is empty, Luna uses `EditorBuildSettings`: then make
  yours the only enabled scene there. When filled, `luna.json` wins.
- **`creativeName`**: fresh per creative, never reused (output is named after it). Company standard:
  `<Game>_<Theme>_PA_<NN>_<ddMMyy>_<Author>` (see [shipping-checklist.md](./shipping-checklist.md#naming)).
- **IDs**: when copying `luna.json` to a new project/game, reset `projectId`, `applicationID`, `environmentID`,
  `creativeName`, `pgServer` (get new values from the Luna dashboard). Don't share IDs between games;
  `applicationID: 0` means the build is tied to no app.
- Build flags seen in company projects: `minify: true` (release), `enableMecanimWASM: true`,
  `disableAntiAliasing: true`, `enableLunaCompilerV2: false`, `graphicsConstraint: 24`, realtime shadows off,
  batching off, `postprocesses.ssao.isActive: false`, optional half-precision meshes/animations to save size.
  `forceBase64`: set per network requirement (`true` = single self-contained HTML, ~33% larger assets; `false` for
  zip / Create Hub). `packages` (networks) typically `["default", "ironsource", "facebook"]`.
- `playerPrefs: true` exists but don't rely on persistence.

### 3.2 Assets
- **Excludes** (`unity.assets.excludes`): `**/Editor/`, ad/analytics SDK folders (`FacebookSDK`, `AdMobPlugin`,
  `IronSource`, `AppsFlyer`, `Firebase`, `UnityAds`, `UnityPurchasing`), `Stubs/*`, unused TMP defaults
  (`EmojiOne.png`, `LiberationSans SDF*.asset`), unreferenced large textures from Size Breakdown.
- **Force-includes** (`unity.assets.includes`): `Assets/TextMesh Pro/Resources/TMP Settings.asset`
  (+ `Assets/Resources/DOTweenSettings.asset` if DOTween). Includes override excludes.
- Everything referenced by the scene, its prefabs, ScriptableObjects and managers is exported: remove unused
  references, reference sprites explicitly (a folder `DefaultAsset` pulls the whole folder in), and give each
  creative its own catalog ScriptableObject holding only its assets.
- **Texture rule** default: max **512**, `png32`, `quantSize` quality **55–65**. Add **explicit per-asset/folder
  overrides** (`compression: none`, larger `maxWidth/maxHeight`) for Spine/PMA atlases, backgrounds, data textures
  and anything with white boxes or blur. Regenerating `luna.json` from defaults drops these overrides: re-add them.
- **Audio rule**: mono **64 kbps**; use the plugin's Sound Timeline cropping to trim tails.
- **Video**: no transcoding, ships byte-for-byte; pre-encode with ffmpeg ([constraints §10](./constraints-and-gotchas.md#10-video)).

### 3.3 Font SDF atlas overrides get regenerated
On every export the plugin (decompiled `EditorScripts.dll`, 7.2.0) deletes a font asset's own texture override and
recreates it from the texture default, or from the **longest folder override whose name prefixes the asset path**,
with compression forced to `none`. A size typed on the font asset's own row is reset each build. Fix: add a
**folder** override (e.g. `"name": "Assets/Fonts/Baloo/"`, `maxWidth/maxHeight: 1024`) or raise the default;
verify `LunaTemp/stage2/assets/textures/<id>.png` dimensions. English-only: regenerate a static ASCII atlas at
256×256 or 512×512 instead (saves ~120 KB+).

### 3.4 Code
- **Luna transpiles every non-excluded C# script**, used by the scene or not. `unity.scripts.excludes`: `**/Editor/`,
  `**/Luna/RuntimeScripts/`, ad/analytics SDKs, `**/Obi/`, `**/Sirenix/Odin Inspector/`,
  `**/StompyRobot/SRDebugger/`, unused game modes. Excluding unused modes can cut megabytes of JS.
- Managed DLLs without source are not transpiled: exclude them and add C# stubs under `Stubs/`.
- `unusedModules` / `unusedClasses` / `unusedMethods`: when porting `luna.json` to a **new** project, clear them so
  Luna regenerates (or un-force `physics3d` / `reflection` if needed).

---

## 4. Toolchain gotchas

| Symptom | Cause | Fix |
|---|---|---|
| `luna.json` edits revert; build compiles excluded files (CS1026/CS0101/CS0708, UniTask CS1031); "compiled all of Assets" | The plugin reads `luna.json` **once per domain reload**, caches it in statics, and writes it back on build/UI change. Reopening the window does not reload it | After editing `luna.json` on disk (by hand, script or agent), **restart Unity or force a domain reload** before building, then re-open `luna.json` after the build to confirm the edit stuck. GangterCity has `Tools/Luna/Refresh From Disk` (reflection bridge) |
| Include/exclude toggles don't stick | Same cache | Re-select the build profile to regenerate excludes |
| Build fails but the Plugin window shows no compiler errors | `Diagnostics.csproj.template` in the SDK doesn't define `UNITY_LUNA`, so errors inside `#if UNITY_LUNA` only reach process output | Read `LunaTemp/luna.log`; add `UNITY_LUNA` to the template's defines in the SDK install used for building |
| Stage 4 `EBUSY: resource busy or locked, unlink '…/stage4/develop/engine/unity/bin/bridge.js'` (`PACKAGE_BUNDLE`) | Unity's Luna preview web server (port 8002) holds the file | Stop the Luna preview / close the Playground window (Console: "Stopping Luna web server"); delete `LunaTemp/stage2..4` if needed (stage1 = Unity export); restart Unity if the lock lingers |
| `LP1032: Package(s) not supported in Luna` | Unsupported packages in `manifest.json` (Timeline, XR, Terrain, Vehicles…) | Grep for references; remove unused packages |
| Stage 1 `LP1025` `Renderer2DData.copyDepthMode` | URP 2D Renderer on Unity 6 | [constraints §5.4](./constraints-and-gotchas.md#54-urp-projects) |
| Login fails `UNKNOWN: unknown error, open '…/.auth_credentials.user'` | Another process holds the token file | Retry (6.4.0 had a retry patch; 7.2.0 does not) |
| MSBuild not found | Build Tools / .NET 4.7 dev pack missing | Install; set `unity.scripts.msbuildWin64` |
| Packages fail to resolve on another machine | `file:` path to the SDK | Re-point `com.unity.playworks.upp` |
| Browser: `Cannot set properties of undefined (setting 'fontRatio')` at `startGame` | Preview opened before `script1.js` finished writing (~5 s after build) | Hard-refresh (Ctrl+F5) after the build completes |
| TMP / DOTween missing at runtime | Library JS not in the compiler manifest | [constraints §4 / §7](./constraints-and-gotchas.md#7-animation-tweens-coroutines) |

---

## 5. Size budget

- No network limit is recorded in company repos. The common Google Ads HTML5 limit is **5 MB zip**; verify the
  current spec per network before relying on it. Working target: production zip and HTML **≤ 5 MB**.
- Shipped Home-Clean deliverables: HTML 3.2–5.1 MB, correct zip 2.4–3.8 MB.
- Plugin **Size Breakdown → Build & Estimate size** shows JS / textures / audio / fonts / meshes / JSON.
- Measure the real zip:
  ```powershell
  python -c "import zipfile,sys;z=zipfile.ZipFile(sys.argv[1]);print(round(sum(i.compress_size for i in z.infolist())/1048576,3),'MB')" "LunaTemp/stage4/create-hub/<build>.zip"
  ```

| Culprit | Fix |
|---|---|
| JS > 1.5–2.5 MB | Exclude unused scripts/folders; `minify: true` |
| Textures > 1.5 MB | Cap ≤ 512 (lower big overrides, e.g. a 2048→1024 atlas saved ~0.7 MB), quality 65→55, exclude unused sprites |
| Fonts > 150 KB | Static ASCII-only SDF atlas 256×256 |
| Audio | Mono 64 kbps, BGM 8–10 s loop, trim tails |
| Video | Re-encode smaller (H.264 Baseline, lower resolution/bitrate) |
