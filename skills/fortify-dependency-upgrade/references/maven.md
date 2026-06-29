# JVM Ecosystem Reference — Maven

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
# Spring Boot example (substitute versions)
https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-<NEW>-Release-Notes
https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-<NEW>-Migration-Guide
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
find . -name "pom.xml"
```

## Dependency Analysis

```bash
# Show dependency tree
mvn dependency:tree

# Show updates
mvn versions:display-dependency-updates

# Analyze dependencies
mvn dependency:analyze

# Security check
mvn dependency-check:check
```

## Lockfile

- **Maven**: No lockfile (uses `pom.xml` only)

## Verify Target Version Exists

Check Maven Central (non-zero `numFound` confirms the version exists):

```text
https://search.maven.org/solrsearch/select?q=g:<group-id>+AND+a:<artifact-id>+AND+v:<version>&rows=1&wt=json
```

Fallback — full version list as XML:

```text
https://repo1.maven.org/maven2/<group-path>/<artifact-id>/maven-metadata.xml
```

Post-change — verify what was actually resolved in the project:

```bash
mvn dependency:tree -Dincludes=<groupId>:<artifactId>
```

If the version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Dependency Tree

Inspect every direct and transitive dependency that was actually resolved — not just what you declared. Useful when a third-party library pulls in a conflicting version that the BOM does not cover, or to confirm a forced version took effect.

```bash
mvn dependency:tree -Dincludes=com.example:target-lib
```

## Build

Sample commands for compiling the project after applying changes (Step 6).

```bash
mvn compile                         # compile main sources
mvn package -DskipTests             # compile + package, skip tests
mvn package                         # compile + package + test
mvn compile -pl affected-module     # single module compile
```

## Run Tests

Sample commands for establishing a baseline and validating changes after an upgrade.

```bash
mvn test
mvn test -pl affected-module        # single module
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
mvn test -Dtest=AffectedTestClass
mvn test -pl affected-module
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

> Use the Dependency Tree section above to identify the conflict first, then apply the pattern below.

**Maven — exclude a transitive and re-add correct version**

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>parent-lib</artifactId>
  <exclusions>
    <exclusion>
      <groupId>com.example</groupId>
      <artifactId>transitive-lib</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>com.example</groupId>
  <artifactId>transitive-lib</artifactId>
  <version>2.0.0</version>
</dependency>
```
