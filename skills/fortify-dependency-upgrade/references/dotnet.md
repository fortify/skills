# .NET Ecosystem Reference — NuGet / dotnet CLI

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### NuGet Registry

```text
# Confirm a specific version exists (200 = published; package id must be lowercase)
https://api.nuget.org/v3/registration5-semver1/<package-id-lowercase>/<version>.json

# Package page with all versions
https://www.nuget.org/packages/<package-id>

# Full version list (JSON; package id must be lowercase)
https://api.nuget.org/v3-flatcontainer/<package-id-lowercase>/index.json
```

### GitHub Releases & Changelog

```text
https://github.com/<org>/<repo>/releases
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
```

### Official Documentation & Migration Guides

```text
# Microsoft Docs — .NET upgrade guides
https://learn.microsoft.com/en-us/dotnet/core/compatibility/
https://learn.microsoft.com/en-us/dotnet/core/whats-new/
```

### Security Advisories

```text
# GitHub Advisory Database
https://github.com/advisories?query=ecosystem%3Anuget+<package-name>

# NuGet Security Advisories
https://github.com/NuGet/Announcements/issues
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

## Finding Project Files with Dependencies

```powershell
# Find all C# project files
Get-ChildItem -Recurse -Filter *.csproj

# Find all .NET project files (C#, F#, VB.NET)
Get-ChildItem -Recurse -Include *.csproj,*.fsproj,*.vbproj

# Unix/Linux/Mac
find . -name "*.csproj" -o -name "*.fsproj" -o -name "*.vbproj"
```

## Dependency Analysis Commands

```powershell
# List all packages with versions
dotnet list package

# Include transitive dependencies (CRITICAL for graph analysis)
dotnet list package --include-transitive

# Check for outdated packages
dotnet list package --outdated

# Check for vulnerable packages
dotnet list package --vulnerable

# List packages for specific project
dotnet list path/to/Project.csproj package --include-transitive

# Restore packages (regenerate package resolution)
dotnet restore

# Clean build artifacts
dotnet clean
```

## Lockfile Location

.NET uses **implicit package resolution** - no lockfile by default.

To enable explicit lockfile (recommended for dependency analysis):
```powershell
# Generate packages.lock.json
dotnet restore --use-lock-file

# Lockfile location: {ProjectDirectory}/packages.lock.json
```

## Verify Target Version Exists

Check the NuGet registry (a 200 response confirms the version exists; the package id must be lowercase):
```text
https://api.nuget.org/v3/registration5-semver1/<package-id-lowercase>/<version>.json
```

Fallback — full version list:
```bash
dotnet package search <package-name> --exact-match
```

Post-change — verify what was actually resolved in the project:
```bash
dotnet list package --include-transitive
```

If the version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Dependency Tree

```bash
dotnet list package --include-transitive
```

## Run Tests

Always run tests at two explicit points in the migration workflow.

### 1. Pre-migration baseline (before any changes)

Run the full test suite and record the pass/fail/skip counts as your baseline.
If tests already fail before the upgrade, note them — do not count them as
upgrade-caused failures later.

```bash
dotnet test
```

### 2. Post-migration verification (after all changes are applied)

Re-run and compare against the baseline. Any new failures must be investigated
and resolved before the migration is considered complete.

```bash
dotnet test
dotnet test --no-build              # skip rebuild if already built
```

### Test project tiers

| Tier | Typical folder | Requires infrastructure? |
|------|---------------|--------------------------|
| Unit | `tests/Unit.Test` | No — safe to run anywhere |
| Integration | `tests/Integration.Test` | Yes — usually needs a DB container |
| End-to-End | `tests/EndToEnd.Test` | Yes — needs full stack running |

If infrastructure is unavailable, run only unit tests and explicitly state in the
migration report what could not be verified and why.

## Find Affected Code

```bash
grep -r "using OldNamespace" . --include="*.cs"
grep -r "OldClass\|OldMethod" . --include="*.cs"
```

## Find Affected Tests

Identify every test file that imports or exercises the package being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
# Search test files for usages of the package namespace or types
grep -r "using OldNamespace\|OldClass\|OldMethod" . \
  --include="*Tests.cs" --include="*Test.cs" --include="*Spec.cs"

# Run only the affected test project
dotnet test tests/MyApp.Tests/
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

**<project>.csproj — override transitive version**
```xml
<PackageReference Include="TransitivePackage" Version="2.0.0" />
```

**Directory.Packages.props — central override**
```xml
<PackageVersion Include="TransitivePackage" Version="2.0.0" OverriddenVersion="true" />
```

## Compatibility Matrix Example

| Package | .NET 6 | .NET 8 | .NET 9 |
|---------|--------|--------|--------|
| ASP.NET Core | 6.x | 8.x | 9.x |
| Entity Framework Core | 6.x | 8.x | 9.x |
| System.Text.Json | 6.x | 8.x | 9.x |

