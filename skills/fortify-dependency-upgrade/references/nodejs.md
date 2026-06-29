# Node.js / TypeScript Ecosystem Reference — npm & yarn

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### npm Registry

```text
# Confirm a specific version exists (200 = published)
https://registry.npmjs.org/<package-name>/<version>

# Full package metadata including all versions
https://registry.npmjs.org/<package-name>

# Human-readable release history
https://www.npmjs.com/package/<package-name>?activeTab=versions
```

### GitHub Releases & Changelog

```text
# GitHub releases page (substitute org/repo)
https://github.com/<org>/<repo>/releases

# GitHub API — releases list (JSON)
https://api.github.com/repos/<org>/<repo>/releases

# Raw CHANGELOG in repo root (common filenames)
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
https://raw.githubusercontent.com/<org>/<repo>/main/HISTORY.md
```

### Official Documentation & Migration Guides

```text
# Package docs site (common pattern)
https://<package>.dev/docs/migration
https://<package>.dev/blog/announcing-v<NEW>
```

### Security Advisories

```text
# GitHub Advisory Database
https://github.com/advisories?query=ecosystem%3Anpm+<package-name>

# Snyk vulnerability database
https://security.snyk.io/package/npm/<package-name>

# npm audit — local scan
npm audit
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

## Finding Project Files
```bash
# Find all package.json files
find . -name "package.json" -not -path "*/node_modules/*"
```

## Dependency Analysis
```bash
# List dependency tree
npm ls                    # or: yarn list / pnpm list
npm ls --all              # include dev dependencies

# List outdated
npm outdated              # or: yarn outdated / pnpm outdated

# View package dependencies
npm view package-name dependencies

# Security audit
npm audit                 # or: yarn audit / pnpm audit
```

## Lockfile
- **npm**: `package-lock.json`
- **yarn**: `yarn.lock`
- **pnpm**: `pnpm-lock.yaml`

## Commands
```bash
npm install               # or: yarn install / pnpm install
npm cache clean --force   # or: yarn cache clean
```

## Verify Target Version Exists

Check the npm registry (a 200 response with matching version confirms it exists):
```text
https://registry.npmjs.org/<package-name>/<version>
```

Fallback — full version list:
```bash
npm view <package-name> versions --json
```

Post-change — verify what was actually resolved in the project:
```bash
npm ls <package-name>               # npm
yarn why <package-name>             # yarn
```

If the version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Check Dependencies of Target Package

When upgrading a library, understand what dependencies it bundles and their version ranges to identify potential conflicts or transitive dependency changes:

```bash
# Check what dependencies the target version bundles
npm view <package-name>@<target-version> dependencies

# Check the full dependency tree (including transitive)
npm view <package-name>@<target-version> --json

# Example (real, stable): check what express 4.19.2 depends on
npm view express@4.19.2 dependencies
# Returns: { accepts: '~1.3.8', 'body-parser': '1.20.2', cookie: '0.6.0', ... }

# To find the oldest compatible version of a bundled dependency:
# 1. Check the version range specified by the package
npm view <package-name>@<target-version> dependencies.<dep-name>

# 2. Check historical versions to find when that dependency was introduced
npm view <package-name> versions --json
# Then check dependencies for earlier versions to see version ranges

# 3. Check the package's repository for dependency version history
# Look at package.json changes in git history or releases
```

**Finding oldest compatible versions of bundled dependencies:**

When a target version declares a dependency (e.g. `body-parser: '1.20.2'`):
- An exact version is what that release ships with; a range (`^`/`~`) is what it accepts
- To find the **oldest compatible** version, check:
  1. The package's release notes for a stated minimum version
  2. Its package.json history across the major-version line
  3. The version range syntax (range vs exact pin)

**Understanding version constraints:**
- `3.15.3` = exact version (no flexibility)
- `^3.15.3` = 3.15.3 ≤ version < 4.0.0
- `~3.15.3` = 3.15.3 ≤ version < 3.16.0
- `>=3.0.0` = any version 3.0.0 or higher

## Check Peer Dependencies & Requirements

Verify what the package expects from YOUR environment and identify related packages that need upgrading:

```bash
# Check peer dependencies (what YOU must provide)
npm view <package-name>@<target-version> peerDependencies

