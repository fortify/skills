# Task: Explain NCD Concepts and Reporting Workflows

Use this workflow when the user asks for conceptual guidance, a process walkthrough,
or "help me understand" support for Fortify NCD reporting. This workflow is
**explain-only**: it clarifies concepts and workflows without running commands,
creating reports, or applying updates.

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)
- Do not execute `fcli` commands while handling an explain request unless the user
  explicitly switches to an execution task.

## Step 1: Load concepts and provide overview

Load [concepts.md](concepts.md), then display the literal text of the following sections, including sub-sections where applicable, with proper formatting:

1. **Skill overview**
2. **NCD definition**
3. **Reporting domain**
4. **Execution models**
5. **Roles**
6. **High-level workflow patterns (single-run vs federated)**

### Step 1 gate
- [ ] [concepts.md](concepts.md) loaded
- [ ] Literal text of all required sections displayed to user
- [ ] Output formatted in user-friendly way, for example with unnecessary line breaks within sentences removed
- [ ] No other information displayed to user

## Step 2: Exit explain mode

Return control to [../SKILL.md](../SKILL.md) and restart at Step 2 (Choose task)

### Step 2 gate
- [ ] Explain workflow exited and handed back to [../SKILL.md](../SKILL.md) Step 2
- [ ] Next task selection was explicitly requested after explain output