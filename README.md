# claude-skills

Claude Code skills for infrastructure and scripting best practices.

## Skills

| Skill                    | Description                                          |
| ------------------------ | ---------------------------------------------------- |
| `developing-helm-charts` | Helm chart conventions, security, values design      |
| `writing-rego-policies`  | OPA/Rego policy rules, testing, Kubernetes admission |
| `writing-dockerfiles`    | Dockerfile conventions, multi-stage builds, security |
| `writing-shell-scripts`  | Bash scripting, CLI tools, deployment scripts        |

## Structure

Each skill uses progressive disclosure — `SKILL.md` is the overview loaded on trigger, with deeper reference files loaded only when needed:

```
<skill>/
├── SKILL.md              # Overview + conventions (loaded on trigger)
├── patterns/             # Full examples by use case (loaded on demand)
│   ├── <pattern>.md
│   └── ...
└── <reference>.md        # Cheatsheets, matrices (loaded on demand)
```

## Naming convention

Skill directories use gerund form: `<verb>ing-<noun>` (e.g., `developing-helm-charts`).
