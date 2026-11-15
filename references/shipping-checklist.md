# New Concept → Shipped Creative Checklist

Company workflow for one playable-ad concept, from brief to deliverable. Rules referenced here live in
[constraints-and-gotchas.md](./constraints-and-gotchas.md); `luna.json` details in
[plugin-ui-and-luna-json.md](./plugin-ui-and-luna-json.md).

**Principle: never build from scratch: copy the closest existing scene.** Every working PA carries solved Luna
problems (URP assembly keep, `[Preserve]`, input module, video retries). A fresh scene throws that away.

---

## 1. Start a concept

1. **Brief**: reference video/store link/assets; write down flow in 3–6 steps, orientation(s), CTA moment, target
   networks, end-card timing, duration.
2. Reference research on a live competitor ad HTML: extract its textures/audio with **PA_Assets_Extractor**
   (`node extract.js ad.html`). **Reference only: never ship extracted assets.**
3. **Choose the project**: same genre/mechanic → that project ([company-projects.md](./company-projects.md)). New
   mechanic → a clean base (PlayableKit `Template Scene` if the team uses the kit).
4. Do the project's setup first (re-point the machine-local Luna `file:` path).
5. **Copy the closest scene** (Ctrl+D / Save As); never edit the original; keep its folder convention.

## 2. Naming

- Scene: follow the project's convention (`ScenePA_Mode_<Mode>[_Variant]`, `Playable_Ads/PA_<Name>`,
  `<Game>_PA_<NN>`).
- <a id="naming"></a>**`creativeName`**: `<Game>_<Theme>_PA_<NN>_<ddMMyy>_<Author>`, date = build day `ddMMyy`
  (e.g. `Mukbang_Halloween_Collab_PA_03_271026_Phuc`). Never reuse one.
- Deliverable files: `<Game>_<Theme>_<Variant>_<ddMMyy>.{html,zip}` (e.g. `HomeClean_Car_Wash_050826`).
  Per-network prefixes when packaging: `GGA_` (Google Ads), `Minter_` (Mintegral), `FB_`, `Tiktok_`, `Applovin_`
  (company `RenameTool.html` applies them from a zip with one folder per network).

## 3. Gameplay wiring

- [ ] One lifecycle manager in the scene: `GameStarted` on first touch, `GameEnded` once, synchronous
      `InstallFullGame` ([luna-api-and-lifecycle.md](./luna-api-and-lifecycle.md)).
- [ ] End timer + idle rule (store **> 5 s and < 7 s after the last touch**) + full-screen CTA on the end card.
- [ ] Tutorial hand on the first interactive step, hidden on first touch (reuse the project's hand prefab).
- [ ] Audio: all clips **Decompress On Load**; BGM 8–10 s looped; mute routed through one place.
- [ ] Portrait **and** landscape unless the brief says one.
- [ ] Remove unused UI, prefabs, ScriptableObjects and manager references (everything referenced is exported).

## 4. Before build

- [ ] Scene saved, **exactly one enabled scene**, `startupScene: 0`, fresh `creativeName`, IDs not shared with
      another game.
- [ ] Run the project's checkers (which exist differs per project:
      [company-projects.md](./company-projects.md#editor-tools-by-project)). Fix every Error, understand every Warning.
      Without tools, grep runtime scripts for: `PlayerPrefs`, `Resources.Load`, `JsonUtility`, `System.Reflection`,
      `try`, `goto`, `async`, `UniTask`, `OnMouseDown`, `RuntimeInitializeOnLoadMethod`, `GameObject.Find`,
      `FindObjectOfType`, `= new();`, `is not`, ` or `/` and ` patterns, `[^`, `Array.Empty<`, `StringToHash`,
      `AudioListener.pause`, `GetComponentInParent<.*>(true)`, LINQ `using System.Linq`.
- [ ] No `LayoutGroup` / `ContentSizeFitter` / `AspectRatioFitter` / `ScrollRect` / `RectMask2D` left in the scene.
- [ ] No Spine `.skel.bytes`; no URP Lit / Shader Graph materials; no DOTween Pro folders unless proven.
- [ ] No editor-only code at runtime; no debug cheats, `Debug.Log` spam, FPS counters, test objects.
- [ ] No persistence, no ad/analytics SDKs, no Odin/Firebase/Adjust/UniTask.
- [ ] Textures: overrides for atlases/backgrounds/data textures; no sprites Luna would halve inside atlases.
- [ ] TMP: static atlas, `Mobile/Distance Field`, 3.0.7 shaders, `TMP Settings.asset` force-included.

## 5. Build & test

1. Build **Develop** (full build if serialized fields changed since the last one). Stop the Luna preview server first
   if rebuilding.
2. Confirm post-build patches ran if the project has a patcher (physics3d, video loop).
3. Open in a desktop browser **and** a phone-size window; DevTools console must have no red errors
   (`is not a function`, null refs).
4. Test:
   - [ ] First tap starts the game, tutorial hand disappears.
   - [ ] Portrait **and** landscape, rotating while playing; tap zones cover the screen in both.
   - [ ] Mute/unmute silences everything; with sound on, BGM and SFX actually play.
   - [ ] Every CTA (in-game + end card) opens the store; one tap opens it once.
   - [ ] End timer fires; idle path opens the store 5–7 s after the last touch.
   - [ ] Store → return: game resumes, timers don't jump.
   - [ ] Replay from end card and tap spam don't break state.
5. Build **Release** (minified) and repeat: stripped-API bugs appear only here.
6. Check size ([plugin-ui-and-luna-json.md §5](./plugin-ui-and-luna-json.md#5-size-budget)).

## 6. Deliverable

1. Final build → HTML named after `creativeName`; rename to the deliverable name.
2. **The zip must contain exactly the single `.html` at its root.** `unzip -l X.zip` must list one entry. (Three
   past Home-Clean zips were double-zipped: zip inside zip.)
3. Record the creative (concept, scene path, `creativeName`, size, flow, CTA timing, Luna issues + fix, deliverable
   path) in the team's PA record / handbook; add any new Luna trap with the file:line of the fix.

## 7. Repo hygiene

| Don't | Why |
|---|---|
| Commit `LunaTemp/`, `LunaTemp.stale-*`, `Library/`, `*.csproj` | Generated, huge (one repo tracked 1,326 LunaTemp files) |
| Commit keystores, `.auth_*`, credentials | Secrets |
| Share `applicationID` / `projectId` across games | Wrong app attribution |
| Hand-edit `LunaTemp/` as a quick fix | Overwritten on build; only the two automated patches are allowed |
| Test only in the Editor or only Develop | Luna-only and minifier-only bugs |
| Leave stray files / hardcoded machine paths in tools | Breaks for the next person |
