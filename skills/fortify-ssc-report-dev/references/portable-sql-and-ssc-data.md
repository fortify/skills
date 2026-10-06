# Portable SQL And SSC Data Semantics

## Step 1: Set The Portability Contract

Default targets are MySQL, Microsoft SQL Server and Oracle. An explicitly customer-specific report may
use a narrower target; record that exception. Check supported DB versions for the target SSC release.
Inspect actual schema/view definitions and seed queries before adding joins; names here describe an
observed SSC 26.2 schema, not a stable public schema contract.

Prefer portable `CASE`, `COALESCE`, joins and aggregates over scalar, non-LOB columns. Bind all values
through JDBC parameters. Dynamic identifiers need an allowlist; parameters cannot bind identifiers.

| Trap | Safer approach |
|------|----------------|
| MySQL `SUM(condition)` | `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` |
| `IF`, `IFNULL`, `NVL`, `ISNULL` | Standard `CASE`/`COALESCE`, with compatible operand types |
| `LIMIT`, `TOP`, `ROWNUM` | Avoid a query-level limit when possible; use a verified BIRT row-fetch limit |
| `GROUP_CONCAT`, `STRING_AGG`, `LISTAGG` | Return rows and format/group in BIRT |
| Vendor hashes such as `MD5`/`HASHBYTES` | Compare description keys outside SQL; a hash alone is not equality proof |
| `CONCAT`, `+`, `||` for output strings | Format labels/text in BIRT; argument/null semantics differ by DB |
| Constant `SELECT` without a table | Oracle normally requires a source; use a verified source or scripted rows, not an invented singleton table |
| Comparing/grouping CLOB, TEXT or legacy SQL Server text | Move long-text equality/grouping out of SQL or agree on tested vendor adapters |
| Mixed string/number `CASE` for `N/A` | Keep numeric measures and separate string display columns |
| Empty string versus NULL | Oracle treats empty strings as NULL; define null/empty behavior explicitly |

Date arithmetic, casts, identifier quoting, collation, case sensitivity and ORDER BY alias support
also need review. Do not infer SQL `TOP` usage from report titles or JavaScript `.substring()` calls.
Even standard constructs need compatible types, aliases, metadata and cardinality checks.

### Example: Portable Measures

After confirming these columns in the target schema, use this kind of parameterized query:

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

Ensure the cache/filter-set join produces one row per selected issue; otherwise counts multiply.
Use the same predicates for category drivers, details, summaries and independent count checks.
Do not use `DISTINCT` as a blanket repair for a broken join.

Execute on every target DB when available. If only one is available, perform portability review and
explicitly report the remaining engine tests as unverified; do not claim three-DB certification.

## Step 2: Separate Issue Inclusion From Optional Enrichment

Useful observed objects include `baseissueview`, `defaultissueview`, `issuecache`, `filterset`,
`projectversion_rule`, `ruledescription`, `rule_t`, `ruleview`, `scan`, `pluginmetadata`, and catpack tables.

- Drive category/detail inclusion from selected issues, not from rule descriptions.
- Treat rule text, external mappings, source snippets and plugin metadata as optional enrichments.
- Keep project-version scoping in joins and check engine/rule-key collisions.
- Rule text lookup through `projectversion_rule` can account for overridden descriptions; a direct
  `issue.ruleGuid = ruledescription.guid` join is not always equivalent.
- Select a deterministic locale from available rule text, with an agreed fallback. Multiple language
  or rule-description rows must not duplicate findings or arbitrarily change text.

A Developer Workbook observed in SSC 26.2 drove its Results Outline through `ruledescription` and
`ruleview`, while its overview counted issues directly. FAA and Debricked issues existed but had no
matching project-rule metadata. This caused missing categories in the outline, not missing issues in SSC.
The repair was issue-driven inclusion with optional rule text, not inserting fake rules in the DB.

## Step 3: Preserve Visibility And Count Meaning

The issue's global `hidden` flag and `issuecache.hidden` for a chosen filter set may disagree.
Always use the report's selected filter-set semantics consistently; otherwise headers can report
categories whose detail queries return no rows, or counts can disagree between sections.

Document treatment of suppressed, hidden, removed, audited and correlated issues. The historical
column `friority` represents Fortify priority; do not substitute raw `severity` or engine priority.
For audited/total display, follow the report's existing audited predicate and label the convention.
Avoid dividing or interpreting `0 / 0` as a percentage; a bare `0` may match the existing table style.

Test no findings, one finding, mixed engines, nonzero audited counts, and filter-set changes.
Use independent fixtures or test-only report copies if production data lacks a branch; never alter DB
data to manufacture a test case. Such test copies are not deployable deliverables.

## Step 4: Group Descriptions Without Losing Information

Agree on the content channels that define equality: abstract, explanation, remediation and any tips
or references that the report actually renders. Equal category alone does not imply equal description.
Define whether line-ending normalization, null/empty equivalence or markup normalization is intended.
Compare exact content tuples under that policy; do not normalize away meaningful text silently.

Grouping by rule-description ID is a cheaper identity-based optimization, but does not combine two
different IDs with equal text. State that limitation if the user accepts it. Selecting `MAX(ruleGuid)`
per category chooses arbitrary content and can put findings under the wrong description.

Use BIRT/scripted grouping with cached rule lookups or a verified preprocessing layer for exact
long-text grouping. Distinguish key domains for rule-backed and issue-text-backed groups, null versus
literal "null", and separate tuple fields unambiguously. If hashing is used, verify full content on
collisions. Assess memory and query count; do not repeat a multi-kilobyte SAST description per issue
unless the user explicitly chooses that tradeoff.

## Step 5: Handle Engine And Mapping Diversity

Keep raw engine codes for predicates, joins and presence checks. Expose display labels separately:
for example, raw `SCA` may be displayed as `Fortify SAST`; it does not mean open-source dependency SCA.
Observed `SECURITYSCOPE`/`WEBINSPECTAGENT` aliases require version-specific confirmation before merging.

Present engines in category tables come from that category's selected issues, not a fixed pivot.
For Executive Summary, distinguish "these engines have scan results" from a full engine inventory.
One observed inventory combined standard Fortify engines with enabled parser metadata and marked
presence from scans. Verify lifecycle/version fields such as `lastAction` and `lastUsedOfKind`; they
are not a universal enabled-plugin API. If inventory cannot be established, use an explicit
"Results Present:" list rather than implying absent engines do not exist.

`issueAbstract` may contain per-finding HTML, plain text or Markdown; `issueRecommendation` and
`snippetId` may be empty for parser findings. Inspect available channels rather than assuming every
engine has SAST taint/source/sink data. Use plain-text rendering for untrusted labels; use a vetted
allowlist sanitizer for rich content. Regex HTML detection is not sanitization.

External list reports in the observed bundle mapped `mappedCategory` through `catpacklookup` to
`catpackexternalcategory`/`catpackexternallist`. This is distinct from rule metadata and from grouping
in Developer Workbook. Unmapped categories should be visible as unmapped, not silently counted as
"compliant". Do not assign OWASP categories by kingdom alone or by substring guesses.
Supplemental metadata support must be confirmed for the SSC version; do not modify vendor-managed
ExternalMetadata files or catpack DB rows as part of report editing.
