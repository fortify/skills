<!--
  Worked example for the AI-assisted `list-contributors` ->
  `update-contributor-status` flow.

  This is intentionally small and concrete. It is meant to show users:
  - what `lsc -o json` output looks like
  - which rows the AI should change
  - what minimal `ucs` input should look like
-->

# Worked example — `lsc` to `ucs`

This example shows a small contributor export, the reasoning a reviewer or AI should apply, and the minimal JSON update payload to feed into `update-contributor-status`.

## Step 1: Export contributing users from the report

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -q "contributionStatus=='contributing'" -o json --to-file contributors-contributing.json
```

Example exported rows:

```json
[
  {
    "authorId": "f7b4e6f4f9d14db2a3f2d541eb9f1c01",
    "authorName": "John Smith",
    "authorEmail": "john.smith@acme.com",
    "contributionStatus": "contributing",
    "duplicateOf": "",
    "overrideStatus": "",
    "overrideStatusConfidence": "",
    "overrideStatusNotes": ""
  },
  {
    "authorId": "8c2f89b3277043e0a39b1ae5389035e7",
    "authorName": "J. Smith",
    "authorEmail": "jsmith@acme.com",
    "contributionStatus": "contributing",
    "duplicateOf": "",
    "overrideStatus": "",
    "overrideStatusConfidence": "",
    "overrideStatusNotes": ""
  },
  {
    "authorId": "42bd1e85a0e747cc935dd0fa0d093a0b",
    "authorName": "build-bot[bot]",
    "authorEmail": "build-bot@users.noreply.github.com",
    "contributionStatus": "contributing",
    "duplicateOf": "",
    "overrideStatus": "",
    "overrideStatusConfidence": "",
    "overrideStatusNotes": ""
  }
]
```

## Step 2: Decide which rows should change

In this example:
- `J. Smith` is likely the same person as `John Smith` because the surname matches and the email local-part is a common abbreviation of the same name.
- `build-bot[bot]` is likely non-human and should be ignored.
- `John Smith` remains unchanged, so it should not appear in the update payload.

## Step 3: Generate review output with apply-now and manual-review sections

Use one LLM response that separates high-confidence apply-now updates from medium-confidence manual-review candidates:

```json
{
  "proposedUpdates": [
    {
      "authorId": "8c2f89b3277043e0a39b1ae5389035e7",
      "duplicateOf": "f7b4e6f4f9d14db2a3f2d541eb9f1c01",
      "overrideStatus": "duplicate",
      "overrideStatusConfidence": "0.95",
      "overrideStatusNotes": "Likely same human contributor as John Smith; abbreviated first name and matching surname/email naming pattern."
    },
    {
      "authorId": "42bd1e85a0e747cc935dd0fa0d093a0b",
      "overrideStatus": "ignored",
      "overrideStatusConfidence": "0.99",
      "overrideStatusNotes": "Bot marker '[bot]' in the author name and noreply automation address indicate a non-human account."
    }
  ],
  "manualReviewCandidates": [],
  "summary": {
    "duplicateHigh": 1,
    "ignoredHigh": 1,
    "duplicateMedium": 0,
    "ignoredMedium": 0
  }
}
```

## Step 4: Apply the reviewed updates

Save `proposedUpdates` to a file such as `contributors-reviewed.json`, then run:

```bash
fcli license ncd-report update-contributor-status -r ncd-report.zip -c contributors-reviewed.json --min-confidence 0.90
```

## Step 5: Spot-check the result

Re-run `list-contributors` and confirm that:
- `J. Smith` now points to `John Smith` as `duplicateOf`
- `build-bot[bot]` is now ignored
- unchanged rows like `John Smith` were not modified unnecessarily

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json
```

## What this example is teaching

- Preserve `authorId` values exactly.
- Point `duplicateOf` to another existing `authorId` from the same export.
- Keep apply-now updates in `proposedUpdates` and medium-confidence items in `manualReviewCandidates`.
- Include confidence and notes for every AI-suggested change.
- Keep the review conservative; when evidence is weak, leave the row unchanged.