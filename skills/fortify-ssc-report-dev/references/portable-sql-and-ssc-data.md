# Portable SQL And SSC Data Semantics

## Step 1: Set The Portability Contract

Default targets are MySQL, Microsoft SQL Server and Oracle. An explicitly customer-specific report may
use a narrower target; record that exception. Check supported DB versions for the target SSC release.
Inspect actual schema/view definitions and seed queries before adding joins. Seed reports use
version-dependent internal objects and helper scripts, not a stable public schema contract.

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

Execute on every target DB when available. If only one is available, perform portability review and
explicitly report the remaining engine tests as unverified; do not claim three-DB certification.

## Step 2: Establish Row Grain And Source Of Truth

Define what one source row and one output row represent: application/version, scan, snapshot,
measurement, correlated group, issue, metadata selection, or an aggregate. Choose the driving source
that matches that purpose, rather than always starting from current issues.

- Use join types that implement the intended inclusion rules. Optional enrichment must not remove
  required records; deliberate mandatory joins may legitimately restrict the population.
- Check join cardinality at the selected grain. Multiple scans, language rows, metadata values or
  history samples can multiply a measure even if the final table looks plausible.
- Keep identifiers scoped correctly; some reports select one version while others aggregate a portfolio.
- Separate keys used for selection/grouping from friendly display labels and translated names.
- Use the same defined population in independent validation queries; do not repair unknown duplicates
  by adding `DISTINCT` blindly.

Illustrative patterns observed in an SSC seed bundle:

| Example | Data pattern | Relevant development concern |
|---------|--------------|------------------------------|
| `issueTrend.rptdesign` | Snapshot-backed `measurementhistory` and `variablehistory`, with date parameters | Historical observations and interval/boundary semantics, not just current issue totals |
| `kpi.rptdesign` | Multi-version scope helpers, metadata dimensions and derived measures | Authorized portfolio selection, grouping and metric definition |
| `project_report.rptdesign` | Risk calculations, project metadata and first/last scan dates | Scope, units, aggregation level and missing scans |
| `AppSecCorrelation.rptdesign` | Scan-context and finding datasets, including latest-per-engine scan information | Appropriate scan selection and correlation/count meaning |
| `NewWIEVulnerability.rptdesign` | Library-inherited parameters/components | Follow the owning library rather than assuming all data logic is local |

These are reference choices, not prescribed templates or fixed schema contracts. Choose only the
family relevant to the requested report. Issue-detail and external-list examples are optional and
are routed separately from the main skill.

## Step 3: Preserve Scope, Time And Metric Meaning

Read the report's actual parameters, scripts and query logic. Only request a version/filter-set GUID
when the report uses them; portfolio selectors, date ranges, measurement definitions or metadata
filters may be the relevant inputs instead.
For issue reports inheriting `FortifyBasicElements.projectversionid`, normally preserve the companion
`FortifyBasicElements.filterSetGuid` parameter and its library/registration contract. Its library
definition is optional, distinct, string, static/simple text-box, unconcealed and unformatted, with
standard default GUID `a243b195-0a59-3f8b-1403-d55b7a7d78e6`; SSC updates this default for the selected
version. Reuse it where possible. A local definition must preserve these characteristics/default and
aligned dataset parameter metadata. Standalone BIRT must receive the intended version ID and GUID;
intentionally unfiltered reports require an explicit scope decision.
Use selected cache visibility and folder membership
consistently across category drivers, summary counts and detail rows; do not substitute global hidden
flags or select an arbitrary filter-set row. Verify one matching cache row per issue/version/filter set.
Folders are fully customizable: they are not the four fixed SSC criticalities. Where a criticality
breakdown is requested, use `friority` (Critical/High/Medium/Low) rather than folder labels or raw
severity, and flag invalid/missing priorities during validation instead of adding `Other`.

For history/trend reports, establish:

- Current state versus snapshot/measurement history or an as-of-date value.
- Date storage type/units, input timezone, inclusive/exclusive boundaries and ordering.
- Which observation is selected at an interval boundary; a latest sample before a date is not a sum.
- Whether absent periods mean no observation, zero, carry-forward or another explicitly defined behavior.
- Selection grain: latest overall scan, latest per version, latest per engine or all observations.

Do not assume scan/snapshot dates are SQL timestamps rather than numeric values. Review conversions
in JDBC parameters and report scripts, and preserve boundary conventions unless intentionally changed.
Stored historical values need not equal a recalculation from today's issue records.

For KPI/summary/correlation reports, define numerators, denominators, units, weighting, null handling,
rounding and the level at which each calculation is performed. Counts of findings, correlated groups,
URLs, scans or applications are different measures. Average severity, Fortify priority counts and
engine-specific scores are not interchangeable. Percentages with no denominator are not automatically
zero, and averaging already-aggregated percentages may produce a different portfolio result.

Verify dataset and presentation filters, user-selected dimensions, and authorization/scope helpers.
For example, sampled templates used `util.getProjectVersionIdsInClause(...)`; inspect the helper and
its expansion rather than replacing it with a fixed version ID or unrestricted query. Generated SQL
must preserve authorized scope, validate identifiers/values, and use binding wherever supported.

Issue visibility, audit flags, rule text, engine aliases and external mappings matter only when the
report uses those concepts. Apply their semantics consistently when relevant, not to every metric.

## Step 4: Trace The Full Data Pipeline

The SQL query is not always the final source of a displayed value. Follow scripted datasets, computed
columns, joint datasets, cube dimensions/measures, aggregations, chart expressions, sorting/filtering
and event handlers that affect it. Validate calculations at each relevant boundary.

Preserve parameter types/defaults and conditional sections. A chart or crosstab may need grouped data
at a different grain from a detail table; do not flatten or double-aggregate it accidentally.
Define deterministic grouping, locale and ordering appropriate to that output. For long text, apply
the portability restrictions only if the report actually groups or compares it.

## Step 5: Choose Purpose-Specific Data Checks

Use fixtures for empty and populated results plus the branches changed by the request. Examples are
multiple versions, several time periods with a missing interval, metadata selections, a zero denominator,
ties in scan selection, or joined records lacking optional enrichment. For issue reports, add the
requested visibility/engine/audit/description cases rather than requiring them for every report.

Reconcile displayed measures with source observations and the agreed calculation, including units and
rounding. Use independent fixtures or test-only report copies when production data lacks a branch;
never alter SSC data to manufacture a test. Such test copies are not deployable deliverables.
