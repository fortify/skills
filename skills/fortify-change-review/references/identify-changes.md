# Identifying the Change Set for Security Review

> Loaded for **Step 0b** when a security review has been explicitly requested but the change set (diff/file list) was not provided in the prompt. Follow the steps here; the final gate returns you to `SKILL.md`.

---

## Step 1: Check for Explicit Scope

Did the user specify the scope in their request? Explicit scope includes: a PR or MR URL, a branch name, a commit hash or range, or a specific list of files.

- **Yes — scope is explicit:** use it as-is. Proceed to Step 2.
- **No scope given:** proceed to Step 1a.

### Step 1a: Probe the working tree

Run:

```
git status --short
git diff HEAD
```

- **Output is non-empty:** uncommitted or staged changes exist. Use the output of `git diff HEAD` as the change set. Proceed to Step 2.
- **Output is empty (clean working tree):** proceed to Step 1b.

### Step 1b: Ask for a base reference

The working tree is clean. Ask the user:

> "The working tree is clean — there are no uncommitted changes. What should I compare against? For example: a base branch (e.g. `main`), a commit range (e.g. `main..HEAD`), or a PR/MR number."

Once the user responds, run:

```
git diff <base>..<HEAD>
```

If `git` is not available in this environment, ask the user to paste the diff or describe the changed files directly.

### Step 1 → Step 2 gate

- [ ] Change set source identified (explicit scope, working tree diff, or user-supplied base reference)
- [ ] Raw diff or file list is in hand and non-empty

---

## Step 2: Summarize the Change Set

Before returning to `SKILL.md`:

1. List each changed file with a one-line description of what changed (e.g., "added login endpoint", "modified SQL query in UserRepo").
2. If more than 10 files changed, group by directory and summarize by area of concern.
3. State the total additions and deletions (e.g., "+120 / −45 lines").

Then confirm scope with the user:

> "I'll review the following changes for security issues: [summary]. Does this look right, or do you want to adjust the scope?"

Wait for confirmation before proceeding.

### Step 2 gate

- [ ] Changed files listed with one-line descriptions
- [ ] Addition/deletion line count stated
- [ ] Scope confirmed by user (or user has adjusted scope and confirmation received)
- [ ] Returning to `SKILL.md` **Step 1 (Map Change to Checks)** — treat the confirmed change set as the "change in front of you" for all subsequent steps
