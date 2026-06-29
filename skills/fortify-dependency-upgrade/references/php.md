# PHP Ecosystem Reference — Composer

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### Packagist Registry

```text
# Confirm a specific version exists
https://repo.packagist.org/p2/<vendor>/<package>.json

# Human-readable package page with all versions
https://packagist.org/packages/<vendor>/<package>
```

### GitHub Releases & Changelog

```text
https://github.com/<org>/<repo>/releases
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
```

### Official Documentation & Migration Guides

```text
# Symfony example (substitute versions)
https://symfony.com/doc/<NEW>/setup/upgrade_major.html

# Laravel example
https://laravel.com/docs/<NEW>/upgrade
```

### Security Advisories

```text
# PHP Security Advisories Database
https://github.com/FriendsOfPHP/security-advisories

# GitHub Advisory Database
https://github.com/advisories?query=ecosystem%3Acomposer+<package-name>

# OSV database
https://osv.dev/list?ecosystem=Packagist&q=<vendor>/<package>
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

## Finding Project Files
```bash
# Find all composer.json files
find . -name "composer.json"
```

## Dependency Analysis
```bash
# Show dependency tree
composer show --tree

# List installed packages
composer show

# Show outdated packages
composer outdated

# Security audit
composer audit
```

## Lockfile
- **Composer**: `composer.lock`

## Commands
```bash
# Install dependencies
composer install

# Update dependencies
composer update

# Validate composer.json
composer validate
```

## Verify Target Version Exists

Check the Packagist registry (a 200 response confirms the version exists):
```text
https://repo.packagist.org/p2/<vendor>/<package>.json
```

Fallback — full version list:
```bash
composer show <vendor/package> --all
```

Post-change — verify what was actually resolved in the project:
```bash
composer show <vendor/package>
```

If the version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Dependency Tree

```bash
composer depends vendor/package     # who depends on it?
composer why vendor/package
```

## Run Tests

```bash
./vendor/bin/phpunit
./vendor/bin/pest                   # Pest testing framework
```

## Find Affected Code

```bash
grep -r "use OldNamespace\\" src/ --include="*.php"
grep -r "OldClass\|old_function" src/ --include="*.php"
```

## Find Affected Tests

Identify every test file that imports or exercises the package being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
# Find test files that use the package namespace
grep -rl "use OldNamespace\\" tests/ --include="*.php"

# Run only the affected tests (PHPUnit)
./vendor/bin/phpunit --filter OldNamespaceTest tests/

# Pest
./vendor/bin/pest --filter=OldNamespace
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

Composer has no override field for transitive versions. To control a transitive dependency's version, promote it to a direct requirement — Composer then resolves the whole graph against that constraint:

```bash
# Find which packages constrain the transitive dependency
composer why vendor/transitive-package

# Require it directly at the needed version
composer require vendor/transitive-package:^2.0
```

> Note: the `conflict` field in `composer.json` only *blocks* versions from being installed — it cannot select or pin a version. Use it to express known incompatibilities, not to resolve them.
