# Luna API, Lifecycle, CTA & End-Card Reference

Official docs: [Unity Playworks code & API overview](https://docs.lunalabs.io/docs/playable/overview).
Everything here is plain Luna API; no company kit required. If the project uses PlayableKit, its `PALunaManager`
already implements this: see the `unity-luna-playablekit` sub-skill instead of writing your own.

---

## 1. Core `Luna.Unity` APIs

```csharp
Luna.Unity.LifeCycle.GameStarted();      // ONCE, on the player's first interaction
Luna.Unity.LifeCycle.GameEnded();        // AT MOST ONCE, at the final end state (end card)
Luna.Unity.Playable.InstallFullGame();   // store redirect (MRAID / network click-through); does NOT end the game
Luna.Unity.LifeCycle.OnPause  += HandlePause;   // tab hidden, store opened, network pause
Luna.Unity.LifeCycle.OnResume += HandleResume;
Luna.Unity.Analytics.LogEvent("level_start", 0);                          // one name + one int
Luna.Unity.Analytics.LogEvent(Luna.Unity.Analytics.EventType.LevelWon, 1);
```

Wrap Luna calls in `#if UNITY_LUNA` (or one wrapper class) so the Editor stays a clean no-op that logs.

### Rules
1. **`GameStarted()` once** on first pointer down/drag. Guard with a bool.
2. **`GameEnded()` at most once**, only at the final exit. **Never** on a mid-game CTA if the player can return
   from the store and keep playing.
3. **`InstallFullGame()` synchronously inside the click/tap handler.** Popup blockers in iOS/Android WebViews block
   store redirects that happen later (no `Invoke`, coroutine, `DOVirtual.DelayedCall`). One tap can reach several
   listeners (button + full-screen catcher): guard with a **0.25 s** repeat window measured in
   `Time.realtimeSinceStartup`.
4. Mark every method that is only referenced from a `Button.onClick` with `[UnityEngine.Scripting.Preserve]`
   (Luna strips UnityEvent-only methods).
5. Only **one** object (the lifecycle manager) calls `Luna.Unity.*`. Gameplay calls the manager.
6. A no-interaction timeout that ends the ad would send `GameEnded` **before** `GameStarted`; whether Luna accepts
   that order is unverified. Show the end card but skip `GameEnded` (or test it) when the game never started.

**Project managers differ: read the project's own `GoToStore` before wiring it.** Most company `LunaManager`s keep
playing after a mid-game CTA, but GangterCity's `GoToStore()` = `GameEnded()` + `InstallFullGame()` (and its
auto-CTA timer calls it).

