# claude-skills

Claude Code skills for infrastructure, languages, and development best practices.

## Skills

| Skill                         | Description                                              |
| ----------------------------- | -------------------------------------------------------- |
| `developing-backend-services` | API design, database patterns, observability, 12-factor  |
| `developing-frontend-apps`    | Component architecture, performance, accessibility       |
| `developing-helm-charts`      | Helm chart conventions, security, values design          |
| `writing-aws-infrastructure`  | IAM, CloudFormation/CDK, VPC, Lambda, ECS, cost          |
| `writing-dockerfiles`         | Dockerfile conventions, multi-stage builds, security     |
| `writing-golang`              | Go project layout, error handling, concurrency, testing  |
| `writing-python`              | Python project structure, type hints, pytest, async      |
| `writing-rego-policies`       | OPA/Rego policy rules, testing, Kubernetes admission     |
| `writing-rust`                | Rust ownership, error handling, async, traits, testing   |
| `writing-shell-scripts`       | Bash scripting, CLI tools, deployment scripts            |

## Structure

Each skill uses progressive disclosure — `SKILL.md` is the overview loaded on trigger, with deeper reference files loaded only when needed:

```text
<skill>/
├── SKILL.md              # Overview + conventions (loaded on trigger)
├── patterns/             # Full examples by use case (loaded on demand)
│   ├── <pattern>.md
│   └── ...
└── <reference>.md        # Cheatsheets, matrices (loaded on demand)
```

## Naming convention

Skill directories use gerund form: `<verb>ing-<noun>` (e.g., `developing-helm-charts`).

## Contributing

### Structure constraints

- **SKILL.md**:Under 500 lines. This is the trigger file — keep it concise with progressive disclosure to pattern files.
- **Pattern files** (`patterns/*.md`): 100-250 lines each. Full examples organized by use case.
- **Reference files** (`*.md` at skill root): Cheatsheets, matrices, lookup tables.

### Style

- Actionable, not theoretical. Show the correct pattern, not a lecture on why.
- Code examples over prose. Every guideline should have a code snippet.
- Bad/good pairs for anti-patterns. Show what's wrong and the fix.
- No duplicate content between SKILL.md and pattern files. SKILL.md summarizes, patterns go deep.

### Adding content

- New best practices go in SKILL.md if they fit within the line budget, otherwise create a pattern file and link to it.
- New anti-patterns go in the existing anti-patterns file (if one exists) or as a "Rules" list in SKILL.md.
- Cheatsheet entries should be one-liner examples with a signature and brief description.
