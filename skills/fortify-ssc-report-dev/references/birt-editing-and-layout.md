# BIRT Editing, Inheritance And Layout

## Step 1: Trace The Owning Element

Resolve the report-definition entry to its actual template, then follow `libraries`, `extends`,
datasets, component bindings, scripts and nested inherited elements. A thin `.rptdesign` may delegate
its behavior to several `.rptlibrary` files. Start at the failure, not at every file in the bundle.

For one report, prefer template-local changes over modifying shipped libraries, but choose how much
library dependency to retain. For library maintenance, agree on affected consumers and test their
relevant branches. Record the chosen tradeoff before materializing large components.

### Dependency Strategy

| Strategy | Benefit | Upgrade responsibility |
|----------|---------|------------------------|
| Inherit library components/datasets | Small customization; receives library fixes, including query/schema adaptations | Library changes can break element IDs, overrides, layouts or binding contracts; test against updated libraries |
| Copy layout, retain dataset/library references | Isolates presentation while potentially receiving schema-related query fixes | Query output/parameter contracts, scripts and styles can still change; inspect retained dependencies |
| Copy required components and datasets, retain shared F360 datasource | More predictable standalone custom-report behavior when libraries change | Manually adapt copied queries to SSC schema changes and port relevant fixes from updated seeds |

For an individual report, the third option may make sense even though seed-library updates are
infrequent. It is not automatically safer for someone who cannot maintain the copied queries.
Neither inherited nor copied reports are guaranteed to survive an SSC/schema/runtime upgrade.

Keep the F360 DB datasource shared: in an observed seed this was `FEM360DB`, ultimately extending
`FortifyF360DataSource.FEM360DB`. Discover the actual names/namespace for the target version. Do not
embed copied installation credentials or replace SSC's datasource-injection contract accidentally.
Keep the necessary datasource library include even when other library references are removed.

Copy the dependency closure deliberately: nested components, datasets, parameters, computed/bound
columns, styles, cubes, master pages, scripts and images. Copying only the visible grid while leaving
transitive `extends` or script/style references does not provide full isolation. Preserve or package
required script/image resources without assuming they can all be inlined. Confirm unique IDs/names
and rewrite local references consistently; remove only library includes proven unused.

Record each copied element's source bundle/library version, customized behavior and retained references
in the agreed record. Keep a pristine comparison baseline. Licensed resources remain subject to their
distribution terms; copying into a customer template does not make them public skill assets.

An observed external-list report delegated most layout/data to
`FortifyExternalListIssueReport`, `FortifyExternalListCommonSections`,
`FortifyExternalListBasicElements`, and `FortifyBasicElements`. Developer Workbook had local sections
plus inherited components. Changing a shared element can affect both families and other reports.

For new reports, use a small compatible seed example with working datasource and parameters rather
than immediately cloning the largest report. If registration is required, inspect the installed
`ReportDefinitions.xml`/`ReportLibraryDefinitions.xml` structure and parameter types; do not invent XML.

## Step 2: Preserve The BIRT Model

- Use a namespace-aware XML parser; verify well-formedness, unique element IDs and relevant name uniqueness.
- Preserve library namespaces and `baseId` links. New element IDs must not collide with existing ones.
- `overridden-values` is an element, not a named property. A helper that looks it up by `name` may
  silently discard introductory text, project identity, bindings or other overrides.
- Distinguish inherited data sources/datasets from local ones. Rebinding to a report-local dataset
  may require explicit bound-column definitions, including aggregate aliases used by footers.
- Update JDBC parameter positions, defaults, `paramName`, result metadata, computed columns and table
  bindings together. Do not assume a query override automatically updates them.
- Do not duplicate or blindly copy Designer `designerValues` metadata; it can be stale after query changes.
- Use XML escaping/CDATA APIs correctly. XML entities, report JavaScript strings and rendered HTML are
  different encoding layers. Never globally replace names/engine codes across the whole report.
