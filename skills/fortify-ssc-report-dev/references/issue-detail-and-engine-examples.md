# Optional Issue-Detail, Engine And External-List Examples

Load this file only when the requested report uses issue detail, engine-specific data, long
descriptions or external-category mappings. These examples came from one mixed-engine workbook
customization and related seed reports; they are not requirements for trend, portfolio, KPI or other
reports. Object and component names describe an observed SSC 26.2 bundle, not a stable schema/API.

## Step 1: Diagnose Issue Coverage Where Applicable

Useful observed objects include `baseissueview`, `defaultissueview`, `issuecache`, `filterset`,
`projectversion_rule`, `ruledescription`, `rule_t`, `ruleview`, `scan`, `pluginmetadata`, and catpack tables.

For an issue-detail report intended to cover selected findings:

- Drive category/detail inclusion from those issues, not from optional rule descriptions.
- Treat rule text, external mappings, source snippets and plugin metadata as enrichment.
- Check engine/rule-key collisions and keep joins compatible with the agreed scope.
- Rule text lookup through `projectversion_rule` can account for overridden descriptions; a direct
  `issue.ruleGuid = ruledescription.guid` join is not always equivalent.
- Select a deterministic locale and agreed fallback; multiple language/description rows must not duplicate findings.

Load [portable SQL and SSC data](references/portable-sql-and-ssc-data.md) for issue scope, selected
filter-set visibility, and criticality rules. This file owns enrichment and example queries, not a
separate selection policy.

An observed Developer Workbook drove Results Outline through `ruledescription` and `ruleview`,
while its overview counted issues directly. FAA and Debricked findings lacked matching project-rule
metadata, so some categories disappeared. Issue-driven inclusion with optional rule text repaired
that particular report; it is not a reason to replace snapshot/measurement queries in a trend report.
Do not insert fake rules into SSC to satisfy a report join.

## Step 2: Example Portable Issue Measures

After confirming the columns and selected-filter semantics in the target schema, this illustrates
portable aggregation for a category/engine issue table:

```sql
SELECT i.engineType AS ENGINE_CODE,
       COUNT(*) AS ISSUE_COUNT,
       SUM(CASE WHEN i.friority='Critical' THEN 1 ELSE 0 END) AS CRITICAL_COUNT,
       SUM(CASE WHEN i.friority='Critical' AND i.audited='Y' THEN 1 ELSE 0 END) AS CRITICAL_AUDITED,
       SUM(CASE WHEN i.audited='Y' THEN 1 ELSE 0 END) AS TOTAL_AUDITED
FROM baseissueview i
INNER JOIN issuecache ic ON ic.issue_id=i.id AND ic.projectVersion_id=i.projectVersion_id
INNER JOIN filterset fs ON fs.id=ic.filterSet_id AND fs.projectVersion_id=i.projectVersion_id
WHERE i.projectVersion_id=? AND fs.guid=? AND i.mappedCategory=?
  AND ic.hidden='N' AND i.suppressed='N' AND i.scanStatus<>'REMOVED'
GROUP BY i.engineType
ORDER BY i.engineType
```

Apply the SQL reference's cache-cardinality, selected-visibility and fixed-criticality rules.
Document suppressed, removed, audited and correlated treatment for this report. An audited/total
table may display `a / t` for nonzero counts and a bare `0`, matching a seed convention.

## Step 3: Group Long Descriptions Only When Requested

Agree on equality channels: abstract, explanation, remediation and any tips/references actually
rendered. Equal category alone does not imply equal description. Define line-ending normalization,
null/empty equivalence or markup normalization explicitly; do not remove meaningful differences.

Grouping by rule-description ID is cheaper but does not combine different IDs with equal text.
State that limitation if accepted. Selecting `MAX(ruleGuid)` per category can put findings under
arbitrary or inappropriate descriptions.

For exact long-text grouping, use BIRT/scripted grouping with cached lookups or a verified preprocessing
layer rather than vendor-specific LOB grouping. Distinguish key domains, null versus literal "null",
and tuple fields unambiguously. If hashing is used, verify full content on collisions. Assess memory
and query count; repeated multi-kilobyte descriptions per finding can make a workbook much larger.
None of this processing is needed for a report that displays only summary measures.

## Step 4: Engine-Specific Presentation Examples

Keep raw identifiers for joins and other logic. By default, display the deprecated/technical `SCA`
engine code as `Fortify SAST` to avoid confusion with Software Composition Analysis, unless the user
requests otherwise. Confirm observed `SECURITYSCOPE`/`WEBINSPECTAGENT` aliases for the target version
before merging them.

If a requested table breaks down selected issues by engine, derive present engines from those issues.
If a report needs a full engine inventory, verify standard engines and available/enabled plugin metadata
separately from scan presence. One observed inventory used `lastAction` and `lastUsedOfKind`; these
fields are not a universal enabled-plugin API. Without reliable inventory, an explicit "Results Present:"
list is clearer than implying all available engines are listed. Unrelated reports need no inventory.

`issueAbstract` can contain HTML, plain text or Markdown; `issueRecommendation` and `snippetId` may
be empty for parser findings. Inspect actual channels rather than assuming SAST taint/source/sink
data exists for every engine. Use plain output for labels and a vetted allowlist sanitizer for rich
content; regex HTML detection is not sanitization.

A workbook engine table used Engine Type spanning two header rows, the category label spanning
Critical/High/Medium/Low columns, and a Total column. It reused seed blue/severity styles and
alternating rows. This is one layout option, not a required design for every report.

## Step 5: External-Category Mapping Examples

Observed external-list reports mapped `mappedCategory` through `catpacklookup` to
`catpackexternalcategory`/`catpackexternallist`. That differs from rule metadata and description
grouping. When external-list coverage is requested, surface unmapped data rather than silently
interpreting it as compliance. Do not map OWASP categories by kingdom alone or substring guesses.
Confirm supplemental metadata support for the installed SSC version; do not edit vendor-managed
ExternalMetadata files or catpack rows as a report-development workaround.

## Step 6: Choose Relevant Acceptance Checks

For the corresponding feature, examples include:

- Project identity survives replacement of a workbook engine component.
- Plugin-only categories appear when mixed-engine issue coverage is intended.
- Critical/High/Medium/Low, audited and Total counts match independently selected issues.
- The four criticality counts sum to Total; flag missing/unrecognized values as a validation problem,
  not an additional report criticality. Renamed/custom folders do not change that classification.
- Default/selected filter-set GUID and visibility match across summaries, category groups and details;
  include a case where global `hidden` and selected cache visibility differ.
- Distinct description content is not collapsed by category/rule identity.
- Results Outline has one body heading plus its TOC entry, with an empty-result fallback.
- Category/summary/table adjacency, header spans/colors and natural long-description pagination meet agreed layout goals.

Select these checks only if the report has those features. Use temporal, portfolio, metric or other
purpose-specific assertions for other report families.
