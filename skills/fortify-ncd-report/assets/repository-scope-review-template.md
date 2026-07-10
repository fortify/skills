# Repository scope review worksheet

Use this worksheet after exporting repositories with `fcli license ncd-report list-repositories`.
The assistant should generate this file from `repositories-included.json` and keep the chat output
to a short summary plus the file path.

If the runtime can render markdown directly from fcli expression output, prefer that. Otherwise,
use a small local script to convert the JSON export into this worksheet.

## Summary

- Report:
- Included repositories:
- Dormant repositories:
- Non-dormant repositories:
- Unknown-dormancy repositories:
- High-confidence exclusions:
- Medium-confidence exclusions:
- Low-confidence exclusions:

## High-confidence exclusions

| Select | repositoryUrl | fork | dormant | status | sourceReport | evidence | suggested config change | notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [ ] |  |  |  |  |  |  |  |  |

## Medium-confidence exclusions

| Select | repositoryUrl | fork | dormant | status | sourceReport | evidence | suggested config change | notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [ ] |  |  |  |  |  |  |  |  |

## Low-confidence exclusions

| Select | repositoryUrl | fork | dormant | status | sourceReport | evidence | suggested config change | notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [ ] |  |  |  |  |  |  |  |  |

## Review notes

- Tick `Select` for repositories to exclude.
- Fill `evidence` with the short reason that should be carried into the config discussion.
- Fill `suggested config change` with `includeForks: false`, a `repositoryIncludeExpression` change,
  or a source-level selector narrowing.
- Leave unselected rows in scope.
- If you want a machine-readable companion file, convert the checked rows into
  `repositories-reviewed.json` with `repositoryUrl`, `selected`, `confidence`, and `notes` fields.
