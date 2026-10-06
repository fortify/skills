# Rendering, Diagnostics And Checkpoints

## Step 1: Prepare A Reproducible Renderer

Use a scratch directory with the report and its dependency closure, a matching BIRT runtime, and
an approved JDBC driver. Keep these out of the public skills repo. A GUI preview and a command-line
runner are complementary: automate repeatable checks and inspect the rendered layout visually.

SSC normally injects its datasource settings/resources. A standalone renderer needs an equivalent
configuration; copying the template alone is insufficient. Prefer explicit read-only JDBC access
and resource paths over embedding passwords in `.rptlibrary` files.

Generate a small Java runner using the target runtime's APIs. Its required behavior is:

1. Obtain JDBC URL/user/credentials from approved environment/secret configuration, not arguments
   echoed into shell history or a committed file; avoid logging them on errors.
2. Use a read-only DB account and request `Connection.setReadOnly(true)`. This flag alone is not a
   security boundary: verify server-side permissions and do not run unreviewed report scripts.
3. Create `EngineConfig`, set the report-resource directory and parent classloader, then start BIRT.
4. Inject a caller-owned JDBC connection into engine/task context if the selected runtime supports it.
5. Open the design and create an `IRunAndRenderTask`; pass parameters in their proper Java types.
6. Select HTML or PDF render options, run, inspect `getErrors()` and all nested causes, and exit nonzero
   on thrown exceptions or recorded task errors.
7. Close the task, destroy the engine, shut down BIRT and close JDBC in cleanup, including failure paths.

The following API pattern worked with SSC's observed patched BIRT 4.16 runtime. Verify equivalent
behavior for other runtimes rather than assuming these app-context keys are universal:

```java
EngineConfig config = new EngineConfig();
config.setResourcePath(reportResourceDirectory);
config.getAppContext().put("PARENT_CLASSLOADER", ReportRunner.class.getClassLoader());
config.getAppContext().put("OdaJDBCDriverPassInConnection", connection);
config.getAppContext().put("OdaJDBCDriverPassInConnectionCloseAfterUse", false);
Platform.startup(config);
IReportEngineFactory factory = (IReportEngineFactory) Platform.createFactoryObject(
    IReportEngineFactory.EXTENSION_REPORT_ENGINE_FACTORY);
IReportEngine engine = factory.createReportEngine(config);
IRunAndRenderTask task = engine.createRunAndRenderTask(engine.openReportDesign(reportPath));
task.getAppContext().put("OdaJDBCDriverPassInConnection", connection);
task.getAppContext().put("OdaJDBCDriverPassInConnectionCloseAfterUse", false);
task.setParameterValues(parameters);
task.setRenderOption(renderOptions);
task.run();
if (!task.getErrors().isEmpty()) {
    throw new IllegalStateException("BIRT recorded task errors; inspect nested causes in protected diagnostics");
}
```

This is a configuration excerpt, not a complete runnable program. Include imports, credential handling,
type validation, safe error reporting and cleanup in the generated runner. Use JDBC driver/server TLS
requirements; do not disable certificate validation to make a reproduction work.

After generating and reviewing the runner, typical POSIX compilation/invocation is:

```bash
javac -proc:none -cp "$BIRT_RUNTIME_DIR/*:$JDBC_DRIVER_JAR" ReportRunner.java
java -cp "$BIRT_RUNTIME_DIR/*:$JDBC_DRIVER_JAR:." ReportRunner <runner-specific-arguments>
```

On Windows the classpath separator is `;`, not `:`. Use the selected runtime's supported Java version.
Runner arguments can name input/output/parameter files but must not contain passwords.
Typical SSC parameters include `projectversionid` as a numeric value and `filterSetGuid` as a string;
read the actual template definition, since defaults, names and types vary. Do not infer required types
from a successful literal SQL query.

## Step 2: Establish A Baseline And Diagnostics

Run the untouched baseline first with identical resources, DB and parameters. Keep logs protected:
SQL errors can include connection strings, application names or finding bodies. Redact before sharing.

Validation tiers are separate:

