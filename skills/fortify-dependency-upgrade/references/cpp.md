# C++ Ecosystem Reference — CMake / Make / Conan / vcpkg

## External Research Sources (Step 4 — populate external research report first)

Before making any changes, fetch data from these official sources and record everything in
`reports/{package}-{old}-to-{new}-external-research.md`. Use the template in
[../SKILL.md](../SKILL.md) Step 4.

### Package Registry

```text
# Conan Center — confirm a specific version exists
https://conan.io/center/recipes/<package-name>
conan search <package-name> -r=conancenter

# vcpkg registry — confirm version
vcpkg search <package-name>
https://github.com/microsoft/vcpkg/tree/master/ports/<package-name>

# For source-based (Git) deps — confirm tag exists
git ls-remote --tags <repo-url>
```

### GitHub Releases & Changelog

```text
https://github.com/<org>/<repo>/releases
https://github.com/<org>/<repo>/blob/main/CHANGELOG.md
```

### Security Advisories

```text
# GitHub Advisory Database
https://github.com/advisories?query=<library-name>

# OSV database
https://osv.dev/list?q=<library-name>

# NVD CVE search
https://nvd.nist.gov/vuln/search/results?query=<library-name>
```

> **Record every URL you visit** in the Sources table of the external research report (template in [../SKILL.md](../SKILL.md) Step 4).
> Do not proceed to Step 5 (Breaking Changes) until the report file exists.

## Finding Project Files

```bash
# Find common C++ project/build files
find . -name "CMakeLists.txt" -o -name "Makefile" -o -name "conanfile.txt" -o -name "conanfile.py" -o -name "vcpkg.json"
```

## Dependency Analysis

### CMake

```bash
# Configure project and inspect dependency discovery
cmake -S . -B build

# Verbose build output
cmake --build build --verbose

# Show cached variables, including package paths
cmake -S . -B build -LAH
```

### Conan

```bash
# Install dependencies
conan install . --build=missing

# Show dependency graph
conan graph info .

# Check installed packages
conan list "*"
```

### vcpkg

```bash
# Install dependencies from manifest
vcpkg install

# List installed packages
vcpkg list

# Show package info
vcpkg search <package-name>
```

## Lockfile

* **CMake**: No standard lockfile
* **Make**: No standard lockfile
* **Conan**: `conan.lock`
* **vcpkg**: no lockfile — versions are pinned via `builtin-baseline` in `vcpkg.json` and/or a baseline in `vcpkg-configuration.json`

## Verify Target Version Exists

Check Conan Center:

```bash
conan search <package-name> -r=conancenter
```

Check vcpkg registry:

```bash
vcpkg search <package-name>
```

For source-based dependencies, verify the Git tag or release exists:

```bash
git ls-remote --tags <repo-url>
```

If the requested version is not listed, stop and tell the user it has not been published yet, then provide the latest available stable version.

## Dependency Tree

Inspect direct and transitive dependencies actually resolved by the package manager or build system.

```bash
# Conan
conan graph info .

# Conan with profile
conan graph info . --profile=default

# vcpkg manifest dependencies
vcpkg install --dry-run

# CMake dependency discovery
cmake -S . -B build --debug-find
```

## Run Tests

Sample commands for establishing a baseline and validating changes after an upgrade.

```bash
# CMake / CTest
cmake -S . -B build
cmake --build build
ctest --test-dir build --output-on-failure

# Make
make
make test

# Ninja
cmake -S . -B build -G Ninja
cmake --build build
ctest --test-dir build --output-on-failure
```

## Find Affected Code

Sample search commands for locating usages impacted by dependency changes.

**Find Renamed Class Usage**

```bash
grep -r "OldClassName" src/ include/ --include="*.cpp" --include="*.hpp" --include="*.h" --include="*.cc" --include="*.cxx"
```

**Find Changed Header Includes**

```bash
grep -r '#include <old/header.hpp>' src/ include/ --include="*.cpp" --include="*.hpp" --include="*.h" --include="*.cc" --include="*.cxx"
```

**Find Namespace Usage**

```bash
grep -r "old_namespace::" src/ include/ --include="*.cpp" --include="*.hpp" --include="*.h" --include="*.cc" --include="*.cxx"
```

**Find Macro Usage**

```bash
grep -r "OLD_MACRO" src/ include/ --include="*.cpp" --include="*.hpp" --include="*.h"
```

## Find Affected Tests

Identify every test file that includes or exercises the library being upgraded.
Run these **before any changes** to record the affected test scope.

```bash
grep -r "OldClassName\|old_namespace::\|#include <old/header.hpp>" tests/ \
  --include="*.cpp" --include="*.hpp" --include="*.h" --include="*.cc" --include="*.cxx"

# Run only the affected tests via CTest
ctest --test-dir build -R "affected_test_pattern" --output-on-failure
```

Report the resulting file list as the **affected test scope** for use in Step 7.

## ABI / Compiler Compatibility Checks

C++ dependency upgrades can break ABI compatibility even when APIs look similar.

```bash
# Check compiler version
c++ --version
gcc --version
clang++ --version

# Inspect linked shared libraries
ldd ./build/path/to/binary

# macOS equivalent
otool -L ./build/path/to/binary

# Inspect symbols
nm -C ./build/path/to/library.a
nm -D -C ./build/path/to/library.so
```

## Transitive / Peer Conflict Resolution

> Use the Dependency Tree section above to identify the conflict first, then apply one of the patterns below.

### Conan — override dependency version

```ini
[requires]
parent-lib/1.0.0
conflict-lib/2.0.0

[options]
```

Or in `conanfile.py`:

```python
def requirements(self):
    self.requires("parent-lib/1.0.0")
    self.requires("conflict-lib/2.0.0", override=True)
```

### vcpkg — pin dependency version

In `vcpkg.json`:

```json
{
  "dependencies": [
    {
      "name": "example-lib",
      "version>=": "2.0.0"
    }
  ]
}
```

Use a baseline in `vcpkg-configuration.json`:

```json
{
  "default-registry": {
    "kind": "git",
    "repository": "https://github.com/microsoft/vcpkg",
    "baseline": "<commit-sha>"
  }
}
```

### CMake — force a package path

```bash
cmake -S . -B build -DCMAKE_PREFIX_PATH=/path/to/dependency
```

Or inside `CMakeLists.txt`:

```cmake
set(CMAKE_PREFIX_PATH "/path/to/dependency")

find_package(example-lib REQUIRED)
target_link_libraries(my_target PRIVATE example-lib::example-lib)
```

### CMake — FetchContent specific version

```cmake
include(FetchContent)

FetchContent_Declare(
  example_lib
  GIT_REPOSITORY https://github.com/example/example-lib.git
  GIT_TAG v2.0.0
)

FetchContent_MakeAvailable(example_lib)
```

## Clean Rebuild

Use a clean build when changing compiler flags, dependency versions, toolchains, or ABI-sensitive libraries.

```bash
# Remove CMake build directory
rm -rf build

# Reconfigure and rebuild
cmake -S . -B build
cmake --build build

# Conan clean install
rm -rf build
conan install . --build=missing

# vcpkg rebuild package
vcpkg remove <package-name>
vcpkg install <package-name>
```
