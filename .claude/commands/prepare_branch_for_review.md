description: Prepare a branch for team pull review
allowed-tools: Bash(git:*), Bash(moon:*), Read, Grep, Glob, Task, TodoWrite, Write, Edit, MultiEdit
---

## Goals
- Remove debug logs and excessive comments
- Delete stray files
- Run lint & tests
- Check CLAUDE.md adherence
- Identify security/stability issues
- Ensure DRY (no duplication)
- Suggest commit message
- Remove any unnecessary backwards compatibility code that was left in

## Process
1. **Plan Intake**
   - If plan path given: Read fully, note progress, implement.
   - If not: Ask for plan path (e.g. `/implement_plan docs/2025-01-15-PLAN.md`).

2. **Discover ALL Changed Files**
   - FIRST, run `git diff --name-only master...HEAD` to get complete list of changed files
   - Store this list - it will be passed to ALL delegated tasks
   - NEVER assume which files have changed - always get the full list from git

3. **Delegated Tasks**
   - **IMPORTANT**: Pass the COMPLETE list of changed files from step 2 to each Task agent
   - **Task: Debug and Comment Cleanup** – scan & strip debug logs from ALL changed files. clean up unnecessary comments as well, especially ones that aren't meant for future reference (for example, delete "// no change needed", "// Trigger CI").
   - **Task: Backward Compatibility** – remove unnecessary backwards compatibility code from ALL changed files.
   - **Task: File Hygiene** – remove untracked/unwanted new files.
   - **Task: Code Quality** – run `moon run [app]:lint`, `moon run [app]:test`.
   - **Task: Compliance** – check CLAUDE.md rules on ALL changed files. Make sure any necessary version bumps are completed.
   - **Task: Review** – scan ALL changed files for security/stability issues.
   - **Task: DRY Audit** – flag duplication or redundancy across ALL changed files.
   - Always allowed to spawn additional `Task`s if deeper checks are needed (integration, debugging, pattern analysis, etc).

4. **Verification**
   - ✅ Tests pass (`moon run [app]:test`)
   - ✅ Linting passes (`moon run [app]:lint`)
   - ✅ Build clean (`moon run [app]:build`)
   - Manual: CLAUDE.md adherence, DRY, stability

5. **Progress Tracking**
   Append Implementation Log to plan:

```markdown
### Phase N: [Name] – [Date]
Status: ✅ | 🚧 | ⏸️
Changes: [summary]
Issues: [notes]
Verification: [results]
```

6. **When Blocked**
•	Stop, log mismatch.
•	Propose solution, request guidance.

**Output**
•	Updated plan with Implementation Log
•	Suggested a commit message for the entire branch / PR in chat
