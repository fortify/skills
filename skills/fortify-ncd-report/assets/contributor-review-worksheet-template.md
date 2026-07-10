# Contributor review worksheet

Use this worksheet after exporting contributing contributors with `fcli license ncd-report list-contributors`.
The assistant should generate this file from `contributors.json` and keep the chat output to a short
summary plus the file path.

If the runtime can render markdown directly from fcli expression output, prefer that. Otherwise,
use a small local script to convert the JSON export into this worksheet.

## Summary

- Report:
- Contributing contributors:
- Dormant contributors:
- Non-dormant contributors:
- Unknown-dormancy contributors:
- Suggested ignores:
- Suggested duplicates:
- Manual-review candidates:

## Suggested ignores

| Select | authorId | authorName | authorEmail | sourceReport | repositoryUrl(s) | confidence | notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| [ ] |  |  |  |  |  |  |  |

## Suggested duplicates

| Select | authorId | authorName | authorEmail | duplicateOf | sourceReport | repositoryUrl(s) | confidence | notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [ ] |  |  |  |  |  |  |  |  |

## Manual review

| Select | authorId | authorName | authorEmail | proposed status | duplicateOf | sourceReport | repositoryUrl(s) | confidence | notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [ ] |  |  |  |  |  |  |  |  |  |

## Review notes

- Tick `Select` for rows to include in `contributors-reviewed.json`.
- Fill `proposed status` with `ignored`, `duplicate`, or `contributing` as appropriate.
- For duplicates, fill `duplicateOf` with another existing `authorId` from the same export.
- Keep `authorId` unchanged.
- Add any rationale that should be preserved in `overrideStatusNotes`.
- If you want a machine-readable companion file, convert the checked rows into
  `contributors-reviewed.json` with `authorId`, `overrideStatus`, `overrideStatusConfidence`,
  `overrideStatusNotes`, and `duplicateOf` where applicable.
