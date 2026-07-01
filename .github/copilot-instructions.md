# Fortify Skills Repository

This is the public OpenText Fortify AppSec skills repository. It contains AI assistant
skills following the Agent Skills standard, designed to teach AI coding assistants how
to use Fortify effectively.

## Structure

```
.
├── .claude-plugin/
│   ├── plugin.json            # Claude Code plugin manifest (no version field — intentional)
│   └── marketplace.json       # Claude Code marketplace catalog + version authority
├── .codex-plugin/
│   └── plugin.json            # OpenAI Codex plugin manifest + version field
├── .agents/
│   └── plugins/
│       └── marketplace.json   # OpenAI Codex repo-scoped marketplace catalog
├── gemini-extension.json      # Gemini CLI extension manifest + version field
├── skills/                    # The skills (Agent Skills spec)
├── agents/                    # AI assistant agents (*.agent.md)
├── version.txt                # Overall version (managed by release-please)
├── README.md                  # End-user documentation
├── CONTRIBUTING.md            # Developer/contributor guide
└── LICENSE
```

## Conventions

- Skill directories use kebab-case names
- SKILL.md frontmatter fields: `name`, `description`, and optional `disable-model-invocation`, `user-invocable`, `allowed-tools`, `context`, `argument-hint`
- Reference files are linked from SKILL.md and loaded on demand (progressive disclosure)
- Keep descriptions concise — they're loaded at startup for skill matching
- Skills must be assistant-agnostic: work with Claude Code, GitHub Copilot, Cursor, Gemini CLI, etc.
- Skills teach workflows using FCLI commands — the AI assistant executes them
- Leverage best practices where appropriate (https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

### Workflow terminology

Use **Step** for all numbered workflow divisions — in `SKILL.md`, `agent.md`, and reference files. Do not use "Phase", "Stage", or "Phrase" as structural labels.

| Level | Term | Format | Where |
|-------|------|--------|-------|
| Primary | **Step** | `## Step N: Name` | Top-level workflow divisions in `SKILL.md` and `agent.md` |
| Secondary | **Step (lettered)** | `### Step Na: Name` | Sub-actions within a Step |
| Gates | **Step N gate** | `### Step N gate` or `### Step N → Step N gate` | Checklists between steps |

- Step numbering **resets per file** — each file (SKILL.md, each reference file) starts at Step 0 or Step 1.
- Cross-file references use the file name for disambiguation: "Step 2 in `references/sast-dast-fix-planning.md`".
- Gates between steps within a reference file use `### Step N → Step N gate`.

## Reference path rules (per Agent Skills spec)

Each skill must be self-contained. References from `SKILL.md` and from files inside `references/`:

- **One level deep only.** Use `references/<file>` — never `references/<subdir>/<file>`.
- **No `../` references.** Never reach up into a sibling skill or any parent directory.
- **Cross-skill composition** is by skill name, not by path. To pull in another skill, instruct the assistant to activate that skill (e.g., "activate the `fortify-fod` skill"), not to load `../fortify-fod/SKILL.md`.

### Duplicated reference files — keep in sync

Because skills cannot reach across directories, some reference files are physically duplicated. The canonical copy lives in `skills/fcli-common/references/`; consumer skills carry verbatim copies. **Any edit to a canonical file must be propagated to every consumer copy in the same commit.**

| Canonical file | Consumer copies (must match byte-for-byte) |
|----------------|---------------------------------------------|
| `skills/fcli-common/references/fcli-query-output.md` | `skills/fortify-fod/references/fcli-query-output.md`, `skills/fortify-remediate/references/fcli-query-output.md`, `skills/fortify-ssc/references/fcli-query-output.md` |
| `skills/fcli-common/references/fcli-install.md` | `skills/fortify-create-app/references/fcli-install.md`, `skills/fortify-fod/references/fcli-install.md`, `skills/fortify-remediate/references/fcli-install.md`, `skills/fortify-ssc/references/fcli-install.md` |
| `skills/fcli-common/references/output-formats.md` | `skills/fortify-fod/references/output-formats.md`, `skills/fortify-ssc/references/output-formats.md` |
| `skills/fcli-common/references/mutating-operations.md` | `skills/fortify-fod/references/mutating-operations.md`, `skills/fortify-ssc/references/mutating-operations.md` |
| `skills/fortify-fod/references/resolving-release.md` | `skills/fortify-remediate/references/resolving-release.md` |
| `skills/fortify-ssc/references/resolving-appversion.md` | `skills/fortify-remediate/references/resolving-appversion.md` |

When editing any file in the left column, copy it over every file in the right column. When adding a new shared reference, list it here.

## Fortify Product Architecture

Fortify has two product versions with the same underlying scanning technology but different interfaces and commands:

- **FoD** (Fortify on Demand) — SaaS
- **On-premise / SSC** (Software Security Center) — self-hosted

Skills often need to branch based on which version the customer uses.

### CLI Tools

- **FCLI** — primary CLI for both FoD and SSC, but with different command sets for each. Self-documenting via `fcli help`. Source: https://github.com/fortify/fcli
- **sourceanalyzer** and **scancentral** — used for some tasks instead of FCLI (details TBD)

When developing skills, you can invoke `fcli` directly to explore its commands and options.

## Versioning

Overall version is managed by [release-please](https://github.com/googleapis/release-please) via conventional commits. Individual skill versions are maintained in `metadata.version` in each SKILL.md frontmatter.

- CI checks that `metadata.version` is bumped when skill content changes — but only once per release cycle at minimum. The check compares against the **last published release tag**, so: bump the version the first time you change a skill after a release. Within the same release cycle, re-bump if the nature of subsequent changes warrants a higher semver level (e.g. a patch bump was done first but a new feature is added later — bump to the next minor). New skills (not yet in any release) must declare version `1.0.0` and must **not** bump the version until after the first release — CI enforces this.
- release-please auto-bumps `version.txt` and all JSON version fields on release
- **Do not use commit scopes.** The release tooling does not support a parenthesized scope/component (e.g. `feat(fortify-fod): …` is **not** supported). Use the plain type only (`feat:`, `fix:`) and name the affected skill in the description.

See `CONTRIBUTING.md` for full versioning details.
