# Python Ecosystem Reference — pip / poetry / uv

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### PyPI Registry

```text
# Confirm a specific version exists (200 = published)
https://pypi.org/pypi/<package-name>/<version>/json

# List all published versions
https://pypi.org/pypi/<package-name>/json   (key: releases)

# Human-readable release history
https://pypi.org/project/<package-name>/#history
```

### GitHub Releases & Changelog

```text
# GitHub releases page (substitute org/repo)
https://github.com/<org>/<repo>/releases

# GitHub API — releases list (JSON)
https://api.github.com/repos/<org>/<repo>/releases

# Raw CHANGELOG in repo root (common filenames)
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.md
https://raw.githubusercontent.com/<org>/<repo>/main/CHANGELOG.rst
https://raw.githubusercontent.com/<org>/<repo>/main/HISTORY.md
```

### Official Documentation & Migration Guides

```text
# ReadTheDocs (common pattern)
https://<package>.readthedocs.io/en/stable/changelog.html
https://<package>.readthedocs.io/en/stable/migration.html

# Django-specific (substitute versions)
https://docs.djangoproject.com/en/<NEW>/releases/<NEW>.html
https://docs.djangoproject.com/en/<NEW>/howto/upgrade-version.html
```

### Security Advisories

```text
# PyPI advisory database (OSV format)
https://osv.dev/list?ecosystem=PyPI&q=<package-name>

# GitHub Security Advisories
https://github.com/advisories?query=ecosystem%3Apip+<package-name>
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

## Finding Project Files

```bash
find . -name "requirements*.txt" -o -name "setup.py" -o -name "pyproject.toml" -o -name "Pipfile"
```

## Dependency Analysis
```bash
# List installed packages
pip list                  # or: poetry show / uv pip list

# Show package details
pip show package-name     # or: poetry show package-name

# List outdated
pip list --outdated       # or: poetry show --outdated

# Dependency tree
pipdeptree                # or: poetry show --tree

# Security audit
pip-audit                 # or: safety check
```

## Lockfile
- **pip**: `requirements.txt` (not a lockfile)
- **poetry**: `poetry.lock`
- **pipenv**: `Pipfile.lock`
- **uv**: `uv.lock`

## Commands
```bash
# Install
pip install -r requirements.txt    # or: poetry install / uv pip install
```

## Verify Target Version Exists

Check the PyPI registry (a 200 response confirms the version exists):
```text
https://pypi.org/pypi/<package-name>/<version>/json
```

Fallback — full version list:
```bash
pip index versions <package-name>
```

Post-change — verify what was actually resolved in the project:
```bash
pip show <package-name>
```

If the version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Dependency Tree

```bash
pip show package-name               # shows reverse dependencies
pipdeptree -p package-name          # requires pipdeptree
pipdeptree --reverse -p package-name
```

## Run Tests

```bash
# Full suite — always run this for baseline and final verification
pytest
pytest --tb=short -q                # brief output
pytest -v --tb=long                 # verbose, full tracebacks

# Save baseline output to file for diff comparison
pytest --tb=short -q 2>&1 | tee baseline-test-results.txt
```

## Find Affected Code

```bash
grep -r "from old_module import\|import old_module" src/ tests/
grep -r "old_function\|OldClass" src/ tests/
```

## Find Affected Tests

Identify every test file that imports or exercises the package being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
# Replace <package> with the import name (e.g. "django", "requests", "sqlalchemy")
grep -rl "import <package>\|from <package>" tests/

# Run only the affected tests to get a focused baseline
pytest $(grep -rl "import <package>\|from <package>" tests/) --tb=short -q

# Also check conftest files that may inject fixtures from the package
grep -rl "import <package>\|from <package>" tests/ --include="conftest.py"
```

Report the resulting file list as the **affected test scope** — these are the tests most likely to break due to the upgrade.

## Transitive / Peer Conflict Resolution

```bash
# pip — pin the transitive in constraints.txt
echo "transitive-package==2.0.0" >> constraints.txt
pip install -r requirements.txt -c constraints.txt

# poetry — add with constraint
poetry add "transitive-package@2.0.0"

# uv — override
uv add "transitive-package==2.0.0"
```