- Retain bookmark/TOC identifiers when relocating headings, and validate navigation after the move.

In one workbook, rebinding Engine Breakdown to a local dataset dropped inherited aggregate aliases
`sca`, `wi`, `ss`, and `total`. The SQL itself executed correctly, but BIRT returned
`Invalid bound column name`. Copying the required bindings fixed it.
An outer-grid `Error.ReportQueryLoadingError` was only a wrapper for those nested errors.

## Step 3: Make Small Structural Changes

Copy only the subcomponent needed to insert a group/row that cannot be added through inheritance.
When replacing a grid, inspect all its cells: an engine grid in the observed workbook also contained
Project Name and Project Version. Replacing the grid without preserving those rows removed identity.

Use column tracks whose widths sum to the available table/page width. A practical engine table has:

- Engine Type spanning both header rows.
- Category label spanning the severity columns, with `(audited / total)` in the same text channel.
- Critical/High/Medium/Low second-row labels.
- A Total column spanning the header rows.

Use shared seed styles when appropriate: observed examples included `headerTable0Small` (blue),
`friorityCritical`, `friorityHigh`, `friorityMedium`, `friorityLow`, and `alternatingRowColor`.
Verify actual colors and highlight rules; a style described as alternating can use two grays rather
than white/gray. Match font size between project identity and engine status rows, not merely family.

Place `verticalAlign=top` on spanning header cells when aligning Engine Type/Total with category text.
BIRT HTML may emit `valign="top"` rather than CSS `vertical-align`; check both.
Prefer plain-text expressions for engine/category labels and explicit line-break handling.
Use consistent report-generated description heading styles instead of fragile regex rewriting of
rich finding descriptions. Do not mislabel different content channels merely to make headings match.

## Step 4: Diagnose Pagination Locally

Inspect page-break settings on the heading, row, grid, list/table, group and relevant styles:
`pageBreakBefore`, `pageBreakAfter`, `pageBreakInside`, repeated headers, master page transitions,
and available page height. A keep-with-next chain can move a whole cluster to the next page.

Start with the smallest controlling break. Use PDF evidence after each change:

1. Category heading with Issue Summary and Engine Breakdown.
2. Results Outline section heading with its first category.
3. Engine Breakdown with the first description where the available page height permits it.
4. No new blank page immediately before or after the moved content.

An observed workbook was improved by keeping the category-title row with the following summary rows.
For its section-level split, removing a redundant list master page was insufficient. Moving Results
Outline into the first category's layout and removing the old outer forced break fixed the split.

Do not blindly reproduce that workaround. Confirm first-row numbering in the target runtime, ensure
the heading appears once, preserve the TOC link, and provide an empty-result fallback. A heading
inside a data-driven list disappears if the list has no rows unless that case is explicitly handled.
The table-of-contents entry is not a duplicate body heading; count each separately.

If a structural change produces a blank predecessor page, undo the redundant outer break rather than
accepting the shifted defect. Long rich descriptions can still require natural page breaks; do not
promise all content fits together when it exceeds page height.

## Step 5: Assess Upgrade And Bundle Impact

For an SSC upgrade, compare the original source seed with the new version before reusing a custom
report. Review schema/view and dataset changes, parameter/result types, visibility joins, datasource
injection, script/runtime compatibility and locale behavior. Port relevant fixes into copied datasets
manually, preserving custom behavior; do not blindly replace a customized query with the new seed query.
Recheck cross-DB support and rerun count/render/layout tests on the target SSC version.

For shared libraries, use the reverse dependency graph plus runtime checks of actual branches.
Suggested representative consumers include Developer Workbook, an OWASP list report, a CWE list
report, and a project report, when those templates use the changed component.

When handing off, include only changed deliverable resources and agreed definition entries.
Keep checkpoint designs outside seed-bundle import/discovery paths; otherwise a packaging process
may ship dozens of near-identical templates. Never hand off the temporary test variants as production.
