# Environment, Runtime And Seed Bundles

## Step 1: Locate Compatible Report Resources

Use the smallest authorized source that supplies the template and its dependency closure:

| Source | Manual route | Automation route |
|--------|--------------|------------------|
| Existing local bundle | User selects an extracted directory or report seed ZIP | List archive entries, identify the active template, extract into a separate working directory |
| SSC distribution/local installation | Locate the matching report seed package through installation documentation or scoped file listing | Copy the known archive or report-resource directory without modifying the installation |
| SSC administrator interface | Use the installed version's documented seed/template export or download capability, if available | Use a documented read-only API only after verifying its endpoint and permissions |
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

A seed bundle observed in SSC 26.2 contained `NewDevWorkbook.rptdesign`,
`DeveloperWorkBook.rptdesign`, `FortifyBasicElements.rptlibrary`,
`FortifyCompositeElements.rptlibrary`, `FortifyF360DataSource.rptlibrary`, supporting `.js` files,
and `ReportDefinitions.xml`/`ReportLibraryDefinitions.xml`. These are examples, not a universal manifest.
Check report definitions to determine the active workbook; do not infer it from file size or name.

Keep styles, includes, datasource definitions, scripts, images and registration entries together.
For new reports, use these as reference implementations, not files to redistribute under this skill's license.

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
The client need not be the same interface used by BIRT, but test JDBC behavior separately.

| Database | Example discovery/test interfaces |
|----------|-----------------------------------|
| MySQL | `mysql`, approved MySQL/MariaDB JDBC driver, read-only DB MCP tools |
| Microsoft SQL Server | `sqlcmd`, SSMS, Microsoft JDBC driver, read-only DB MCP tools |
| Oracle | SQLcl/SQL*Plus, Oracle JDBC driver, read-only DB MCP tools |

For example, `mysql --host="$DB_HOST" --port="$DB_PORT" --user="$DB_USER" --password`
lets the user type the secret directly into their terminal. Prefer an approved secret store or
protected client configuration for unattended runs. Never route interactive passwords through the agent.
Use equivalent integrated authentication/wallet/secret-store configuration for SQL Server and Oracle.

For MCP, verify the connected server/database, the tool's supported operations, and the enforced
read-only role. A tool named "query" is not proof that mutations are blocked.

Inspect only relevant table/view definitions, query metadata and narrowly scoped sample data.
MySQL `DESCRIBE`/`SHOW CREATE VIEW`, SQL Server catalogs, and Oracle dictionary views are different;
environment discovery may use vendor-specific read-only commands. SQL embedded in a portable report
must still meet the cross-database contract.

## Step 4: Record The Reproduction Environment

Keep nonsecret facts with the agreed test record: SSC/seed/Java/BIRT/JDBC versions, DB vendor,
report-resource paths, datasource strategy, locale, application version and filter-set parameters.
Do not publish customer names, URLs, issue bodies or credentials in this skill/repo.

A local test against one SSC instance cannot reproduce errors on another by reading the local logs.
Ask for the failing instance's nested error or reproduce with the same report/runtime/schema versions.
Expired or removed temporary directories are normal: reestablish tools rather than assuming old paths exist.
