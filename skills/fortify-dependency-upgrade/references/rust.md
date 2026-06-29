# Rust Ecosystem Reference — Cargo

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### crates.io Registry

```text
# Confirm a specific version exists (200 = published)
https://crates.io/api/v1/crates/<crate-name>/<version>

# Full version list (JSON)
https://crates.io/api/v1/crates/<crate-name>/versions

# Human-readable crate page
https://crates.io/crates/<crate-name>
```

### GitHub Releases & Changelog

```text
https://github.com/<org>/<repo>/releases
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
```

### Security Advisories

```text
# RustSec Advisory Database
https://rustsec.org/advisories/?search=<crate-name>
https://github.com/RustSec/advisory-db

# GitHub Advisory Database
https://github.com/advisories?query=ecosystem%3Acargo+<crate-name>

# OSV database
https://osv.dev/list?ecosystem=crates.io&q=<crate-name>
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

## Finding Project Files
```bash
# Find all Cargo.toml files
find . -name "Cargo.toml"
```

## Dependency Analysis
```bash
# Show dependency tree
cargo tree

# Show outdated dependencies
cargo outdated            # requires cargo-outdated

# Security audit
cargo audit               # requires cargo-audit

# Show duplicate dependencies
cargo tree --duplicates
```

## Lockfile
- **Cargo**: `Cargo.lock`

## Commands
```bash
# Download dependencies
cargo fetch

# Build with dependencies
cargo build

# Update dependencies
cargo update
```

## Verify Target Version Exists

Check the crates.io registry (a 200 response confirms the version exists):
```text
https://crates.io/api/v1/crates/<crate-name>/<version>
```

Fallback — full version list:
```bash
cargo search <crate-name>           # shows latest version
```

Post-change — verify what was actually resolved in the project:
```bash
cargo tree -i <crate-name>
```

If the version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Dependency Tree

```bash
cargo tree
cargo tree -i package-name          # inverted — who depends on it?
cargo tree -d                       # show duplicate versions
```

## Run Tests

```bash
cargo test
```

## Find Affected Code

```bash
grep -r "use old_crate::" src/ --include="*.rs"
grep -r "old_crate::" src/ --include="*.rs"
```

## Find Affected Tests

Identify every test that imports or exercises the crate being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
# Find source files that use the crate — include tests/ (integration tests) alongside src/
grep -rl "use old_crate::\|extern crate old_crate" src/ tests/ --include="*.rs"

# Run only tests in an affected module
cargo test module_name
cargo test module_name -- --show-output    # include stdout of passing tests
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

One `[patch.crates-io]` entry per crate — duplicate keys are invalid TOML. Patch with a local path:

```toml
[patch.crates-io]
conflict-crate = { path = "../patched-version" }
```

Or with a fork:

```toml
[patch.crates-io]
conflict-crate = { git = "https://github.com/fork/crate", branch = "fix" }
```
