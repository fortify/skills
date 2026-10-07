# BIRT Editing, Inheritance And Layout

## Step 1: Trace The Owning Element

Resolve the report-definition entry to its actual template, then follow `libraries`, `extends`,
datasets, component bindings, scripts and nested inherited elements. A thin `.rptdesign` may delegate
its behavior to several `.rptlibrary` files. Start at the failure, not at every file in the bundle.

Libraries define reusable parameters, datasets, layout, and styles. Before creating a report or
changing ownership, determine which components should be inherited, copied, or defined locally based
on compatibility and the task. Do not assume a blanket reuse preference specifies every component;
resolve only ownership decisions needed to proceed. Always retain the shared datasource reference
(`FEM360DB` in observed seeds; verify the actual name/namespace) and never copy its connection settings.

| Component | Starting choice | Main consideration |
|-----------|-----------------|--------------------|
| `FEM360DB` datasource | Always retain shared library/reference | SSC injects connection settings; never copy them |
| Parameters | Reuse/inherit when compatible | Usually lower risk, but type/default and SSC-injected behavior must match |
| Datasets | Reuse when scope/query/output contract fits | Medium risk: schema fixes help, while parameter, grain and result-binding contracts can change |
| Layout elements/styles | Reuse only if they fit the requested layout or can be safely overridden/customized | Higher risk: inherited components may bring nested layout, styles, scripts or behavior |

Record each component as inherited, copied, or newly defined. Inheritance requires actual library
includes/`extends` or theme/style references; matching local definitions do not count. Verify ownership
in XML and exercise inherited elements at runtime. Do not silently copy selected inherited components
to avoid dependencies or scripts. If a contract or layout is incompatible, explain the tradeoff and
obtain the user's decision before substituting a local alternative. For other components, choose a
compatible seed copy or local definition based on the established requirements; surface the choice if
it materially changes ownership or upgrade responsibility. Include the dependency closure (bindings,
styles, scripts, images, resources and IDs); retain only references proven unused. Copies require manual
porting of later library/schema fixes, so no option is upgrade-proof. Record source versions and
customizations. For shared-library edits, trace consumers and determine representative regressions.

For issue reports using `projectversionid`, normally retain `filterSetGuid`; its exact contract is in
[portable SQL and SSC data](references/portable-sql-and-ssc-data.md).

Inheritance varies by report family: sampled WIE templates inherited parameters/components from
`FortifyWIEElements`, while project and workbook templates mixed local logic with common components.
External-list templates also reused dedicated libraries. Follow the actual dependencies rather than
assuming a previous report's ownership graph applies to the next one.

For a new report, begin with a small compatible seed and a working datasource/parameter path, not the
largest available report. If registration is needed, inspect `ReportDefinitions.xml`,
`ReportLibraryDefinitions.xml`, and parameter types; do not invent XML.

### Step 1a: Establish Presentation And Resolve Styles

For a new report or requested redesign, establish presentation needs such as style, authorized branding,
assets, ToC, and generic content. Use the request and comparable seed reports to guide defaults; resolve
ambiguities that materially affect the design, and avoid adding unrequested boilerplate. A ToC may help
long or multi-section reports; there is no fixed page threshold. Do not fabricate branding. For existing
reports, preserve their style, logo, ToC, and boilerplate unless asked otherwise; a new element does not
imply redesign.

Match actual local/inherited style definitions, theme/highlight rules, and comparable elements, not
just style names. For new elements without a counterpart, use standard SSC styles unless directed
otherwise. Inspect the matching seed rather than guessing fonts/colors. Preserve familiar Critical/
High/Medium/Low color meaning unless the user directs otherwise. Do not edit a shared style library
for a single report. Verify fonts/substitution, contrast, hierarchy, spacing, and surrounding content
in requested PDF/DOC/XLS outputs.

For logos, verify resource path, format, proportions, and placement; package only authorized assets.
For ToC, verify bookmarks, labels, links, and page references where supported, including empty reports
and long labels. Keep the ToC entry distinct from its body heading; check for duplicate headings,
blank pages, or detached content. Record presentation choices and dependencies with the test record.

## Step 2: Preserve The BIRT Model

- Use a namespace-aware XML parser; verify well-formedness, unique element IDs and relevant name uniqueness.
- Preserve library namespaces and `baseId` links. New element IDs must not collide with existing ones.
- `overridden-values` is an element, not a named property. A helper that looks it up by `name` may
  silently discard introductory text, project identity, bindings or other overrides.
- Distinguish inherited data sources/datasets from local ones. Rebinding to a report-local dataset
  may require explicit bound-column definitions, including aggregate aliases used by footers.
- Update JDBC parameter positions, defaults, `paramName`, result metadata, computed columns and table
  bindings together. Do not assume a query override automatically updates them.
- BIRT `data` elements must reference a valid `resultSetColumn` with matching bound columns. For
  calculated display values, define a named expression binding and reference it; a standalone
  `valueExpr` can be ignored, yielding blank output even when the task records no errors. Assert
  actual rendered labels/values before declaring the binding successful.
- Check duplicate named properties as well as IDs; accidentally appending a second `dataType` is
  not a deliberate metadata update, even if a runtime happens to accept it.
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
Apply Step 1a/1b for presentation decisions and resolved styles. Validate layout in the agreed SSC
formats using this skill's `references/validation-and-checkpoints.md`; diagnostic HTML must not drive
format-specific template fixes. For metric/criticality semantics use the SQL reference, not color cues.

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
