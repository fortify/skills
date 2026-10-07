# Environment, Runtime And Seed Bundles

## Step 0: Choose Accessible Durable Locations

Establish paths before copying licensed resources, creating a runner or rendering output. First reuse
a location the user already selected. Otherwise, a clearly relevant private report workspace may be
selected automatically if it is safe, writable and accessible through the actual authorized tools;
announce that choice. If ownership, durability, exposure or access is unclear, confirm with the user
rather than choosing an unrelated workspace folder or a temporary destination by convenience.

Prefer a report project directory in the IDE's existing workspace when appropriate, not the public
skills repository. Keep licensed runtimes/seed resources, credentials and customer output out of
public repositories; use separate private resource/secret paths as needed. Do not add workspace
roots, change folder trust, modify tool scopes or alter agent/editor configuration to gain access.
A nonpublic external report directory can also be appropriate if the user and viewer can access it.

Announce resolved absolute paths for working files, final output directory, template/package filename,
checkpoints and disposable runtime/scratch material. Use unique names and preserve existing files.
Final designs/packages and the test record should live in a durable user-agreed location; temporary
directories may disappear on cleanup/reboot and should not hold the only delivered copy. If scratch
is temporary, agree on the final destination and copy only verified deliverables there before handoff.
Read-back/checksum the delivered copy and identify it separately from render outputs and checkpoints.

### Step 0a: Prefer Direct Artifact Access

Use editor file links, authorized filesystem viewers or an available PDF/document viewer first.
Check the actual viewer's access to a representative harmless file before committing to an output
location; terminal access does not prove browser/editor access. Diagnose the specific denial: a
browser's trusted-folder error is not proof that Autopilot blocks all local filesystem access.
If access is denied, explain it and ask for an accessible approved location or a user-opened file.
Do not respond by changing trust settings or silently serving the denied path over HTTP.

An HTTP preview/download server is a fallback only after the user approves its purpose, exact artifact
exposure and lifetime and the environment's restrictions permit it. Serve only an isolated output
directory or exact allowlisted filenames; bind to loopback, disable directory listing and deny path
traversal. Never expose the working tree, secrets, runtime jars, logs, checkpoints or unrelated files.
Loopback is not authentication: other local processes/users or IDE port forwarding can expose
customer content. Confirm forwarding/access assumptions; do not bind publicly, create tunnels or
publish files without separate approval. Stop the server after agreed use, state if it remains running,
and provide durable file paths independently of its URLs. A server must never bypass access controls.

## Step 1: Locate Compatible Report Resources

Use the smallest authorized source that supplies the template and its dependency closure:

| Source | Manual route | Automation route |
|--------|--------------|------------------|
| Existing local bundle | User selects an extracted directory or report seed ZIP | List archive entries, identify the active template, extract into a separate working directory |
| SSC distribution/local installation | Locate the matching report seed package through installation documentation or scoped file listing | Copy the known archive or report-resource directory without modifying the installation |
| SSC administrator interface | Use the installed version's documented seed/template export or download capability, if available | Use a documented read-only API only after verifying its endpoint and permissions |
| FCLI authenticated SSC session | Select report resources by name/ID | Discover installed `fcli ssc report` template/library/resource capabilities and inspect dependency coverage |
| Kubernetes/container SSC | Discover the authorized namespace, pod/container and actual installation path | Read-only archive stream or `kubectl cp` of the known bundle/resource directory |
| SSC DB resource storage | Ask an administrator to identify the stored report/bundle object | Inspect the matching schema, then export only the required blob through a read-only client |
| Vendor distribution/download | Obtain the correct version through an authorized vendor channel | Download only when access and license terms permit; record version/checksum |

Do not assume every SSC version exposes an export API or stores the original ZIP verbatim in a blob.
Database exports may be individual designs/libraries, not an importable seed bundle.
Use SSC's installed API documentation or activate `fortify-ssc`/`fcli-common` for command discovery;
do not guess endpoints. Upload/import operations require separate approval.

Before extracting, list archive contents and reject absolute paths, parent traversal and unsafe links.
Use a safe extraction tool into an isolated destination; do not overwrite an installation or user files.