# Check engine requirements (Node.js, npm versions)
npm view <package-name>@<target-version> engines

# Example (real, stable): react-dom 18.2.0 requires a matching react
npm view react-dom@18.2.0 peerDependencies   # { react: '^18.2.0' }
npm view next@14.2.3 engines                 # { node: '>=18.17.0' }
```

> Always query the registry for the actual target version — do not rely on remembered version requirements; they change between releases and go stale.

## Plan Coordinated Package Upgrades

When upgrading a package, upgrade its ecosystem family together using the **oldest compatible versions** to minimize cascading breaking changes.

### 1. Find Related Packages That Must Stay in Sync

```bash
# Find all related packages in your project
npm ls | grep <package-pattern>

# Example: Find Prisma family
npm ls | grep prisma    # Shows: @prisma/client, @prisma/adapter-pg, prisma
```

**Common package families:**
- **Prisma**: `prisma`, `@prisma/client`, `@prisma/adapter-*` (same version)
- **React**: `react`, `react-dom` (matching versions)
- **Next.js**: `next`, `@next/*` packages
- **AWS SDK**: All `@aws-sdk/*` (compatible versions)
- **Radix UI**: All `@radix-ui/*` (compatible versions)

### 2. Preview and Execute Upgrade

```bash
# Dry-run to preview conflicts
npm install <package>@<version> --dry-run

# Upgrade all related packages together (example: the Prisma family)
npm install prisma@<version> @prisma/client@<version> @prisma/adapter-pg@<version>

# Verify versions
npm ls prisma @prisma/client
```

### 3. Validate

```bash
npm ls 2>&1 | grep -i "UNMET"    # Check for unmet peer dependencies
npm audit                         # Security audit
npm outdated                      # Check outdated dependencies
```

## Dependency Tree

```bash
npm ls package-name                 # why is it installed?
npm ls --depth=0                    # top-level only
npm dedupe                          # remove duplicate sub-trees

yarn why package-name
```

## Run Tests

```bash
# npm
npm test

# yarn
yarn test

# Jest directly
npx jest

# Vitest
npx vitest run
```

## Find Affected Code

```bash
grep -r "require('old-package')\|from 'old-package'" src/ --include="*.ts" --include="*.js" --include="*.tsx" --include="*.jsx"
grep -r "OldClass\|oldFunction" src/ --include="*.ts" --include="*.tsx"
```

## Find Affected Tests

Identify every test file that imports or exercises the package being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
# Replace <package> with the package name (e.g. "express", "lodash")
grep -rl "require('<package>')\|from '<package>'" \
  tests/ test/ __tests__/ \
  --include="*.test.ts" --include="*.test.js" --include="*.spec.ts" --include="*.spec.js"

# Run only the affected tests — pass the matched file list directly (Jest)
npx jest $(grep -rl "from '<package>'\|require('<package>')" tests/ test/ __tests__/ 2>/dev/null | tr '\n' ' ')

# Run only the affected tests (Vitest)
npx vitest run $(grep -rl "from '<package>'\|require('<package>')" tests/ test/ __tests__/ 2>/dev/null | tr '\n' ' ')
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

```bash
# npm 7+ strict peer deps
npm install --legacy-peer-deps       # bypass (last resort)
npm install --force                  # override (avoid in production)

# npm overrides (package.json)
{
  "overrides": {
    "transitive-package": "2.0.0"
  }
}

# yarn resolutions (package.json)
{
  "resolutions": {
    "transitive-package": "^2.0.0"
  }
}
```
