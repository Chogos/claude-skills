# claude-skills

Claude Code skills for infrastructure, languages, and development best practices.

## Installation

```bash
# Add the marketplace
claude plugin marketplace add Chogos/claude-skills

# Install the plugin
claude plugin install chogos@chogos-skills

# Update to latest
claude plugin marketplace update chogos-skills
claude plugin update chogos@chogos-skills
```

Once installed, skills are available as slash commands:

```text
/writing-python
/developing-backend-services
```

## Skills

| Skill                         | Description                                              |
| ----------------------------- | -------------------------------------------------------- |
| `developing-backend-services` | API design, database patterns, observability, 12-factor  |
| `developing-frontend-apps`    | Component architecture, performance, accessibility       |
| `developing-helm-charts`      | Helm chart conventions, security, values design          |
| `managing-k8s-operators`      | K8s operators: Strimzi, Keycloak, Prometheus, certs      |
| `writing-aws-infrastructure`  | IAM, CloudFormation/CDK, VPC, Lambda, ECS, cost          |
| `writing-dockerfiles`         | Dockerfile conventions, multi-stage builds, security     |
| `writing-golang`              | Go project layout, error handling, concurrency, testing  |
| `writing-python`              | Python project structure, type hints, pytest, async      |
| `writing-rego-policies`       | OPA/Rego policy rules, testing, Kubernetes admission     |
| `writing-rust`                | Rust ownership, error handling, async, traits, testing   |
| `writing-shell-scripts`       | Bash scripting, CLI tools, deployment scripts            |
| `writing-postgres`            | Schema design, queries, migrations — PostgreSQL          |
| `writing-sqlite`              | Schema design, queries, migrations — SQLite              |

## Structure

Each skill uses progressive disclosure — `SKILL.md` is the overview loaded on trigger, with deeper reference files loaded only when needed:

```text
skills/
└── <skill>/
    ├── SKILL.md              # Overview + conventions (loaded on trigger)
    ├── patterns/             # Full examples by use case (loaded on demand)
    │   ├── <pattern>.md
    │   └── ...
    └── <reference>.md        # Cheatsheets, matrices (loaded on demand)
```

## Naming convention

Skill directories use gerund form: `<verb>ing-<noun>` (e.g., `developing-helm-charts`).

## Skill authoring best practices

Full reference: [Skill authoring best practices — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

### Core principles

- **Concise is key.** The context window is a shared resource. Only add context Claude doesn't already have — challenge every paragraph: "Does Claude really need this?"
- **Set appropriate degrees of freedom.** High freedom (text instructions) for heuristic tasks, low freedom (exact scripts) for fragile operations. Match specificity to the task's fragility.
- **Test with all models you plan to use.** What works for Opus might need more detail for Haiku.

### Structure constraints

- **SKILL.md**: Under 500 lines. Overview loaded on trigger — keep it concise with progressive disclosure to pattern files.
- **Pattern files** (`patterns/*.md`): 100–250 lines each. Full examples organized by use case.
- **Reference files** (`*.md` at skill root): Cheatsheets, matrices, lookup tables.
- **One level deep.** All reference files link directly from SKILL.md — no nested chains of references.
- **TOC for long files.** Reference files over 100 lines get a table of contents at the top.

### SKILL.md frontmatter

- `name`: Lowercase letters, numbers, hyphens only. Max 64 chars. Gerund form preferred (`writing-python`, `developing-helm-charts`).
- `description`: Max 1024 chars. Third person. Include both *what* the skill does and *when* to use it. Be specific — Claude uses this to pick the right skill from 100+ candidates.

### Style

- Code examples over prose. Every guideline should have a snippet.
- Bad/good pairs for anti-patterns. Show what's wrong and the fix.
- No duplicate content between SKILL.md and pattern files. SKILL.md summarizes, patterns go deep.
- Consistent terminology — pick one term and use it throughout (not "endpoint" in one place and "route" in another).
- No time-sensitive information. Use an "old patterns" section with `<details>` for deprecated approaches.

### Workflows and feedback loops

- Break complex operations into clear sequential steps with a checklist Claude can track.
- Use feedback loops: run validator → fix errors → repeat. This pattern greatly improves output quality.
- For conditional workflows, guide Claude through decision points explicitly.

### Adding content

- New best practices go in SKILL.md if they fit within the line budget, otherwise create a pattern file and link to it.
- New anti-patterns go in the existing anti-patterns file (if one exists) or as a "Rules" list in SKILL.md.
- Cheatsheet entries should be one-liner examples with a signature and brief description.

### Iteration

- Build evaluations *before* writing extensive documentation — at least three scenarios that test real gaps.
- Develop iteratively with Claude: use one instance to author, another to test. Observe how Claude navigates the skill and refine based on actual behavior, not assumptions.
