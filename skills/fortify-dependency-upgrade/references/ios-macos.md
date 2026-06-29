# iOS / macOS Ecosystem Reference — CocoaPods / Swift Package Manager / Carthage

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### Registry & Release Lookup

```text
# CocoaPods — confirm version exists and view all releases
https://cocoapods.org/pods/<PodName>

# SPM — confirm Git tag exists
git ls-remote --tags https://github.com/<org>/<repo>.git

# Swift Package Index (human-readable)
https://swiftpackageindex.com/<org>/<repo>
```

### GitHub Releases & Changelog

```text
https://github.com/<org>/<repo>/releases
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
```

### Official Documentation & Migration Guides

```text
# Apple developer release notes
https://developer.apple.com/documentation/xcode-release-notes
https://developer.apple.com/news/releases/
```

### Security Advisories

```text
# GitHub Advisory Database
https://github.com/advisories?query=ecosystem%3Aswift+<package-name>

# OSV database
https://osv.dev/list?ecosystem=SwiftURL&q=<package-name>
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

A single Xcode project often mixes two or three dependency managers. Check for all of them
before assuming the dependency set is complete: `Podfile` (CocoaPods), `Package.swift` (SPM),
`Cartfile` (Carthage).

## Finding Project Files

```bash
# Find CocoaPods manifests
find . -name "Podfile" -o -name "Podfile.lock"

# Find SPM manifests
find . -name "Package.swift" -o -name "Package.resolved"

# Find Carthage manifests
find . -name "Cartfile" -o -name "Cartfile.resolved"
```

## Dependency Analysis

### CocoaPods

```bash
# Read resolved pods and versions from the lockfile (authoritative; YAML)
cat Podfile.lock

# List outdated pods
pod outdated
```

### Swift Package Manager (SPM)

```bash
# Show full dependency tree
swift package show-dependencies --format text

# JSON output (for scripting)
swift package show-dependencies --format json

# List all resolved versions
cat Package.resolved          # top-level JSON; Xcode-managed projects store it at:
# <proj>.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved

# Check for available updates
swift package update --dry-run 2>/dev/null || swift package resolve
```

### Carthage

```bash
# List all pinned packages and commits
cat Cartfile.resolved

# List outdated packages
carthage outdated
```

## Lockfile

* **CocoaPods**: `Podfile.lock`
* **SPM**: `Package.resolved` (or inside `.xcodeproj/…/swiftpm/Package.resolved`)
* **Carthage**: `Cartfile.resolved`

## Upgrade a Dependency

### CocoaPods

1. Edit `Podfile` to specify the new version:

```ruby
pod 'Alamofire', '~> 5.8'
```

2. Update the specific pod:

```bash
pod update Alamofire
```

3. Or update all pods:

```bash
pod update
```

### Swift Package Manager

1. Edit `Package.swift` — bump the version requirement:

```swift
.package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
```

2. Resolve and update `Package.resolved`:

```bash
swift package update Alamofire
# Or update all:
swift package update
```

3. For an Xcode-managed project, use Xcode → File → Packages → Update To Latest Package Versions,
   or resolve from the command line via `xcodebuild`:

```bash
xcodebuild -resolvePackageDependencies -workspace <workspace>.xcworkspace -scheme <scheme>
```

### Carthage

1. Edit `Cartfile` to bump the version constraint:

```
github "Alamofire/Alamofire" ~> 5.8
```

2. Update the specific dependency:

```bash
carthage update Alamofire --platform iOS
```

## Verify Target Version Exists

### CocoaPods

```bash
pod spec which <pod-name>
pod trunk info <pod-name>        # lists all published versions
```

### SPM

```bash
# List tags in the upstream repo
git ls-remote --tags https://github.com/<org>/<repo>.git
```

### Carthage

```bash
# Confirm the release exists on GitHub
git ls-remote --tags https://github.com/<org>/<repo>.git | grep <version>
```

If the requested version is not listed, stop and tell the user it has not been published yet,
then provide the latest available stable version.

## Dependency Tree

```bash
# CocoaPods — read Podfile.lock PODS: section; shows each pod and its sub-dependencies
cat Podfile.lock

# SPM
swift package show-dependencies --format text

# Carthage — flat list; no built-in tree command
cat Cartfile.resolved
```

## Build

```bash
# CocoaPods
pod install                     # generates/updates the workspace
xcodebuild -workspace <workspace>.xcworkspace -scheme <scheme> build

# SPM (library/command-line tool)
swift build

# Xcode project
xcodebuild -project <project>.xcodeproj -scheme <scheme> build

# Carthage — rebuild frameworks after upgrade
carthage build --platform iOS
```

## Run Tests

```bash
# SPM
swift test
swift test --filter <TestSuiteName>

# Xcode / CocoaPods workspace
xcodebuild test \
  -workspace <workspace>.xcworkspace \
  -scheme <scheme> \
  -destination 'platform=iOS Simulator,name=iPhone 15,OS=latest'

# Xcode project (no workspace)
xcodebuild test \
  -project <project>.xcodeproj \
  -scheme <scheme> \
  -destination 'platform=macOS'
```

## Find Affected Code

```bash
# Swift
grep -r "import OldModule\|OldType\|oldFunction" . --include="*.swift"

# Objective-C
grep -r "#import <OldLib/\|OldClass\|oldMethod" . --include="*.m" --include="*.h"
```

## Find Affected Tests

Identify every test file that imports or exercises the dependency being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
# Find Swift test files that use the module
grep -rl "import OldModule\|OldType\|oldFunction" . \
  --include="*Tests.swift" --include="*Test.swift" --include="*Spec.swift"

# Run only the affected test target (SPM)
swift test --filter OldModuleTests

# Run only the affected test target (Xcode)
xcodebuild test \
  -workspace <workspace>.xcworkspace \
  -scheme <scheme> \
  -only-testing <TestTarget>/OldModuleTests \
  -destination 'platform=iOS Simulator,name=iPhone 15,OS=latest'
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

```bash
# CocoaPods — read dependency conflicts from pod install output
pod install --verbose 2>&1 | grep -i "conflict\|warning"

# SPM — check Package.resolved for unexpected transitive pins
swift package resolve
cat Package.resolved | python3 -m json.tool | grep -A3 '"identity"'
```
