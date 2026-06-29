# JVM Ecosystem Reference — Gradle

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### Maven Central Registry

```text
# Confirm a specific version exists (non-zero numFound = published)
https://search.maven.org/solrsearch/select?q=g:<group-id>+AND+a:<artifact-id>+AND+v:<version>&rows=1&wt=json

# Full version list as XML
https://repo1.maven.org/maven2/<group-path>/<artifact-id>/maven-metadata.xml

# Human-readable package page
https://central.sonatype.com/artifact/<group-id>/<artifact-id>
```

### GitHub Releases & Changelog

```text
https://github.com/<org>/<repo>/releases
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
```

### Official Documentation & Migration Guides

```text
# Gradle plugin portal
https://plugins.gradle.org/plugin/<plugin-id>
```

### Security Advisories

```text
# GitHub Advisory Database
https://github.com/advisories?query=ecosystem%3Amaven+<artifact-id>

# OSV database
https://osv.dev/list?ecosystem=Maven&q=<artifact-id>
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

## Finding Project Files

```bash
find . -name "build.gradle" -o -name "build.gradle.kts"
```

## Dependency Analysis

```bash
# Show dependency tree
./gradlew dependencies

# Show updates
./gradlew dependencyUpdates    # requires the com.github.ben-manes.versions plugin

# Security check
./gradlew dependencyCheckAnalyze    # requires the org.owasp.dependencycheck plugin
```

## Lockfile

- **Gradle**: `gradle.lockfile` (if enabled)

## Verify Target Version Exists

Check Maven Central (Gradle resolves JVM artifacts from Maven Central by default):

```text
https://search.maven.org/solrsearch/select?q=g:<group-id>+AND+a:<artifact-id>+AND+v:<version>&rows=1&wt=json
```

Fallback — full version list as XML:

```text
https://repo1.maven.org/maven2/<group-path>/<artifact-id>/maven-metadata.xml
```

Post-change — verify what Gradle resolved in the project:

```bash
./gradlew :module:dependencyInsight --dependency <artifact-id> --configuration runtimeClasspath
```

If the version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Dependency Tree

Inspect every direct and transitive dependency that was actually resolved — not just what you declared. Useful when a third-party library pulls in a conflicting version that the BOM does not cover, or to confirm a forced version took effect.

```bash
./gradlew :module:dependencies --configuration runtimeClasspath
./gradlew :module:dependencies --configuration compileClasspath
```

## Build

Sample commands for compiling the project after applying changes (Step 6).

```bash
./gradlew build                     # compile + test all modules
./gradlew compileJava               # compile only (skip tests)
./gradlew compileKotlin             # Kotlin sources
./gradlew :module:build             # single module
./gradlew :module:compileJava       # single module compile only
```

## Run Tests

Sample commands for establishing a baseline and validating changes after an upgrade.

```bash
./gradlew test                      # all modules
./gradlew :module:test              # single module
```

## Find Affected Code

Sample search commands for locating usages impacted by dependency changes.

**Find Renamed Class Usage**

```bash
grep -r "OldClassName" src/ --include="*.java" --include="*.kt"
```

**Find Changed Package Imports**

```bash
grep -r "import com.example.old" src/ --include="*.java" --include="*.kt"
```

**Find Annotation Usage**

```bash
grep -r "@OldAnnotation" src/ --include="*.java" --include="*.kt"
```

## Find Affected Tests

Identify every test file that imports or exercises the package being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
grep -r "import com.example.old\|OldClass" src/test/ --include="*.java" --include="*.kt"
```

Run only a specific test class or module:

```bash
./gradlew test --tests "com.example.AffectedTestClass"
./gradlew :module:test --tests "com.example.*"
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

> Use the Dependency Tree section above to identify the conflict first, then apply one of the patterns below.

**Gradle — force a transitive version**

```kotlin
configurations.all {
    resolutionStrategy.force("com.example:conflict-lib:2.0.0")
}
```

**Gradle — substitute a module**

```kotlin
configurations.all {
    resolutionStrategy.dependencySubstitution {
        substitute(module("com.example:old-lib")).using(module("com.example:new-lib:2.0.0"))
    }
}
```
