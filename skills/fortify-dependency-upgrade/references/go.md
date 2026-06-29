# Go Ecosystem Reference

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### Go Module Proxy & Registry

```text
# Confirm a specific version exists (200 = published)
https://proxy.golang.org/<module-path>/@v/<version>.info

# Full version list
https://proxy.golang.org/<module-path>/@v/list

# Module page on pkg.go.dev
https://pkg.go.dev/<module-path>@<version>
```

### GitHub Releases & Changelog

```text
https://github.com/<org>/<repo>/releases
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
```

### Security Advisories

```text
# Go vulnerability database
https://pkg.go.dev/vuln/?q=<module-path>
https://osv.dev/list?ecosystem=Go&q=<module-path>

# GitHub Advisory Database
https://github.com/advisories?query=ecosystem%3Ago+<module-name>
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

## Finding Project Files
```bash
# Find all go.mod files
find . -name "go.mod"
```

## Dependency Analysis
```bash
# Show dependencies
go list -m all

# Show dependency graph
go mod graph

# Show why dependency needed
go mod why -m module-path

# Check for updates
go list -u -m all
```

## Lockfile
- **go mod**: `go.sum` (checksums only)

## Commands
```bash
go mod download           # download dependencies
go mod tidy               # clean unused
go mod verify             # verify checksums
```

## Verify Target Version Exists

Check the Go module proxy (a 200 response confirms the version exists):
```text
https://proxy.golang.org/<module-path>/@v/<version>.info
```

Fallback — full version list:
```bash
go list -m -versions module/path
```

Post-change — verify what was actually resolved in the project:
```bash
go mod why module/path
go list -m all | grep module/path
```

If the version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Dependency Tree

```bash
go mod why module/path
go mod graph | grep module/path
go mod graph | grep direct          # direct deps only
```

## Major Version Modules

Go major versions ≥ v2 require a module path suffix:

```bash
# v1 → v2: import path changes
go get module/path/v2@v2.0.0
```

## Compile

```bash
go build ./...
go vet ./...                        # static analysis
```

## Run Tests

```bash
go test ./...
go test ./... -run TestName
go test ./... -race                 # data race detector
go test -v ./pkg/...                # verbose
go test -count=1 ./...              # disable test caching
```

## Find Affected Code

```bash
grep -r "\"old/module/path\"" . --include="*.go"
grep -r "old\.Symbol\|old\.Function" . --include="*.go"
```

## Find Affected Tests

Identify every test file that imports or exercises the module being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
# Find test files that import the module
grep -rl "\"old/module/path\"" . --include="*_test.go"

# Run only packages that contain affected tests
go test ./pkg/... -run TestAffectedName -v
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

```bash
# Replace a dependency (e.g., fork or patched version)
go mod edit -replace old/module=new/module@v2.0.0
go mod tidy

# Pin to a specific pseudo-version
go get module/path@v0.0.0-20240101000000-abcdefabcdef

# Remove the replace directive after fix is upstream
go mod edit -dropreplace old/module
```
