
# Project rules — staging directory

These `.mdc` files belong inside the `outegram/market` repository at:

```
<market-repo>/.cursor/rules/
```

They are staged here because the repo isn't cloned locally yet. When you clone it, run:

```bash
mkdir -p <market-repo>/.cursor/rules
cp ~/.cursor/skills/outegram-market-milestones/project-rules/*.mdc <market-repo>/.cursor/rules/
```

Do **not** copy this `README.md` — Cursor ignores non-`.mdc` files in `.cursor/rules/`, but the folder stays tidier without it.

## Files and their scope

| File | `alwaysApply` | Globs |
|---|---|---|
| `project-overview.mdc` | yes | (all) |
| `branches-and-prs.mdc` | yes | (all) |
| `atomic-actions.mdc` | no | `app/Domain/*/Actions/Atomic/**/*.php` |
| `orchestrators.mdc` | no | `app/Domain/*/Actions/Orchestrators/**/*.php` |
| `migrations-and-ledger.mdc` | no | `database/migrations/**/*.php` |
| `controllers-and-policies.mdc` | no | `app/Http/Controllers/**/*.php` |
| `jobs-and-events.mdc` | no | `app/Jobs/**/*.php`, `app/**/Listeners/**/*.php` |

## Relationship to the skill

These rules are the **file-scoped enforcement layer**. The companion skill (`../SKILL.md`) is the **workflow-scoped planning layer**. Together they cover:

- Skill: "What issue am I working on, what's the gate, which tasks, which acceptance criteria?"
- Rules: "While I'm typing in this specific file, what must I never do and what shape must this code take?"