| Check | What it proves |
|-------|----------------|
| XML/frontmatter parsing, IDs, names, dependency paths | Static structural consistency |
| Parameterized JDBC query and result metadata | Selected DB/driver query compatibility |
| BIRT task with zero recorded/thrown errors | Runtime wiring and execution in that environment |
| Rendered HTML/CSS/attributes | Actual labels, spans, styles and font/alignment behavior |
| PDF text/page checks plus visual inspection | Page placement, clipping/wrapping and visual acceptance |
| Render through target SSC | SSC integration including resources, registration and runtime parity |

Do not collapse these into one "tests passed" claim. Three-DB support requires either actual tests on
each target or a clear statement that portability review is not execution coverage.

An observed MariaDB driver emitted parameter-type-metadata warnings for both the original and edited
report. The baseline task still completed with zero errors. Compare warnings and task errors before
blaming the edit; do not filter all exception output or pipe a still-running renderer into `head`.

## Step 3: Verify Behavior And Layout

Use an HTML parser for rendered HTML, not an XML parser or ad hoc regex stripping. Inspect CSS and
inline/legacy attributes (`valign`, `align`, `rowspan`, `colspan`) as well as visible text.
Resolve inherited font/style values; a CSS-only assertion may miss valid HTML alignment attributes.

For PDF, use an available page-text extractor (`pdftotext`, an approved PDF library, or OpenPDF when
already in the BIRT runtime). Confirm section/category/table ordering and inspect the predecessor
page for blank/header-footer-only output. Page-text checks cannot prove visual alignment or clipping;
use rendered-page images or manual inspection for those requirements.

Acceptance examples derived from mixed-engine workbook development:

- Executive project name/version remain after replacing the engine component.
- Inventory displays enabled/installed engines only under the agreed semantics; standard absent
  engines get Not Present, or a present-only list has an explicit Results Present heading.
- Raw `SCA` can display as `Fortify SAST` without changing joins or identifiers.
- Each selected category appears, including plugin-only categories lacking rule text.
- Critical/High/Medium/Low and Total match independent SQL counts; audited values do too.
- Severity sum equals Total for a fixture with only the four recognized priorities; diagnose rather
  than hide unexpected/NULL priorities if the equality fails.
- Exactly one body Results Outline heading exists, separate from its TOC entry, including empty results.
- Section heading/first category and each category/summary/table share pages as agreed; no blank page appears.
- Spanning header cells are top-aligned, the widths fit, and colors/alternating rows match the seed style.
- Distinct descriptions are not collapsed merely because their category or rule identifier matches.

Include empty results and at least one populated branch for each new formatter/condition.
When a fixture lacks audited issues, supplement with a safe isolated formatter test or another
approved read-only fixture. Do not mutate SSC to manufacture audited findings.

## Step 4: Maintain Immutable Checkpoints

Before editing, save an unmodified baseline and required resource versions. For each successful step,
choose a unique noncolliding name such as `DeveloperWorkbook.<timestamp>-03-engine-counts.rptdesign`
in an agreed checkpoint directory outside the deployable bundle. Preserve changed libraries too.

Do not execute a render/check and then copy regardless of its exit code. Gate the copy explicitly on
all required checks succeeding. A command chain whose later `cp` runs after a failed assertion must
not create a "tested" checkpoint. Retrying validation successfully can establish a good candidate,
but record that outcome honestly.

Record: change, selected baseline, resources/runtime/DB, nonsecret parameters, checks, and failures or
unverified requirements. Reuse the user's test record; a new Markdown summary is not required.
If the user edits the report between steps, reread it and preserve those edits instead of applying an
old transformation blindly. Never overwrite checkpoints, auto-number by assuming old files vanished,
or reset the current report to `.orig` without authorization.

After a failed candidate, isolate the local cause. Prefer a single nearby read/test over broad
exploration. If evidence remains missing, retain the last good report and ask for the nested error,
runtime/schema facts or visual output needed to proceed.

## Step 5: Validate And Package The Handoff

For a template change, verify the complete report as well as the focused slice. For a shared-library
change, run the agreed dependent report set. Compare final user-visible identity, counts, descriptions
and navigation with the baseline to catch unrelated losses.

List the best checkpoint and how to reproduce its tests. Identify which SSC instance/DB/runtime was
used and which target environments remain untested. A local SSC runtime does not verify a different
customer's SSC deployment.
Package only agreed designs/libraries/scripts/assets/definitions, not runner jars, outputs, secrets,
test-only variants or rollback copies. Ask before uploading/importing or replacing installed resources.
