# Luna API, Lifecycle & Store Flow Reference

Official documentation: [Unity Playworks Code & API Overview](https://docs.lunalabs.io/docs/playable/overview)

---

## 1. Core `Luna.Unity` APIs

### 1.1 `Luna.Unity.LifeCycle`
Manages the lifecycle of the playable ad across ad networks (Unity Ads, IronSource, AppLovin, Mintegral, Facebook, Vungle, TikTok).

```csharp
// 1. Call ONCE on the player's first interaction (pointer down / drag)
Luna.Unity.LifeCycle.GameStarted();

// 2. Call AT MOST ONCE when the playable reaches its final end state (timer expired, won/lost, end card shown)
Luna.Unity.LifeCycle.GameEnded();

// 3. Subscribe to pause/resume (e.g., user minimizes app, opens store, or switches tabs)
private void OnEnable()
{
    Luna.Unity.LifeCycle.OnPause += HandlePause;
    Luna.Unity.LifeCycle.OnResume += HandleResume;
}

private void OnDisable()
{
    Luna.Unity.LifeCycle.OnPause -= HandlePause;
    Luna.Unity.LifeCycle.OnResume -= HandleResume;
}
```

> [!IMPORTANT]
> - **Never call `GameEnded()` on mid-game CTA clicks** if the player is allowed to return from the store and keep playing. Reserve `GameEnded()` for the final exit (`EndGame`).
> - Guard `GameStarted()` and `GameEnded()` with boolean flags (`IsStarted`, `IsEnded`) so they never fire more than once per session.

---

### 1.2 `Luna.Unity.Playable` (Store / CTA Redirection)

```csharp
// Triggers the ad network's app store redirection (MRAID / network click-through)
Luna.Unity.Playable.InstallFullGame();
```

- **Call synchronously inside the user's click/tap frame**: Browser and WebView popup blockers on iOS/Android can block store redirects that happen asynchronously after a delay.
- **Repeat Guard**: One player click can trigger multiple listeners (e.g., button `onClick` + full-screen overlay). Guard `InstallFullGame()` with a `0.25s` cooldown (`InstallRepeatGuardSeconds = 0.25f`).

---

### 1.3 `Luna.Unity.Analytics` (Playable Insights)

```csharp
#if UNITY_LUNA
Luna.Unity.Analytics.LogEvent("level_start", 0);
Luna.Unity.Analytics.LogEvent(Luna.Unity.Analytics.EventType.LevelWon, 1);
#endif
```

- Wrap direct `Luna.Unity.Analytics.LogEvent` calls in `#if UNITY_LUNA` or route through an abstraction like `PlayableAnalytics` (`IAnalyticsSink`) so the Editor remains a clean no-op (or logs `ANALYTICS <name> <value>` to console).
- Luna's custom event signature takes **one event name (`string` or `EventType`) and one `int` value**.

---

### 1.4 `[LunaPlaygroundField]` & `[LunaPlaygroundAsset]` (Creative Variations)

Expose `[SerializeField]` or `public` fields on `MonoBehaviour` or `ScriptableObject` classes so UA/creative teams can tweak gameplay parameters and generate unlimited ad variations in Unity Playworks / Luna Playground without rebuilding code:

```csharp
[LunaPlaygroundField("Play Duration (s)", 1, "Timer Settings")]
[Range(10, 60)]
[SerializeField] private int currentEndTime = 25;

[LunaPlaygroundField("Player Speed", 2, "Gameplay")]
public float moveSpeed = 10f;
```

- Signature: `[LunaPlaygroundField(string fieldTitle = null, int fieldOrder = -1, string groupName = null)]`
- For stepped sliders, pair with `[LunaPlaygroundFieldStep(0.5f)]`.
- Note: After adding or changing `[LunaPlaygroundField]` attributes or serialized fields, perform a **full Luna build** (not incremental).

---

## 2. Battle-Tested `LunaManager` Architecture

In this project (`Assets/Scripts/Luna/LunaManager.cs`), lifecycle state is separated into a pure C# `LunaManager.Session` class (testable in EditMode without `Time` or `Luna` dependencies) driven by `LunaManager.Update()`:

| Method | Behavior |
|---|---|
| `LunaManager.StartGame()` | Calls `Session.Start()`. On first call only, invokes `Luna.Unity.LifeCycle.GameStarted()` and starts the countdown timer (`currentEndTime`). |
| `LunaManager.GoToStore()` | Calls `StartGame()` + `OpenStore()` (`Luna.Unity.Playable.InstallFullGame()`). **Does not call `GameEnded()`**, allowing the player to keep playing if they return from the store. |
| `LunaManager.EndGame(bool openStore)` | If `openStore`, ensures `StartGame()` ran. Calls `Session.End()` -> invokes `Luna.Unity.LifeCycle.GameEnded()` **once**. If `openStore`, calls `OpenStore()`. |
| `LunaManager.ClampTickSeconds(dt)` | Clamps `Time.unscaledDeltaTime` to `MaxTickSeconds = 0.25f` so a tab switch or store return does not jump countdown timers in a single frame. |

---

## 3. Store-Return & Pause/Resume Gotchas

### 3.1 `Time.unscaledTime` Freezes While Page Is Hidden
- **Symptom**: After clicking a CTA button to visit the App Store and returning to the playable, the game stays permanently paused (until alt-tabbing again).
- **Root Cause**: In a browser/WebView, `requestAnimationFrame` stops ticking while the page is hidden, so `Time.unscaledTime` **does not advance**. Any debounce check like `if (Time.unscaledTime - _lastStoreTime < 0.5f) return;` rejects the single `OnResume` / `OnApplicationFocus(true)` callback and never retries.
- **Fix**:
  1. Never time-gate lifecycle resume callbacks with `Time.unscaledTime` across a hide/blur.
  2. Use **`Time.realtimeSinceStartup`** (wall-clock) and wait **3 focused frames** (`TryCompleteReturn`) before completing resume logic.
  3. Add a fallback that re-enables tap/timeout resume after 30 visible frames if no explicit resume event arrives.
  4. Always clamp per-frame timer deltas (`Mathf.Clamp(Time.unscaledDeltaTime, 0f, 0.25f)`).
