# Scala Ecosystem Reference — sbt / Mill

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### Maven Central Registry (primary Scala artifact repository)

```text
# Confirm a specific version exists (non-zero numFound = published)
https://search.maven.org/solrsearch/select?q=g:<group-id>+AND+a:<artifact-id>_<scala-version>+AND+v:<version>&rows=1&wt=json

# Full version list as XML
https://repo1.maven.org/maven2/<group-path>/<artifact-id>_<scala-version>/maven-metadata.xml

# Human-readable package page
https://central.sonatype.com/artifact/<group-id>/<artifact-id>_<scala-version>
```

### GitHub Releases & Changelog

```text
https://github.com/<org>/<repo>/releases
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
```

### Official Documentation & Migration Guides

```text
# Akka / Pekko migration (example pattern)
https://pekko.apache.org/docs/pekko/current/project/migration-guides.html

# Scala release notes
https://www.scala-lang.org/news/
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
# Find sbt build files
find . -name "build.sbt" -o -name "Build.scala"
find . -path "*/project/plugins.sbt"

# Find Mill build files
find . -name "build.sc" -o -name "build.mill"
```

## Dependency Analysis

### sbt

```bash
# Full dependency tree (sbt 1.4+; requires sbt-dependency-graph plugin on older versions)
sbt dependencyTree

# Per-subproject tree
sbt projects                          # list all subprojects
sbt "<subproject>/dependencyTree"

# What depends on a specific artifact
sbt "whatDependsOn <org> <name> <version>"

# List outdated dependencies
sbt dependencyUpdates

# Show all resolved dependencies
sbt "show libraryDependencies"
sbt "show dependencyClasspath"
```

On sbt < 1.4, add the plugin first:

```scala
// project/plugins.sbt
addDependencyTreePlugin
```

### Mill

```bash
# Show dependency tree for a module
mill <module>.ivyDepsTree

# With transitive deps
mill <module>.ivyDepsTree --withCompile
```

### Coursier (standalone resolver)

```bash
# Resolve and print full dep tree
cs resolve --tree <org>:<name>:<version>
```

## Lockfile

sbt does not generate a lockfile by default. Options:

* **sbt-lock** plugin: produces `build.lock`
* **Coursier**: `~/.cache/coursier/v1/...` caches resolved artifacts; not a project-level lockfile.
* Commit `project/build.properties` (sbt version) and `project/plugins.sbt` to pin the toolchain.

## Upgrade a Dependency

1. Edit `build.sbt` to bump the version:

```scala
// Before
libraryDependencies += "com.typesafe.akka" %% "akka-http" % "10.2.9"

// After
libraryDependencies += "com.typesafe.akka" %% "akka-http" % "10.5.3"
```

2. Reload sbt and update:

```bash
sbt reload
sbt update
```

> The Coursier cache is keyed by version, so bumping a release version never needs a cache clear — `sbt update` picks it up. Only changing `-SNAPSHOT` artifacts may require removing the artifact's directory under `~/.cache/coursier/v1/` before re-resolving.

### Scala version suffix

Scala's binary-compatibility model publishes one artifact per Scala major version:
`<artifact>_2.12`, `<artifact>_2.13`, `<artifact>_3`. Using `%%` in sbt automatically
appends the correct suffix for your `scalaVersion`. When checking advisories or changelogs,
verify the advisory applies to the suffix you use.

## Verify Target Version Exists

Check Maven Central (the primary Scala artifact repository):

```bash
# Coursier
cs resolve <org>:<name>_<scala-version>:<target-version>

# Or browse Maven Central search
https://search.maven.org/artifact/<groupId>/<artifactId>
```

List all available versions:

```bash
cs resolve --tree <org>:<name>_<scala-version>:latest.release 2>&1 | head -20
# Or via Coursier API
cs complete-dep <org>:<name>_<scala-version>:
```

If the requested version is not listed, stop and tell the user it has not been published yet,
then provide the latest available stable version.

## Dependency Tree

```bash
# sbt
sbt dependencyTree

# Reverse — what depends on this artifact
sbt "whatDependsOn <org> <name> <version>"

# Coursier standalone
cs resolve --tree <org>:<name>:<version>

# Mill
mill <module>.ivyDepsTree
```

## Compile

```bash
# sbt
sbt compile
sbt "~compile"              # continuous compilation

# All subprojects
sbt "+compile"              # cross-compile across Scala versions

# Mill
mill __.compile
```

## Run Tests

```bash
# sbt — run all tests
sbt test

# Run tests in a specific subproject
sbt "<subproject>/test"

# Run a single test class
sbt "testOnly com.example.MySpec"

# Run tests matching a pattern
sbt "testOnly *MySpec*"

# Continuous test on file change
sbt "~test"

# Mill
mill __.test
mill <module>.test
```

## Find Affected Code

```bash
grep -r "oldPackage\|OldClass\|oldMethod" . --include="*.scala" --include="*.java"
```

## Find Affected Tests

Identify every test file that imports or exercises the library being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
# Search test source directories for usages
grep -r "oldPackage\|OldClass\|oldMethod" src/test/ --include="*.scala" --include="*.java"

# Or search by file name pattern across the whole project
grep -rl "import com.example.old\|OldClass" . \
  --include="*Spec.scala" --include="*Suite.scala" --include="*Test.scala" --include="*Tests.scala"
```

Run only the affected test class:

```bash
sbt "testOnly *OldClassSpec"
sbt "testOnly *OldClass*"
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

Show which versions are in conflict:

```bash
sbt evicted                      # lists evicted (overridden) artifacts
```

Then resolve in `build.sbt`:

```scala
// Force a specific version for a transitive dependency
dependencyOverrides += "org.apache.logging.log4j" % "log4j-core" % "2.23.1"

// Exclude a transitive dep entirely and bring in a safe version directly
libraryDependencies += "com.example" %% "library" % "1.0" exclude("log4j", "log4j")
libraryDependencies += "org.apache.logging.log4j" % "log4j-core" % "2.23.1"
```
