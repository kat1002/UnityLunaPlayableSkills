# Unity Luna Playable Skills

Agent skills for building Unity playable ads with Unity Playworks / Luna, distilled from the company's PA projects
and the PA Handbook (`_PA_Handbook/`, verified 2026-11-14).

| Skill | Path | Use |
|---|---|---|
| `unity-luna-playworks` | `SKILL.md` + `references/` | Core Luna rules, lifecycle/CTA, `luna.json`, build & size, shipping checklist, company projects. **No kit required.** |
| `unity-luna-playablekit` | `unity-luna-playablekit/` | Sub-skill for projects using PlayableKit (`com.kat.playablekit` 2.0 or a vendored 1.x copy) |

The core skill links to the sub-skill when it detects PlayableKit, so installing the core alone is enough. Install
the sub-skill separately too if you want it to trigger on its own for kit-only questions.

## Install

```bash
# core (Claude Code: ~/.claude/skills, other agents: ~/.agents/skills)
git clone https://github.com/kat1002/UnityLunaPlayableSkills.git ~/.agents/skills/unity-luna-playworks

# optional: expose the sub-skill as its own skill
ln -s ~/.agents/skills/unity-luna-playworks/unity-luna-playablekit ~/.agents/skills/unity-luna-playablekit
# Windows (cmd): mklink /J %USERPROFILE%\.agents\skills\unity-luna-playablekit %USERPROFILE%\.agents\skills\unity-luna-playworks\unity-luna-playablekit
```

Update with `git pull` in the clone.

## Contributing a new Luna trap

Add a row (symptom → cause → fix → project) to `references/constraints-and-gotchas.md` and to the handbook's
`02_Luna_Rules_And_Pitfalls.md` with the file:line of the fix. Kit-specific behaviour goes in the sub-skill; the core
skill must stay usable without PlayableKit.