**Anti-pattern found in the company (VideoPlayer's old `LunaManager`)**: `GameEnded()` called inside `StartGame()`
and `InstallFullGame()` delayed 0.1 s through `DOVirtual.DelayedCall` (mukbang's doc also says "after 0.1 s"). The
ad reports "ended" at the start and the store call can be blocked. Don't copy it; call synchronously.

---

## 2. Timing rules (CTA, end card, idle)

| Timer | Rule | Source |
|---|---|---|
| End timer | After first tap, play for N s (typical 25–40 s), then show the end card + `GameEnded()` | Home-Clean `currentEndTime` 40 s, BikePlayable 25 s |
| Idle → store | **After the last touch, open the store in > 5 s and < 7 s** (company rule) | team notes `Notes_PlayableAds.xlsx` |
| Intro inactivity | If the player never taps, go to store/end card after a delay (PlayableKit default 15 s) | PlayableKit |
| Safety timeout | Hard cap from load that forces the end card (e.g. 45 s); keep gameplay timers well below it | BikePlayable |
| Auto store at timer end | May be **blocked by the browser** (no user gesture) | squishy |

Because automatic store opens can be blocked, the end card must always contain a **full-screen CTA** fallback.
Replay from the end card and tap spam must not break state.

---

## 3. Pause, resume & store return

| Symptom | Cause | Fix |
|---|---|---|
| After visiting the store and coming back, the game stays paused | In a hidden page `requestAnimationFrame` stops, so `Time.unscaledTime` **does not advance**; a debounce like `if (Time.unscaledTime - last < 0.5f) return;` rejects the single `OnResume` / `OnApplicationFocus(true)` and never retries | Never time-gate resume callbacks with `Time.unscaledTime`. Use `Time.realtimeSinceStartup`; optionally wait ~3 focused frames before completing resume logic |
| Timers jump after tab/store return | One huge `unscaledDeltaTime` | Clamp per-frame step: `Mathf.Clamp(Time.unscaledDeltaTime, 0f, 0.25f)` |
| `AudioListener.pause` CS0117 | Not in Luna | Pause: save `AudioListener.volume`, set `0f`. Resume: restore the **saved** value (forcing `1f` undoes the player's mute) |

---

## 4. Reference implementation (kit-free)

One component per scene. Luna-safe: no lambdas, no LINQ, no `?.`, no `try/catch`, no `PlayerPrefs`.
Adapt the end-card hook to the project; keep the store call synchronous.

```csharp
using UnityEngine;
using UnityEngine.Events;
using UnityEngine.EventSystems;
using UnityEngine.Scripting;

public sealed class LunaLifecycle : MonoBehaviour
{
    public static LunaLifecycle Instance { get; private set; }

    [SerializeField] private float endTime = 30f;         // seconds of play after the first tap
    [SerializeField] private float idleStoreDelay = 6f;   // company rule: > 5 s and < 7 s after the last touch
    [SerializeField] private float safetyTimeout = 45f;   // hard cap from load
    [SerializeField] private UnityEvent onEndCard = new UnityEvent();

    private const float MaxTickSeconds = 0.25f;
    private const float InstallRepeatGuard = 0.25f;

    private bool started, ended, paused, idleStoreOpened;
    private float playTime, idleTime, aliveTime;
    private float lastInstall = -10f;
    private float savedVolume = 1f;

    public bool IsStarted { get { return started; } }
    public bool IsEnded { get { return ended; } }

    private void Awake()
    {
        Instance = this;
#if UNITY_LUNA
        if (EventSystem.current != null && EventSystem.current.GetComponent<StandaloneInputModule>() == null)
            EventSystem.current.gameObject.AddComponent<StandaloneInputModule>();
#endif
    }

    private void OnEnable()
    {
#if UNITY_LUNA
        Luna.Unity.LifeCycle.OnPause += HandlePause;
        Luna.Unity.LifeCycle.OnResume += HandleResume;
#endif
    }

    private void OnDisable()
    {
#if UNITY_LUNA
        Luna.Unity.LifeCycle.OnPause -= HandlePause;
        Luna.Unity.LifeCycle.OnResume -= HandleResume;
#endif
    }

    private void Update()
    {
        if (paused || ended) return;
        float dt = Mathf.Clamp(Time.unscaledDeltaTime, 0f, MaxTickSeconds);
        aliveTime += dt;

        bool touching = Input.GetMouseButton(0) || Input.touchCount > 0;
        if (touching) { StartGame(); idleTime = 0f; idleStoreOpened = false; }
        else if (started) idleTime += dt;

        if (started)
        {
            playTime += dt;
            if (!idleStoreOpened && idleTime >= idleStoreDelay) { idleStoreOpened = true; OpenStore(); }
            if (playTime >= endTime) EndGame(false);
        }
        if (aliveTime >= safetyTimeout) EndGame(false);
    }

    [Preserve]
    public void StartGame()
    {
        if (started) return;
        started = true;
#if UNITY_LUNA
        Luna.Unity.LifeCycle.GameStarted();
#endif
    }

    /// Mid-game CTA: opens the store, keeps playing if the player returns.
    [Preserve]
    public void GoToStore()
    {
        StartGame();
        OpenStore();
    }

    /// Final exit: end card + GameEnded once; optionally open the store (only from a click handler).
    [Preserve]
    public void EndGame(bool openStore)
    {
        if (openStore) { StartGame(); OpenStore(); }
        if (ended) return;
        ended = true;
#if UNITY_LUNA
        if (started) Luna.Unity.LifeCycle.GameEnded();   // GameEnded before GameStarted: unverified, skipped
#endif
        onEndCard.Invoke();
    }

    private void OpenStore()
    {
        float now = Time.realtimeSinceStartup;
        if (now - lastInstall < InstallRepeatGuard) return;
        lastInstall = now;
#if UNITY_LUNA
        Luna.Unity.Playable.InstallFullGame();
#else
        Debug.Log("[Luna] InstallFullGame");
#endif
    }

    private void HandlePause()
    {
        if (paused) return;
        paused = true;
        savedVolume = AudioListener.volume;
        AudioListener.volume = 0f;
    }

    private void HandleResume()
    {
        if (!paused) return;
        paused = false;
        AudioListener.volume = savedVolume;
    }
}
```

Notes:
- `onEndCard` is a serialized `UnityEvent` wired to the end-card show method; that target method needs `[Preserve]`.
- If gameplay must pause, read `LunaLifecycle.Instance` state or add a C# event; don't let gameplay subscribe to
  `Luna.Unity.*` directly.
- Pure session logic (started/ended flags, timers) can live in a plain C# class so it is EditMode-testable without
  `Time` or Luna (mukbang's `LunaManager.Session`).

---

## 5. `[LunaPlaygroundField]` (creative variations)

Exposes serialized/public fields on MonoBehaviours or ScriptableObjects so UA/creative teams can generate variations
in Luna Playground without code changes.

```csharp
[LunaPlaygroundField("Play Duration (s)", 1, "Timer Settings")]
[Range(10, 60)]
[SerializeField] private int endTime = 30;

[LunaPlaygroundField("Player Speed", 2, "Gameplay")]
public float moveSpeed = 10f;
```

- Signature: `[LunaPlaygroundField(string fieldTitle = null, int fieldOrder = -1, string groupName = null)]`;
  stepped sliders: `[LunaPlaygroundFieldStep(0.5f)]`; assets: `[LunaPlaygroundAsset]`.
- After adding/changing these attributes or serialized fields, do a **full** Luna build (not Recompile).
