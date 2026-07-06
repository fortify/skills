<!--
  Prompt and output template for AI-assisted review of
  `fcli license ncd-report list-contributors` output.

  Goal: produce a minimal JSON update file that can be fed to
  `fcli license ncd-report update-contributor-status`.

  Keep the AI output limited to changed rows only. Do not ask it to rewrite
  the full contributor export.
-->

# AI review template — `lsc` to `ucs`

Use this template when you have exported contributing users for dedupe/ignore analysis with:

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -q "contributionStatus=='contributing'" -o json --to-file contributors-contributing.json
```

Then give the AI this prompt, replacing `<PASTE_JSON_HERE>` with the exported JSON file.

## Prompt template

```text
You are reviewing contributor identities from an fcli NCD report.

Task:
- Identify likely duplicate human contributors
- Identify likely bot or non-human contributors that should be ignored
- Keep likely human contributors as contributing unless there is strong evidence otherwise
- Return both apply-now updates and manual-review candidates in a single response

Rules:
- Use only the records provided in the JSON input
- Assume input is already scoped to `contributionStatus=='contributing'`; do not infer changes for users outside that scope
- Preserve every `authorId` exactly as-is
- If marking a contributor as duplicate, set `duplicateOf` to another existing `authorId` from the same input
- Put only high-confidence changes in `proposedUpdates`; do not include unchanged rows
- Put medium-confidence candidates in `manualReviewCandidates` with evidence and recommended action
- For every update row, include `authorId`, `overrideStatusConfidence`, and `overrideStatusNotes`
- Use `overrideStatus` only when needed; allowed values are `contributing`, `duplicate`, and `ignored`
- If you set `duplicateOf`, also set `overrideStatus` to `duplicate` unless the input already makes that unnecessary for your reasoning
- Be conservative: if evidence is weak, omit the row instead of guessing

Confidence tiers:
- High: confidence >= 0.90 (eligible for `proposedUpdates`)
- Medium: confidence >= 0.75 and < 0.90 (must go to `manualReviewCandidates`)
- Low: confidence < 0.75 (exclude)

Heuristics:
- Strong duplicate signals: same clean person name, same email local-part, obvious nickname or abbreviated name variants, same human name with different corporate email domains
- Strong ignore signals: bot markers like `[bot]`, service-account naming, noreply automation addresses, known CI users
- Weak signals: shared last name only, generic first name only, partial email overlap without name support

Return format:
- Output JSON only
- Return an object with exactly these keys:
  - `proposedUpdates` (array for high-confidence apply-now updates)
  - `manualReviewCandidates` (array for medium-confidence review)
  - `summary` (object with counts by category and confidence tier)
- For each `proposedUpdates` object include at least:
  - `authorId`
  - `overrideStatus`
  - `overrideStatusConfidence`
  - `overrideStatusNotes`
  - `duplicateOf` when marking duplicates
- For each `manualReviewCandidates` object include at least:
  - `authorId`
  - `suggestedOverrideStatus`
  - `suggestedDuplicateOf` (for duplicate suggestions)
  - `confidence`
  - `evidence`

Input JSON (`contributors-contributing.json`):
<PASTE_JSON_HERE>
```

## Expected output shape

```json
{
  "proposedUpdates": [
    {
      "authorId": "author-id-1",
      "duplicateOf": "author-id-2",
      "overrideStatus": "duplicate",
      "overrideStatusConfidence": "0.96",
      "overrideStatusNotes": "Same human name and matching email local-part; appears to be the same contributor across two email domains."
    },
    {
      "authorId": "author-id-3",
      "overrideStatus": "ignored",
      "overrideStatusConfidence": "0.99",
      "overrideStatusNotes": "Account name contains [bot] and email address is a CI noreply automation address."
    }
  ],
  "manualReviewCandidates": [
    {
      "authorId": "author-id-4",
      "suggestedOverrideStatus": "duplicate",
      "suggestedDuplicateOf": "author-id-5",
      "confidence": 0.83,
      "evidence": "Name-token overlap and similar email local-parts, but one conflicting signal requires manual review."
    }
  ],
  "summary": {
    "duplicateHigh": 1,
    "ignoredHigh": 1,
    "duplicateMedium": 1,
    "ignoredMedium": 0
  }
}
```

## Apply the reviewed updates

Save `proposedUpdates` from the AI output to a file such as `contributors-reviewed.json`, then run:

```bash
fcli license ncd-report update-contributor-status -r ncd-report.zip -c contributors-reviewed.json --min-confidence 0.90
```

If the results look off, regenerate the review with stricter instructions and compare with a fresh contributing-only `list-contributors` export.