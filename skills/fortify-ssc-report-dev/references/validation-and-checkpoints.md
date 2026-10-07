# Rendering, Diagnostics And Checkpoints

## Step 1: Prepare A Reproducible Renderer

Follow [environment and seed resources](references/environment-and-seeds.md) for locations, runtime
acquisition, credential handling, and access approval. Use the agreed workspace/dependency set; GUI
preview and a command-line runner complement automated checks.

SSC normally injects its datasource settings/resources. A standalone renderer needs an equivalent
configuration; copying the template alone is insufficient. Keep connection settings out of the
report/library.

Generate a small Java runner against the target runtime. It must:

1. Read connection settings from the approved shared credential source, never from command arguments;
    do not log secrets. Use a server-authorized read-only account. `Connection.setReadOnly(true)` is not
    a security boundary, and unfamiliar report scripts must be reviewed before execution.
2. Configure `EngineConfig` with the resource directory and parent classloader, then start BIRT.
    Inject a caller-owned JDBC connection into engine/task context when supported by that runtime.
3. Open the design, create an `IRunAndRenderTask`, set typed parameters, and select the agreed SSC
    format's supported emitter/options (PDF, DOC, or XLS).
4. Inspect `getErrors()` and nested causes; fail on thrown exceptions or recorded errors. Assert
    rendered values/bindings, not merely task completion.
5. Close task, engine, BIRT, and JDBC resources on success or failure. Do not automatically remove
    shared credentials; follow the environment reference for cleanup and report retained material.

SSC supports PDF, DOC and XLS, not HTML. Verify the matching runtime/SSC pipeline's supported emitter
names/options; do not assume a file extension or an XLSX/DOCX emitter is equivalent to SSC's output.
Local HTML can be an optional diagnostic render, but must not become an acceptance target that drives
HTML-specific expressions, soft-break characters or styling changes in the shipped design. Fix and
verify layout in the requested SSC formats; if the local runtime cannot reproduce one, state that
gap and offer a separately approved final SSC render.

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
cd "$REPORT_WORKDIR"
javac --release "$JAVA_RELEASE" -proc:none -cp "$BIRT_RUNTIME_DIR/*:$JDBC_DRIVER_JAR" ReportRunner.java
java -cp "$BIRT_RUNTIME_DIR/*:$JDBC_DRIVER_JAR:." ReportRunner <runner-specific-arguments>
```

On Windows the classpath separator is `;`, not `:`. Check both `java -version` and `javac -version`;
the installed compiler and runtime may differ. Set `JAVA_RELEASE` to the target runtime's compatible
release, not the compiler's default. Replace placeholder paths/arguments with actual discovered values
when giving the user a setup/run command; show the working directory before asking them to run it.
Runner arguments can name input/output/parameter files but must not contain passwords.
Read actual parameter definitions/defaults/types; do not infer requirements from literal SQL alone.
For issue reports, implement the local version/filter-set inputs specified in this skill's
`references/portable-sql-and-ssc-data.md`; other report families may require different parameters.

## Step 2: Establish A Baseline And Diagnostics

Run the untouched baseline first with identical resources, DB and parameters. Keep logs protected:
SQL errors can include connection strings, application names or finding bodies. Redact before sharing.
If rendering is unavailable, run available static checks and label the candidate unrendered/unverified;
do not treat a failing baseline as a regression or claim a static-only checkpoint was rendered.

Validation tiers are separate:

| Check | What it proves |
|-------|----------------|
| XML/frontmatter parsing, IDs, names, dependency paths | Static structural consistency |
| Parameterized JDBC query or scripted-dataset checks | Selected data path and input/output compatibility |
| BIRT task with zero recorded/thrown errors | Runtime wiring and execution in that environment |
| Local HTML/CSS/attributes | Diagnostic evidence of rendered values/styles, not SSC format acceptance |
| PDF text/page checks plus visual inspection | Page placement, clipping/wrapping and visual acceptance |
| DOC/XLS content and visual checks | Requested SSC document/spreadsheet content, layout and cell correctness |
| Render through target SSC | SSC integration including resources, registration and runtime parity |

Do not collapse these into one "tests passed" claim. Three-DB support requires either actual tests on
each target or a clear statement that portability review is not execution coverage.
Even an error-free task can render blank values. Assert identity, labels, counts and representative
details in the artifact; use the layout reference's BIRT binding rules to diagnose missing content.

An observed MariaDB driver emitted parameter-type-metadata warnings for both the original and edited
report. The baseline task still completed with zero errors. Compare warnings and task errors before
blaming the edit; do not filter all exception output or pipe a still-running renderer into `head`.

## Step 3: Verify Behavior And Layout

If local HTML is used diagnostically, use an HTML parser, not an XML parser or ad hoc regex stripping.
Inspect CSS and inline/legacy attributes (`valign`, `align`, `rowspan`, `colspan`) as well as visible text.
Resolve inherited font/style values; a CSS-only assertion may miss valid HTML alignment attributes.

For PDF, use an available page-text extractor (`pdftotext`, an approved PDF library, or OpenPDF when
already in the BIRT runtime). Confirm section/group/content ordering and inspect the predecessor
page for blank/header-footer-only output. Page-text checks cannot prove visual alignment or clipping;
use rendered-page images or manual inspection for those requirements.
For DOC/XLS, inspect the supported document/spreadsheet with an appropriate reader, including
table/header spans, wrapping, merged cells, numeric versus text values and truncation where relevant.
Retain complete sink filenames and check line/priority values, not merely the presence of a table.

Derive assertions from the report purpose and changed feature. Check applicable items such as:

- Scope and identity remain correct across affected datasets; joins do not unexpectedly remove or
    multiply records.
- Tables, charts, and crosstabs reconcile with independent data at the defined grain.
- History boundaries, missing periods, units, weighting, denominators, rounding, and null/zero behavior
    match the report's definitions.
- Dimensions, conditional sections, sorting, locale, navigation, and layout meet the acceptance criteria.
- Empty results are intentional and do not erase required context.

Test empty and populated results for changed branches, plus relevant cases such as multiple versions,
date boundaries, missing periods, metadata selections, or zero denominators. Use isolated calculation
tests or approved read-only fixtures for missing cases; do not mutate SSC to create test data. Load the
optional issue-detail reference only for issue/engine/external-list behavior.

## Step 4: Maintain Immutable Checkpoints

Before editing, save an unmodified baseline and required resource versions. For each successful step,
choose a unique noncolliding name such as `<ReportName>.<timestamp>-03-metric-calculation.rptdesign`
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
runtime/schema facts or visual output needed to proceed. After three unsuccessful attempts at the
same behavior, stop patching and request the missing evidence.

## Step 5: Validate And Package The Handoff

For a template change, verify the complete report as well as the focused slice. For a shared-library
change, run the agreed dependent report set. Compare final user-visible context, measures, content
and navigation with the baseline to catch unrelated losses.

List the best checkpoint, reproduction commands, and tested SSC/DB/runtime/formats versus unavailable
checks. Local execution does not verify another deployment.
Package only agreed designs/libraries/scripts/assets/definitions, not runner jars, outputs, secrets,
test-only variants or rollback copies. Ask before uploading/importing or replacing installed resources.
Label a template/resource ZIP accurately; it is not an importable seed bundle without the actual
registration/bootstrap contract. For optional SSC testing and artifact/credential/server handoff,
follow the environment reference's discovered capabilities, approval boundaries and lifecycle rules.
Identify the verified durable artifact, not just a scratch copy or temporary URL.
