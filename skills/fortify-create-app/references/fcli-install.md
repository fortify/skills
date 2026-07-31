# fcli Installation & Upgrade Reference

Source: https://fortify.github.io/fcli/latest/

---

## Option 1: Using `@fortify/setup` (Recommended)

> **Provenance**: `@fortify/setup` is the official OpenText Fortify npm package, published under the `@fortify` npm scope by the Fortify team. Verify at https://www.npmjs.com/package/@fortify/setup.

Requires Node.js (npx). Works in CI/CD pipelines and local development. Automatically
downloads, verifies (RSA signature), and configures fcli.

**Linux / Mac:**
```bash
npx @fortify/setup env init --tools=fcli:auto
source <(npx @fortify/setup env shell)
fcli --version
```

**Windows PowerShell:**
```powershell
npx @fortify/setup env init --tools=fcli:auto
npx @fortify/setup env powershell | Invoke-Expression
fcli --version
```

**Pin to a specific version (recommended for CI/CD):**
```bash
npx @fortify/setup@2 env init --tools=fcli:v3.23.0
```

**Install additional tools alongside fcli:**
```bash
npx @fortify/setup env init --tools=fcli:auto,sc-client:25.4
```

**Upgrade (same commands):**
```bash
npx @fortify/setup env init --tools=fcli:v3    # latest v3.x
```

---

## Option 2: Manual Installation

### Download

Visit https://github.com/fortify/fcli/releases and download the appropriate asset:

| Asset | Platform |
|-------|---------|
| `fcli-windows.zip` | Windows (native binary) |
| `fcli-linux.tgz` | Linux (native binary) |
| `fcli-mac.tgz` | macOS (native binary) |
| `fcli.jar` | Any platform with Java 17+ |

> Note: Some browsers block `fcli-windows.zip` downloads — bypass the warning if prompted.

### Verify Integrity (required for production)

Each asset has a corresponding `.sha256` and `.rsa_sha256` signature file.

```bash
# Verify RSA signature (Linux/Mac)
openssl dgst -sha256 \
  -verify <(curl -s https://raw.githubusercontent.com/fortify/tool-definitions/main/id_rsa.pub) \
  -signature fcli-linux.tgz.rsa_sha256 \
  fcli-linux.tgz
```

### Managed Installation (internet-connected systems)

Managed installations place all Fortify tools under a shared `bin` directory.

```bash
# 1. Bootstrap using the downloaded binary (adjust path as needed)
/path/to/fcli tool definitions update

# 2. Install latest fcli to managed location
/path/to/fcli tool fcli install -v latest

# 3. Add the managed bin directory to PATH
#    Default: ~/fortify/tools/bin  (Linux/Mac)
#             %USERPROFILE%\fortify\tools\bin  (Windows)

# 4. (Linux/Mac only) Enable tab completion
source ~/fortify/tools/bin/fcli_completion
```

List available or installed fcli versions:
```bash
fcli tool fcli list
```

### Basic Installation

If not using managed installation:
1. Extract the archive (or use `fcli.jar` directly)
2. Copy `fcli` / `fcli.exe` to a directory on your PATH
3. For `fcli.jar`, create a wrapper script: `java -jar /path/to/fcli.jar "$@"`
4. (Linux/Mac only) Enable tab completion: `source <extraction-dir>/fcli_completion`

---

## Upgrade

### Via `@fortify/setup`
Re-run the init command with the desired version:
```bash
npx @fortify/setup env init --tools=fcli:v3   # latest v3.x
```

### Via managed install
```bash
fcli tool definitions update
fcli tool fcli install -v latest --uninstall all

# Or upgrade within a major version only
fcli tool fcli install -v 3 --uninstall 3
```

---

## Docker

fcli Docker images are available: https://hub.docker.com/repository/docker/fortifydocker/fcli

---

## ScanCentral Client (sc-client) Installation

The ScanCentral Client is needed for packaging source code for SAST scanning. Do not rely on a pre-installed `scancentral` binary unless in an airgapped environment. Use fcli to install and run the appropriate version:

```bash
# Install (latest version)
fcli tool sc-client install -y

# Install a specific version
fcli tool sc-client install -v 25.4 -y

# Run the installed client
fcli tool sc-client run -- package -bt gradle -o package.zip
# Or run a specific version
fcli tool sc-client run -v 25.4 -- package -bt gradle -o package.zip
```

**Version selection depends on the target platform:**

| Target | Which version | How to determine |
|--------|--------------|------------------|
| **FoD** | Latest (default) | `fcli tool sc-client install -y` |
| **SSC / SC-SAST** | Sensor-compatible | `fcli sc-sast sensor list --latest-only` → use the "Compatible client version" value |

For SSC/SC-SAST, using an incompatible sc-client version may cause scan failures. Always check sensor compatibility first.

---

## Notes

- **Native binaries** are preferred for performance and auto-completion, but if you encounter unexplained errors, try the `.jar` version — native binary issues are platform-specific.
- **Java requirement** for `.jar`: Java 17+
- **Proxy**: configure via `fcli config proxy add` or environment variables (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`)
