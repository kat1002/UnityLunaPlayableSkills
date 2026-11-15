# Unity Luna Playable Skills

Agent skills (Claude Code, OpenCode, Codex and other agents that read `SKILL.md`) for building **Unity playable ads
with Unity Playworks / Luna** (6.4, 7.2). The rules come from the company's PA projects and the PA Handbook
(verified 2026-11-14): what breaks in Luna builds, how to wire the CTA and end card, how to configure `luna.json`,
how to get under 5 MB, and how to ship a creative.

| Skill | Folder | Use it for |
|---|---|---|
| `unity-luna-playworks` | repo root (`SKILL.md`, `references/`) | Core Luna rules, lifecycle/CTA, `luna.json`, build & size, shipping checklist, company projects. **Works without PlayableKit.** |
| `unity-luna-playablekit` | `unity-luna-playablekit/` | Sub-skill for projects that use [PlayableKit](https://github.com/kat1002/PlayableKit) (`com.kat.playablekit` 2.0 package or an old copy inside `Assets/`) |

Related: **[PlayableKit](https://github.com/kat1002/PlayableKit)**, the company's Luna-safe toolkit (lifecycle and
store handling, `TweenLite`, audio, orientation-aware UI, effects, video, editor checkers, automatic Luna build
patches). Install it in Unity from the git URL
`https://github.com/kat1002/PlayableKit.git?path=Packages/com.kat.playablekit#v2.0.0`.

## Layout

```text
unity-luna-playworks/                  core skill (repo root)
├── SKILL.md                           rules, do/don't table, workflows
├── references/
│   ├── constraints-and-gotchas.md     everything that breaks in Luna + symptom index
│   ├── luna-api-and-lifecycle.md      Luna.Unity API, CTA, end card, idle rule, kit-free LunaLifecycle
│   ├── plugin-ui-and-luna-json.md     versions, build stages, luna.json, toolchain errors, size budget
│   ├── shipping-checklist.md          brief → naming → build → test → deliverable zip
│   ├── architecture-and-porting.md    code/scene structure, dual orientation, porting
│   └── company-projects.md            our 13 PA projects, their tools and traps
└── unity-luna-playablekit/            PlayableKit sub-skill
    ├── SKILL.md
    └── references/                    modules-and-api, tools-and-build-patcher, migration
```

## Install

Clone once into your agent's skills folder. The folder name must stay `unity-luna-playworks` (it has to match the
skill `name`).

| Agent | User-level skills folder |
|---|---|
| Claude Code | `~/.claude/skills/` |
| Agents using the shared folder (omp, OpenCode, Codex…) | `~/.agents/skills/` |
| One project only | `<project>/.claude/skills/` or `<project>/.agents/skills/` |

```bash
git clone https://github.com/kat1002/UnityLunaPlayableSkills.git ~/.agents/skills/unity-luna-playworks
```

That is enough to use both skills: the core skill tells the agent to read the sub-skill when the project uses
PlayableKit.

## Link the PlayableKit sub-skill (optional)

Link the sub-skill into the skills folder if you want it to trigger on its own (e.g. "add a PAGameManagerBase
subclass"). The link name must be `unity-luna-playablekit`.

**macOS / Linux**

```bash
ln -s ~/.agents/skills/unity-luna-playworks/unity-luna-playablekit ~/.agents/skills/unity-luna-playablekit
```

**Windows (PowerShell)**: a directory junction, no admin rights needed:

```powershell
New-Item -ItemType Junction `
  -Path   "$env:USERPROFILE\.agents\skills\unity-luna-playablekit" `
  -Target "$env:USERPROFILE\.agents\skills\unity-luna-playworks\unity-luna-playablekit"
```

**Windows (cmd)**

```bat
mklink /J "%USERPROFILE%\.agents\skills\unity-luna-playablekit" "%USERPROFILE%\.agents\skills\unity-luna-playworks\unity-luna-playablekit"
```

### Use one clone for several agents

Clone once, then link the whole skill into each other agent's folder, e.g. for Claude Code:

```bash
ln -s ~/.agents/skills/unity-luna-playworks ~/.claude/skills/unity-luna-playworks
ln -s ~/.agents/skills/unity-luna-playworks/unity-luna-playablekit ~/.claude/skills/unity-luna-playablekit
```

```powershell
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\unity-luna-playworks" `
  -Target "$env:USERPROFILE\.agents\skills\unity-luna-playworks"
```

### Check

```bash
ls ~/.agents/skills/unity-luna-playworks/SKILL.md ~/.agents/skills/unity-luna-playablekit/SKILL.md
```

Restart the agent (skills are loaded at session start); both names should appear in its skill list.

## Use

The agent picks the skill automatically when a task matches its description: Luna-safe C#/UI, `luna.json`, Luna
build or browser errors, CTA/end card, size, shipping. You can also name it: *"use the unity-luna-playworks skill"*.

Example prompts:

- "This button works in the Editor but does nothing in the Luna build."
- "Black screen in the Luna build of the URP project, console says `reading 'Disabled'`."
- "Wire the CTA and end card: store 5–7 s after the last touch, end after 30 s."
- "The build is 6.2 MB, get it under 5 MB."
- "Port this scene from mukbang_pa to a new Unity project."
- "Start a new PA from the PlayableKit Template Scene." (uses the sub-skill)
- "Migrate this project from the old `Assets/PlayableKit` copy to the PlayableKit package." (uses the sub-skill)

Before shipping, ask the agent to run through `references/shipping-checklist.md`.

## Update

```bash
git -C ~/.agents/skills/unity-luna-playworks pull
```

Links point at the clone, so they update with it.

## Uninstall

Remove links first, then the clone. On Windows remove a junction with `cmd /c rmdir <link>`: it deletes only the
link. Don't delete a junction recursively, or you delete the target's files too.

```bash
rm ~/.agents/skills/unity-luna-playablekit          # the link only
rm -rf ~/.agents/skills/unity-luna-playworks        # the clone
```

## Contributing a new Luna trap

Add a row (symptom → cause → fix → project) to `references/constraints-and-gotchas.md` and to the handbook's
`02_Luna_Rules_And_Pitfalls.md`, with the file:line of the fix. Kit-specific behaviour goes in
`unity-luna-playablekit/`; the core skill must stay usable without PlayableKit.
