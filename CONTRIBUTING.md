# Contributing to Fortify Skills

Thank you for your interest in contributing to Fortify Skills! This guide covers the development workflow, conventions, and release process.

## Repository Structure

```
.
├── skills/                    # Skill directories (kebab-case names)
│   ├── fortify-fod/
│   ├── fortify-ssc/
│   ├── fcli-common/
│   └── ...
├── agents/                    # Agent files (*.agent.md)
├── .claude-plugin/            # Claude Code plugin manifest + marketplace
├── .codex-plugin/             # OpenAI Codex plugin manifest
├── .agents/plugins/           # Codex repo-scoped marketplace catalog
├── gemini-extension.json      # Gemini CLI extension manifest
├── version.txt                # Overall version (managed by release-please)
├── release-please-config.json # release-please configuration
└── .release-please-manifest.json
```

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` with required frontmatter (`name`, `description`, `license`, `metadata.version`)
2. Add reference files under `skills/<skill-name>/references/` as needed
3. Add the skill path to `.claude-plugin/plugin.json` `skills` array
4. Update `README.md` skills table
5. Add a test prompt to `test/`

## Adding a New Agent

1. Create `agents/<name>.agent.md` with YAML frontmatter (`name`, `description`, `tools`, `model`) and a short body activating the target skill
2. The install scripts pick it up automatically — no other changes needed

Agents are thin wrappers — the body activates the corresponding skill and enforces scope constraints. The skill carries the workflow knowledge.

## Shared Reference Files

Some reference files are duplicated across skills because skills cannot reference files outside their own directory (Agent Skills spec constraint). When editing a shared file, propagate the change to all copies. The canonical copies and their consumers are tracked in `.github/copilot-instructions.md`.

## Local Testing

### Claude Code

Validate the plugin:
```bash
claude plugin validate .
```

Test locally before publishing:
```bash
claude plugin marketplace add /path/to/skills
claude plugin install fortify-skills@fortify
```

### Gemini CLI

```bash
gemini extensions link /path/to/skills
```

## Versioning

### Overall Version

The overall version (in `version.txt` and JSON plugin/marketplace files) is managed automatically by [release-please](https://github.com/googleapis/release-please). Use conventional commit messages:

| Commit prefix | Version bump | Example |
|---------------|-------------|---------|
| `fix:` | Patch | `fix: clarify upload step wording in fortify-fod` |
| `feat:` | Minor | `feat: add fortify-create-app skill` |
| `feat!:` or any `BREAKING CHANGE:` | Major | `feat!: remove deprecated fortify-legacy skill` |

> **Do not use commit scopes.** Our release tooling does not support a parenthesized scope/component in the commit type — e.g. `feat(fortify-fod): …` is **not** supported. Use the plain type only (`feat:`, `fix:`) and name the affected skill in the description instead (`feat: disambiguate scan/review in fortify-fod`).

release-please automatically bumps these files on release:

| File | Field(s) |
|------|----------|
| `version.txt` | — |
| `.codex-plugin/plugin.json` | `version` |
| `.claude-plugin/marketplace.json` | `metadata.version`, `plugins[0].version` |
| `gemini-extension.json` | `version` |

> **Why `.claude-plugin/plugin.json` has no version field:** Claude Code's update detection uses the `marketplace.json` plugin entry version. For relative-path plugins, if `plugin.json` also declares a version, it silently overrides `marketplace.json` and breaks update detection. Keep the version in `marketplace.json` only.

### Individual Skill Versions

Each skill has its own version in `metadata.version` in the SKILL.md frontmatter. Bump it manually when you change the skill's content.

A CI check on pull requests verifies that `metadata.version` was bumped when skill content files changed. If you only change `SKILL.md` itself (e.g., description tweak), the check still flags it — update the version accordingly.

### Release Flow

1. Develop on a feature branch, using conventional commit messages
2. Bump `metadata.version` in any SKILL.md files you changed
3. Open a PR — CI checks skill version bumps
4. Merge to `main` — release-please opens/updates a release PR with bumped `version.txt`, `CHANGELOG.md`, and JSON files
5. Merge the release PR — release-please creates a GitHub release with a git tag

## Distribution Channels

| Platform | Mechanism | Where version lives |
|----------|-----------|---------------------|
| Claude Code | `.claude-plugin/marketplace.json` | `metadata.version` + `plugins[0].version` |
| OpenAI Codex | `.codex-plugin/plugin.json` | `version` |
| Gemini CLI | `gemini-extension.json` | `version` |
| GitHub Copilot | Skills directory | `metadata.version` in each SKILL.md |
