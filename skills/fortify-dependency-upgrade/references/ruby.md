# Ruby Ecosystem Reference — Bundler

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### RubyGems Registry

```text
# Confirm a specific version exists (200 = published)
https://rubygems.org/api/v2/rubygems/<gem-name>/versions/<version>.json

# Full version list
https://rubygems.org/api/v1/versions/<gem-name>.json

# Human-readable gem page
https://rubygems.org/gems/<gem-name>/versions
```

### GitHub Releases & Changelog

```text
https://github.com/<org>/<repo>/releases
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
```

### Security Advisories

```text
# RubyGems Advisory Database
https://www.rubysec.com/advisories/?gem=<gem-name>
https://github.com/rubysec/ruby-advisory-db

# GitHub Advisory Database
https://github.com/advisories?query=ecosystem%3Arubygems+<gem-name>

# OSV database
https://osv.dev/list?ecosystem=RubyGems&q=<gem-name>
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

## Finding Project Files
```bash
# Find all Gemfiles
find . -name "Gemfile"
```

## Dependency Analysis
```bash
# Show dependency tree
bundle list

# Show outdated gems
bundle outdated

# Show specific gem info
bundle info gem-name

# Security audit
bundle audit
```

## Lockfile
- **Bundler**: `Gemfile.lock`

## Commands
```bash
# Install dependencies
bundle install

# Update specific gem
bundle update gem-name
```

## Verify Target Version Exists

Check the RubyGems registry (a 200 response confirms the version exists):
```text
https://rubygems.org/api/v2/rubygems/<gem-name>/versions/<version>.json
```

Fallback — full version list:
```bash
gem list <gem-name> --remote --all
```

Post-change — verify what was actually resolved in the project:
```bash
bundle list | grep <gem-name>
```

If the version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Dependency Tree

```bash
bundle list
gem dependency gem-name --reverse-dependencies
```

## Run Tests

```bash
bundle exec rspec
bundle exec rake test               # Minitest via Rake
```

## Find Affected Code

```bash
grep -r "require 'old_gem'\|require \"old_gem\"" . --include="*.rb"
grep -r "OldClass\|old_method" . --include="*.rb"
```

## Find Affected Tests

Identify every test file that imports or exercises the gem being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
# Find spec/test files that require the gem
grep -rl "require 'old_gem'\|require \"old_gem\"" spec/ test/ --include="*.rb"

# Run only affected specs
bundle exec rspec $(grep -rl "require 'old_gem'" spec/ --include="*.rb")
bundle exec rake test TEST=$(grep -rl "require 'old_gem'" test/ --include="*.rb" | tr '\n' ' ')
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## Transitive / Peer Conflict Resolution

Bundler has no per-gem force or override flag. To control a transitive dependency's version, promote it to a direct dependency in the Gemfile — Bundler then resolves the whole graph against that constraint:

```ruby
# Gemfile — pin a transitive dependency by making it direct
gem 'transitive-gem', '>= 2.0'
```

If resolution fails, find which gems constrain it and update them together:

```bash
gem dependency transitive-gem --reverse-dependencies
bundle update transitive-gem parent-gem
```
