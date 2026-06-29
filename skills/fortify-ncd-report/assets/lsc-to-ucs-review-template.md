<!--
  Prompt and output template for AI-assisted review of
  `fcli license ncd-report list-contributors` output.

  Goal: produce a minimal JSON update file that can be fed to
  `fcli license ncd-report update-contributor-status`.

  Keep the AI output limited to changed rows only. Do not ask it to rewrite
  the full contributor export.
-->

# AI review template — `lsc` to `ucs`

Use this template when you have exported contributors with:

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors.json
```

Then give the AI this prompt, replacing `<PASTE_JSON_HERE>` with the exported JSON file.

## Prompt template

```text
You are reviewing contributor identities from an fcli NCD report.

Task:
- Identify likely duplicate human contributors
- Identify likely bot or non-human contributors that should be ignored
- Keep likely human contributors as contributing unless there is strong evidence otherwise

Rules:
- Use only the records provided in the JSON input
- Preserve every `authorId` exactly as-is
- If marking a contributor as duplicate, set `duplicateOf` to another existing `authorId` from the same input
- Output only rows that should change; do not return unchanged rows
- For every output row, include `authorId`, `overrideStatusConfidence`, and `overrideStatusNotes`
- Use `overrideStatus` only when needed; allowed values are `contributing`, `duplicate`, and `ignored`
- If you set `duplicateOf`, also set `overrideStatus` to `duplicate` unless the input already makes that unnecessary for your reasoning
- Be conservative: if evidence is weak, omit the row instead of guessing

Heuristics:
- Strong duplicate signals: same clean person name, same email local-part, obvious nickname or abbreviated name variants, same human name with different corporate email domains
- Strong ignore signals: bot markers like `[bot]`, service-account naming, noreply automation addresses, known CI users
- Weak signals: shared last name only, generic first name only, partial email overlap without name support

Return format:
- Output JSON only
- Return an array of objects
- Each object must include at least:
  - `authorId`
  - `overrideStatus`
  - `overrideStatusConfidence`
  - `overrideStatusNotes`
- Include `duplicateOf` when marking duplicates

Input JSON:
<PASTE_JSON_HERE>
```

## Expected output shape

```json
[
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
]
```

## Apply the reviewed updates

Save the AI output to a file such as `contributors-reviewed.json`, then run:

```bash
fcli license ncd-report update-contributor-status -r ncd-report.zip -c contributors-reviewed.json --min-confidence 0.90
```

If the results look off, regenerate the review with stricter instructions and compare with a fresh `list-contributors` export.