```bash
unzip -l "$SEED_ZIP"
# After checking member paths and selecting a safe extraction destination:
unzip "$SEED_ZIP" -d "$REPORT_WORKDIR"
```

A seed bundle observed in SSC 26.2 contained trend/KPI reports (`issueTrend.rptdesign`, `kpi.rptdesign`),
project and correlation reports, issue workbooks, and WIE/external-list reports. Its resources included
`FortifyBasicElements.rptlibrary`, `FortifyReportLibrary.rptlibrary`, `FortifyWIEElements.rptlibrary`,
`FortifyF360DataSource.rptlibrary`, supporting `.js` files, and
`ReportDefinitions.xml`/`ReportLibraryDefinitions.xml`. These are examples, not a universal manifest.
Choose a relevant report family and resolve the actual target template through report definitions;
do not infer it from file size, filename, or a previously customized workbook.

Keep styles, includes, datasource definitions, scripts, images and registration entries together.
For new reports, use these as reference implementations, not files to redistribute under this skill's license.

### Step 1a: Discover Report And Resource Capabilities With FCLI

Activate `fortify-ssc` for SSC session/context and mutation safety, and `fcli-common` when common
installation, credential-profile, command-discovery, output/query or variable-chaining guidance is
needed. Load the relevant references from the owning skill after activation; do not duplicate their
generic procedures or use cross-directory links. Reuse confirmed session/context facts. Start report
operations under the installed command group, where most/all relevant capabilities are expected:

```bash
fcli ssc report -h
```

Inspect help for relevant discovered subcommands before constructing an invocation. Determine which
operations this version supports: template/library/resource listing, metadata/parameters, download,
configuration scaffolding, upload/registration, generation, status/wait and output download. Do not
pin a fixed command list or assume library transfer is unavailable because older versions lacked it.
Use documented library download/upload commands if discovered, subject to the same approval rules.
If a capability is missing, follow the owning SSC skill's supported discovery/fallback workflow or
obtain resources from another authorized source; never guess command names, flags or API endpoints.

Inspect what each download actually contains; a template file is not proof that all inherited
libraries/scripts/assets were included. Resolve that dependency closure, use unique local destinations,
and inspect actual registration/parameter metadata. Do not bypass a missing dependency with copied
connection credentials or treat a resource archive as an importable seed bundle.

### Step 1b: Optional SSC Integration Test

Prefer the local BIRT runner for repeated edits. SSC template creation/upload and report generation
are optional mutations that consume server resources; read-only DB permission does not authorize them.
Require explicit scope and approval before creating/uploading a template and before generating a report.
Do not replace/delete an existing template or import a seed bundle implicitly.

Use the capabilities discovered in Step 1a, not remembered upload/generation syntax. Inspect actual
configuration and parameter identifiers/types, register only agreed templates/dependencies, and use
the returned IDs. Obtain separate generation approval, verify completion or recorded errors through
the available status/wait operations, then download the agreed output to the durable destination.
Report SSC results separately from local renders. Library uploads, replacement of shared resources,
and deletion of test templates/reports also require explicit scope/approval.

## Step 2: Obtain A Matching BIRT Runtime

Preferred order:

1. Runtime and JDBC driver from the target SSC version, obtained from a local install or approved pod.
2. Compatible BIRT Designer/runtime already supplied by the user.
3. Official public Eclipse BIRT distribution, with its own dependencies, after checking version compatibility.

SSC may ship patched BIRT jars; public BIRT is useful for development but does not prove SSC parity.
Do not mix arbitrary jar versions, assume the newest Designer is compatible, or download JDBC drivers
from unofficial sites. Record Java, BIRT and driver versions and use the Java version they require.

In one SSC 26.2 container, the runtime was under
`/app/ssc/WEB-INF/externallib/birtrunner/`, and a MariaDB JDBC jar was under `/app/ssc/WEB-INF/lib/`.
Its patched BIRT version was 4.16. These paths/versions are illustrative: discover the actual ones.

Example read-only copy, after the user has authorized cluster access and supplied the paths:

