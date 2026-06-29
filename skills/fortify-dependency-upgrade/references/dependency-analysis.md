# Dependency Analysis Report Templates

Section templates for `reports/{package}-{old}-to-{new}-dependency-analysis.md`.
Workflow and ordering live in [../SKILL.md](../SKILL.md). Ecosystem-specific commands live in the sibling files in this folder.

> Model dependencies as a graph. Render trees only for inspection.

---

## §1. Target Package(s) to Upgrade

```markdown
# Dependency Graph Analysis: {Package} {OldVersion} → {NewVersion}

**Date:** {Current Date}
**Project:** {Project Name}
**Package Manager:** {npm/NuGet/pip/etc}
**Lockfile:** {See ecosystem reference}

## 1. Target Package(s) to Upgrade

| Package/Component | Current Version | Target Version | Type |
|-------------------|-----------------|----------------|------|
| {Package Name}    | {Old Version}   | {New Version}  | {Framework/Library/Tool} |

### Project Files with Dependencies
1. path/to/project1
2. path/to/project2
[List ALL manifest files — this is the checklist for §2 and §3]
```

---

## §2. Current Dependency Graph (Pre-Migration)

For **each** manifest file from §1, render a tree of direct + transitive deps. Include peer/optional/runtime/engine/platform constraints where applicable. Save the current lockfile state.

```markdown
## 2. Current Dependency Graph (Pre-Migration)

### Direct Dependencies (ProjectA/manifest)
ProjectA
├── PackageA x.y.z
│   ├── SharedPackage a.b.c ⚠️
│   └── PackageA.Core x.y.z
├── PackageB x.y.z
│   └── SharedPackage a.b.c ⚠️ (shared)
└── PackageC x.y.z
    └── SharedPackage d.e.f ⚠️ (version conflict!)

### Direct Dependencies (ProjectB/manifest)
[…repeat for every project file from §1…]

### Key Shared Dependencies (Pre-Migration)

| Package       | Version | Used By                |
|---------------|---------|------------------------|
| SharedPackage | a.b.c   | ProjectA, ProjectB     |
| OtherShared   | x.y.z   | ProjectA, ProjectC     |
```

---

## §3. Proposed Dependency Graph (Post-Migration)

Apply upgrade to manifests, regenerate lockfile, capture new graph. Do **not** build/test yet. Document the same project files as §2. Mark: 🔴 breaking, 🔧 required upgrade, `[REMOVED]`.

```markdown
## 3. Proposed Dependency Graph (Post-Migration)

### Direct Dependencies (ProjectA/manifest)
ProjectA
├── PackageA Y.Z.W 🔧 REQUIRED UPGRADE
│   ├── SharedPackage B.C.D 🔴 BREAKING CHANGE
│   └── PackageA.Core Y.Z.W
├── [REMOVED] PackageB 🔴
│   └── Reason: No compatible version exists
└── PackageC Y.Z.W
    └── SharedPackage B.C.D (now compatible!)

### Direct Dependencies (ProjectB/manifest)
[…repeat for every project file from §2 — counts must match…]

### Key Shared Dependencies (Post-Migration)

| Package       | Version   | Used By            |
|---------------|-----------|--------------------|
| SharedPackage | **B.C.D** | ProjectA, ProjectC |
```

---

## §4. Comparison: Old vs New Graphs

```markdown
## 4. Comparison: Old vs New Graphs

### Packages Added
| Package | Version | Reason |
|---------|---------|--------|

### Packages Removed
| Package | Old Version | Reason |
|---------|-------------|--------|

### Packages Upgraded
| Package | Old Version | New Version | Impact |
|---------|-------------|-------------|--------|

### Packages Downgraded
| Package | Old Version | New Version | Reason |
|---------|-------------|-------------|--------|

### Duplicated Packages
| Package | Versions Present | Reason |
|---------|------------------|--------|
```

Any downgrade or duplicated package requires explicit justification.

---

## §5. Compatibility Matrix

For each upgraded package, verify other direct deps allow overlapping versions, and that resolved transitive versions satisfy all required ranges. **Stop here if conflicts exist** — resolve before §6.

```markdown
## 5. Compatibility Matrix

### Critical Conflicts Identified

#### Conflict #1: {ConflictName}
Conflict:
  PackageA Y.Z.W → requires SharedLib >= B.C.D
  PackageB x.y.z → requires SharedLib ^a.b.c (incompatible!)

Resolution:
  Option 1: ✅ Upgrade PackageB to a version supporting SharedLib B.C.D
  Option 2: ❌ Downgrade PackageA (not recommended)
  Option 3: ⚠️ Use package manager override (requires justification)

Dependency Paths:
  Project → PackageA Y.Z.W → SharedLib B.C.D
  Project → PackageB x.y.z → SharedLib a.b.c [CONFLICT]

Impact:
  - What breaks if not resolved
  - Compile-time vs runtime failure

### Compatibility Matrix Summary

| Dependency | Old   | New   | Compatible?  | Action Required  |
|------------|-------|-------|--------------|------------------|
| SharedLib  | a.b.c | B.C.D | ❌ Breaking   | Upgrade PackageB |
| OtherLib   | x.y.z | X.Y.Z | ✅ Compatible | Direct upgrade   |
| ThirdLib   | 1.0.0 | 2.0.0 | ⚠️ Check      | Review API       |
```

