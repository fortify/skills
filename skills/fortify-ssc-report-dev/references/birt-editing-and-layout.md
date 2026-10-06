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

Inheritance varies by report family: sampled WIE templates inherited parameters/components from
`FortifyWIEElements`, while project and workbook templates mixed local logic with common components.
External-list templates also reused dedicated libraries. Follow the actual dependencies rather than
assuming a previous report's ownership graph applies to the next one.

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

Illustrative binding failure: in one workbook, rebinding a table to a local dataset dropped aggregate aliases
`sca`, `wi`, `ss`, and `total`. The SQL itself executed correctly, but BIRT returned
`Invalid bound column name`. Copying the required bindings fixed it.
An outer-grid `Error.ReportQueryLoadingError` was only a wrapper for those nested errors.

## Step 3: Make Small Structural Changes

Within the selected dependency strategy, copy the required owning component and dependency closure.
For a targeted structural edit, start with the smallest suitable component. Inspect all content before
replacing it; a grid may contain identity, summary values or other unrelated cells as well as the target.

Use column tracks whose widths sum to the available table/page width. Check multirow headers, spans,
repeated headers, and dynamic columns or crosstab dimensions if present. For charts, preserve intended
axes, series, labels, legends and measurement units. Choose layout based on output format and purpose,
not a universal severity/engine table design.

Reuse appropriate seed styles where that matches the report's visual language. Verify resolved colors,
fonts and highlight rules rather than relying on style names. Match comparable labels and values;
severity colors are relevant only to severity-oriented displays. Readability of trend lines, legends
and aggregate dimensions may be the actual concern for other reports.

Place `verticalAlign=top` on spanning header cells when top alignment is the requested layout.
BIRT HTML may emit `valign="top"` rather than CSS `vertical-align`; check both.
Prefer plain-text expressions for labels and explicit line-break handling.
For any report rendering rich input, use a vetted allowlist sanitizer rather than treating arbitrary
stored text as trusted HTML. Bind data as values, not dynamically evaluated template/script source.
Use consistent report-generated description heading styles instead of fragile regex rewriting of
rich finding descriptions. Do not mislabel different content channels merely to make headings match.

## Step 4: Diagnose Pagination Locally

Inspect page-break settings on the heading, row, grid, list/table, group and relevant styles:
`pageBreakBefore`, `pageBreakAfter`, `pageBreakInside`, repeated headers, master page transitions,
and available page height. A keep-with-next chain can move a whole cluster to the next page.

Start with the smallest controlling break. Use PDF evidence after each change:

1. A section/group heading stays with the intended following content.
2. Repeated rows, charts or crosstabs paginate according to the agreed layout.
3. Header/content adjacency is maintained where available page height permits it.
4. No new blank page appears immediately before or after moved content.

As an optional example, a workbook was improved by keeping the category-title row with the following summary rows.
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
Choose representative consumers from the actual dependency graph and changed behavior: for example,
a trend report for a history-query change, a portfolio report for a scope helper, or a detail/table
report for a shared binding/layout change. Do not always select the same workbook/external-list set.

When handing off, include only changed deliverable resources and agreed definition entries.
Keep checkpoint designs outside seed-bundle import/discovery paths; otherwise a packaging process
may ship dozens of near-identical templates. Never hand off the temporary test variants as production.