```bash
kubectl -n "$SSC_NAMESPACE" exec "$SSC_POD" -c "$SSC_CONTAINER" -- \
  ls "$SSC_RUNTIME_DIR"
mkdir -p "$RUNTIME_WORKDIR"
kubectl -n "$SSC_NAMESPACE" cp -c "$SSC_CONTAINER" \
  "$SSC_POD:$SSC_RUNTIME_DIR" "$RUNTIME_WORKDIR/runtime"
```

`kubectl cp` needs `tar` in the container. If unavailable, copy known individual files through an
approved binary-safe exec/export mechanism. For an entire known directory, an exec `tar` stream can
be used if `tar` exists, with pipeline failure detection and a new isolated extraction destination.
Do not run broad filesystem searches or dump container configuration to locate credentials.
Local-install equivalents copy the corresponding runtime/dependency directory and JDBC jar.

For the public alternative, consult the official [Eclipse BIRT site](https://eclipse-birt.github.io/birt-website/)
and its release documentation. Do not invent a release URL or assume a single jar is a complete runtime.

## Step 3: Establish Read-Only SSC DB Access

Any approved interface is acceptable: native client, JDBC, DB MCP server, or a user-supplied export.
The client need not be the same interface used by BIRT, but test JDBC behavior separately. Decide on
one approved credential source for both before asking the user to enter a password.

| Database | Example discovery/test interfaces |
|----------|-----------------------------------|
| MySQL | `mysql`, approved MySQL/MariaDB JDBC driver, read-only DB MCP tools |
| Microsoft SQL Server | `sqlcmd`, SSMS, Microsoft JDBC driver, read-only DB MCP tools |
| Oracle | SQLcl/SQL*Plus, Oracle JDBC driver, read-only DB MCP tools |

Prefer an existing OS secret store, integrated authentication or approved secret-provider bridge
usable by both the native client and JDBC. Where appropriate, a user-configured session environment
can supply both without another prompt; environment variables are not encrypted storage and can be
visible to sufficiently privileged processes. Do not put secrets in command arguments, shell history,
debug tracing, logs or report files. Use equivalent wallet/integrated authentication for Oracle/SQL Server.

MySQL login-path storage is client-specific: `mysql --print-defaults` masks its password and does not
provide a supported reusable JDBC credential source. Do not configure it as the sole credential path
and only later discover that the renderer needs a second entry. Do not scrape/decrypt stored secrets
or enumerate unrelated profiles. A one-off interactive `mysql ... --password` also does not configure JDBC.

### Step 3a: Choose Credential Setup By Tool Capabilities

Never request a secret through chat or an agent question tool. Do not use password values supplied
in chat, even when the user requests their use. Warn once that the password is exposed, cannot be
used by this workflow and should be rotated; do not repeat it or rotate it yourself. Continue with
the generic credential flow below, independent of whether a password appeared in chat. Do not put
the chat value into a command, environment assignment, file, prompt response or tool argument.

First check for an existing authorized credential source usable by both the native client and JDBC.
If setup is needed, choose it from the tools' declared capabilities, not from chat disclosure:

| Available capability | Setup route |
|----------------------|-------------|
| Protected interactive input entered directly by the user, without passing through the model | Use that channel to configure the shared source; never collect or relay its value through chat/tools |
| Noninteractive tools, unknown prompt support, or secrets would pass through the model | Give exact user-executed setup commands or use an approved secret-provider launcher; do not start a blocking password prompt |

A terminal tool alone does not prove protected prompt support. Resolve the absolute working directory,
runtime/classpath and runner credential interface before asking the user to run setup. Explain how
both agent-launched client and renderer processes can access the same source: an export in a separate
user terminal does not automatically configure the agent's terminal. Confirm readiness without asking
for the secret. If a prompt fails, is cancelled or cannot be answered, stop retries and provide the
manual setup route. Prefer existing stores/session setup; a plaintext password file is only an
explicitly approved fallback under Step 3b, not the default merely because prompts are unavailable.

For a Bash user terminal, one possible shared session setup is below. Resolve every path/variable
in the handoff and implement the runner's environment lookup before suggesting its invocation:

```bash
cd /absolute/private/report-workdir
read -r -s -p 'SSC DB password: ' SSC_REPORT_DB_PASSWORD
printf '\n'
export SSC_REPORT_DB_PASSWORD
MYSQL_PWD="$SSC_REPORT_DB_PASSWORD" mysql --host="$DB_HOST" --port="$DB_PORT" \
  --user="$DB_USER" --database="$DB_NAME"
java -cp "$BIRT_RUNTIME_DIR/*:$JDBC_DRIVER_JAR:." ReportRunner \
  "$REPORT_PATH" "$OUTPUT_PATH" "$PARAMETERS_PATH"
```

This is user-executed setup, not an agent-mediated password prompt. The Java runner reads the same
environment value; it must not independently prompt or save it. Run both commands in that configured
terminal, or use an agreed secret-provider launcher for child processes; an export in the user's
terminal does not automatically appear in the agent's separate terminal. Account for shell/platform
differences and warn about environment exposure; use this fallback only with the user's agreement.
Keep the shared source available for agreed report iterations. After the user confirms completion,
unset the task-owned environment value in its owning terminal; the agent cannot unset a separate
user terminal's environment on their behalf.

### Step 3b: Bound Credential Lifetime And Cleanup

Prefer a protected credential store or shared session environment. If neither works for both tools,
an explicitly approved, temporary plaintext file can let the DB client and runner reuse one entry.
Create it outside repositories, resource bundles and served directories with owner-only permissions
from creation; disclose the exact path and bounded lifetime, and keep it out of checkpoints, archives
and logs. Owner-only permissions are not encryption. Agree on a cleanup owner and whether it is shared
across report iterations; temporary must not silently mean immediate deletion or indefinite retention.

For a user-run Bash setup, choose a private, non-existing path and do not overwrite an existing file.
This example creates it with mode `0600` under the current umask; it does not encrypt the password:

```bash
secret_file="$HOME/.ssc-report-task-secret"
(
  umask 077
  set -o noclobber
  IFS= read -r -s -p 'SSC DB password: ' report_password || exit 1
  printf '\n' >&2
  printf '%s' "$report_password" > "$secret_file" || exit 1
  unset report_password
)
```

If the path already exists, choose another approved path rather than removing or replacing it.
In the same user-controlled shell, load the file into the agreed runner environment variable and use
the corresponding documented client mechanism; for MySQL, for example. MySQL discourages
`MYSQL_PWD` because process-status tools may expose it; use that mechanism only if this exposure is
accepted, otherwise use a protected source supported by both tools.

```bash
(
  export SSC_REPORT_DB_PASSWORD="$(< "$secret_file")"
  MYSQL_PWD="$SSC_REPORT_DB_PASSWORD" mysql --host="$DB_HOST" --port="$DB_PORT" \
    --user="$DB_USER" --database="$DB_NAME" &&
  java -cp "$BIRT_RUNTIME_DIR/*:$JDBC_DRIVER_JAR:." ReportRunner \
    "$REPORT_PATH" "$OUTPUT_PATH" "$PARAMETERS_PATH"
)
```

The runner must read the same environment variable without prompting, persisting it, or logging it.
Environment variables may be visible to sufficiently privileged processes; keep shell tracing off and
use this only with the user's agreement. Resolve the actual client/runner interfaces and exact paths
before giving commands. A file created in a user terminal is not automatically available to an agent
terminal; confirm both tools can access the approved source. Remove only this task-created file after
the user confirms completion and authorizes cleanup, or when its agreed expiry requires it.

Close JDBC connections/tasks and discard process-local secret copies after each run, but do not make
a renderer automatically delete a shared credential file or clear the development session's source.
At final handoff, if this task created a temporary credentials file, identify the generated report
artifact and ask the user to verify it. Then offer two explicit choices: "Continue report development"
(retain the file within its approved lifetime) or "Output verified; report complete; remove the
temporary credentials file" (authorize removal of that task-created file). Do not infer a choice from a successful test,
artifact handoff, status request, or iteration failure. Remove only after the user chooses completion;
verify removal without displaying contents. If no choice is given, leave the file in place only within
its agreed lifetime and report its path, expiry, and cleanup owner. A previously agreed expiry or
emergency cleanup rule still applies; disclose it rather than silently retaining credentials past it.

If a crash, missing completion confirmation or handoff prevents cleanup, identify retained material,
its approved expiry/cleanup owner and the exact removal command; do not claim it was removed.
Never delete preexisting user credentials. Do not promise secure erasure on SSDs/backups. Redact
secret values from the test record and memory.

### Step 3c: Verify Read-Only Enforcement

For MCP, verify the connected server/database, the tool's supported operations, and the enforced
read-only role. A tool named "query" is not proof that mutations are blocked.
If the supplied account has write privileges, do not alter grants. Use and verify an explicit
read-only transaction for every client/JDBC check (for MySQL, `START TRANSACTION READ ONLY`, then
rollback); the JDBC read-only flag alone is not sufficient. If the chosen interface cannot enforce
the required restriction, ask for a suitable read-only profile rather than silently using unrestricted access.

Inspect only relevant table/view definitions, query metadata and narrowly scoped sample data.
MySQL `DESCRIBE`/`SHOW CREATE VIEW`, SQL Server catalogs, and Oracle dictionary views are different;
environment discovery may use vendor-specific read-only commands. SQL embedded in a portable report
must still meet the cross-database contract.

## Step 4: Record The Reproduction Environment

Keep nonsecret facts with the agreed test record: SSC/seed/Java/BIRT/JDBC versions, DB vendor,
report-resource paths, datasource strategy, locale and the parameters relevant to the report's scope
(for example, selected versions, date ranges, dimensions or a filter set).
Do not publish customer names, URLs, issue bodies or credentials in this skill/repo.

A local test against one SSC instance cannot reproduce errors on another by reading the local logs.
Ask for the failing instance's nested error or reproduce with the same report/runtime/schema versions.
Expired or removed temporary directories are normal: reestablish tools rather than assuming old paths exist.

### Step 4a: Maintain Reusable Report-Development Memory

Read existing task-relevant memory before rediscovering resources. During work and at handoff, store
facts that can shorten future sessions in an appropriately scoped persistent memory file. Prefer the
assistant's existing private user/project memory facility; inspect and update a suitable file rather
than creating duplicate notes. If persistent memory is unavailable, agree on a durable private notes
file/location with the user. Do not place environment/customer-specific memory in this public skill
repository, report resource bundles or temporary scratch, or change agent configuration to enable it.

Useful entries include:

- User-confirmed style/layout, logo/palette references, ToC/generic-content choices and output-format
  preferences, identifying the report/project they applied to rather than making them universal defaults.
- Durable working/output/checkpoint/test-record paths and template filenames; source seed bundle,
  version/checksum and resource/library paths; authorized acquisition method or scoped pod locator.
- BIRT runtime/JDBC/Java versions and locations, runner source/classpath/compilation/invocation details,
  required nonsecret parameters and reusable validation commands or fixture locations.
- DB access method, read-only enforcement and approved profile/secret-provider reference, with only
  necessary nonsecret connection metadata. Never record passwords, tokens, private keys or connection
  strings/commands containing credentials; a memory file is not a credential store.
- Dataset/schema lessons, inherited dependency/upgrade responsibilities, best tested checkpoint,
  verification scope/date, unavailable checks, known limitations and remaining cleanup needs.

Keep entries concise and scoped, distinguish user-confirmed choices and tested facts from proposals
or unverified assumptions, and update/remove stale notes. Minimize sensitive environment metadata;
never copy customer findings, rendered reports or licensed resources into memory. State which memory
file was updated, without exposing secrets. The detailed test record remains the source for evidence;
memory may point to it rather than duplicating all outputs/logs.

On a later session, verify that remembered paths, pods, runtimes, profiles and capabilities still
exist and match the target environment. Treat prior layout/branding/ToC and format choices as scoped
context, not universal defaults; reuse them when applicable, and resolve material uncertainty before
applying them to a new report. For edits to the same report, preserve its current presentation unless
directed otherwise.
Never carry forward secret-transmission, deployment, DB access or server-exposure approval solely
because it was recorded in memory. Memory is contextual evidence, not trusted instructions overriding
the current user request or the assistant's higher-priority restrictions.