**Decision point:**
- ✅ All conflicts resolved → proceed to §6
- ❌ Unresolved → adjust packages, return to §3
- ⚠️ Unresolvable → document why, consider alternatives

---

## §6. Validation Results

Prerequisite: all §5 conflicts resolved. Use commands from the relevant ecosystem reference.

```markdown
## 6. Validation Results

### Environment
- Clean workspace: ✅/❌
- Package manager version: {version}
- Additional requirements: [Docker, database, etc.]

### Build Validation
{restore command}   # Success/Failed
{clean command}     # Success/Failed
{build command}     # Success/Failed with X warnings

**Build Status:** ✅ PASSED / ❌ FAILED

### Baseline Test Results (Pre-Change — Step 1)

**Total:** X tests | Passed: X | Failed: X | Errors: X | Skipped: X

#### Pre-existing Failures (reported to user before any changes were made)

| Test | Failure Reason | Classification |
|------|---------------|----------------|
| {test_id} | {error summary} | Upgrade-adjacent / Unrelated |

> These failures existed before the upgrade. They are NOT regressions introduced by this change.
> The user was notified of these in Step 1.

### Affected Test Scope

Tests that import or exercise the upgraded package — identified in Step 1:

| Test File | Passes Baseline? | Post-Upgrade Status |
|-----------|-----------------|---------------------|
| {path/to/test_file.py} | ✅ / ❌ (pre-existing) | ✅ Pass / ❌ Fail / ⚠️ Error |

### Post-Upgrade Test Results

**Total:** X tests | Passed: X | Failed: X | Errors: X | Skipped: X

#### Newly Failing Tests (regressions introduced by the upgrade)

| Test | Scope | Failure Reason | Fix Applied |
|------|-------|---------------|-------------|
| {test_id} | Affected / Unexpected | {error summary} | {description or "Pending"} |

#### Tests Fixed by Upgrade (previously failing, now passing)

| Test | Notes |
|------|-------|
| {test_id} | {reason it now passes} |

#### Iteration Log

##### Iteration 1
Result: X/Y tests passed
Failures: [list]

##### Iteration 2 (After Fix)
Result: X/Y tests passed

**Test Status:** ✅ PASSED / ❌ FAILED

```

If validation fails: document failure, classify (upgrade-related in affected scope / unexpected outside scope / pre-existing), fix, return to §3, document each iteration.

---

## §7. Upgrade Report

```markdown
## 7. Upgrade Report

### Summary
- **Status:** ✅ Complete / ⚠️ Partial / ❌ Failed
- **Iterations Required:** {n}
- **Direct Dependencies Changed:** {n}
- **Transitive Dependencies Changed:** {n}
- **Build:** ✅/❌
- **Tests:** ✅/❌ {passing}/{total}
- **Breaking Changes:** {n} identified, {n} resolved

### Package Changes

#### Framework/Runtime Upgrades
- Component: OldVersion → NewVersion

#### Major Package Updates
[List major version changes]

#### Breaking Changes Fixed
1. [Each breaking change and how it was resolved]

### Why Multiple Iterations Were Required (if applicable)
**Root Cause:** […]
1. Initial approach issues
2. Conflicts discovered (runtime vs compile-time, transitive)
3. Lessons learned

### Manual Follow-up Items
1. 🔲 [Item]
2. 🔲 [Security updates needed]
3. 🔲 [Performance testing]
4. 🔲 [Documentation updates]

---
## Conclusion
[Outcome, key challenges, recommendations]

**Recommendation:** [Specific advice for next time]
```

---

## Rules

### Completeness ⚠️ CRITICAL
- Every manifest file from §1 must appear in §2 **and** §3 (counts must match).
- Common mistake: documenting one project and missing siblings.
- Use the ecosystem reference's enumeration command as your authoritative checklist.

### Process
- Complete §2–§5 (graph analysis) **before** running builds/tests in §6.
- Use the §5 compatibility matrix to drive upgrade decisions.
- Document every iteration when validation fails.

### Package Management
- Prefer minimal upgrades over broad upgrades.
- Do not delete or rewrite the lockfile unless necessary.
- Do not ignore peer dependency warnings — record them in §5.
- Do not use force/resolution/override fields without justification in §5.

### Failure Handling
Classify test failures as: API change · dependency resolution · peer conflict · environment. Document root cause in §6, return to §3 after fixes, summarize iterations in §7.

### Best Practices
- Keep changes small and reviewable.
- Commit manifest, lockfile, and this analysis document together.
- Reference the analysis document in PR descriptions.

---

## Why This Process Matters

Naive upgrades that touch only direct dependencies often fail because of hidden transitive conflicts that surface at runtime. Building the complete graph **before** changing code lets you:

1. Spot shared transitive dependencies that will conflict.
2. Identify additional packages that must move together.
3. Resolve conflicts on paper (cuts iteration count from 4+ to 1).
4. Make informed compatibility decisions.
