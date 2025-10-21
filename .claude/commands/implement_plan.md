---
description: Implement an approved technical plan from research documents with progress tracking
allowed-tools: Bash(git:*), Bash(moon:*), Read, Grep, Glob, Task, TodoWrite, Write, Edit, MultiEdit
---

You implement approved technical plans. These plans contain phases with specific changes and success criteria. You track progress by updating the original plan document.

## CRITICAL: IMPLEMENTATION WITH PROGRESS TRACKING
- UPDATE the original plan file with progress checkmarks and implementation notes
- LOG decisions, issues, and solutions directly in the plan document
- CREATE a comprehensive implementation record for future reference
- VERIFY each phase before proceeding to the next

## Getting Started

### Initial Response
**If plan path provided**: Read plan completely and begin implementation immediately.

**If no plan path provided**:
```
I'll help you implement an approved technical plan. Please provide the path to your plan file.

Example: `/implement_plan docs/thoughts/2025-01-15-PLAN-authentication-flow.md`

I'll read the plan, track progress in the document, and implement each phase systematically.
```

### Setup Process
1. **Read the plan completely** - understand all phases and success criteria
2. **Check for existing progress** - look for completed checkmarks (- [x])
3. **Read referenced files** - original research, CLAUDE.md files, mentioned code
   - **IMPORTANT**: Use Read tool WITHOUT limit/offset parameters for complete context
4. **Ensure latest code**: `git pull origin master && git pull origin $(git branch --show-current)`
5. **Create implementation todo list** using TodoWrite to track current session
6. **Add implementation log** to plan document for tracking decisions and progress

## Implementation Philosophy

Plans are carefully designed, but reality can be messy. Your job is to:
- **Follow the plan's intent** while adapting to what you discover
- **Implement each phase fully** before moving to the next
- **Verify your work** using the defined success criteria
- **Update the plan document** with progress and implementation notes
- **Maintain forward momentum** while ensuring quality

### When Plans Don't Match Reality
If you encounter mismatches between the plan and actual codebase:

1. **STOP and analyze** - understand why the discrepancy exists
2. **Document the issue** in the plan file
3. **Present the problem clearly**:
   ```
   Issue in Phase [N]:
   Expected: [what the plan says]
   Found: [actual situation]
   Why this matters: [impact on implementation]

   Proposed solution: [your recommendation]
   How should I proceed?
   ```

## Implementation Process

### Phase Implementation
For each phase in the plan:

1. **Mark phase as in-progress** in the plan document
2. **Implement changes** following platform conventions:
   - **Backend**: Use Moon commands (`moon run api:test`, `moon run api:lint`)
   - **Frontend**: Follow React/React Native patterns, add `data-cy` attributes
   - **Database**: Create migrations, update models, test with `moon run api:migrate`
   - **Tests**: Create RSpec and Cypress tests early, use FactoryBot patterns

3. **Track implementation decisions** - add notes to plan document about:
   - Code changes made
   - Patterns followed
   - Issues encountered and solutions
   - Deviations from original plan and why

4. **Verify phase completion**:
   - Run automated verification: `moon run [app]:test`, `moon run [app]:lint`, `moon run [app]:build`
   - Perform manual verification as specified
   - Update success criteria checkboxes in plan document

5. **Mark phase as complete** and move to next phase

### Progress Tracking Format
Add an "Implementation Log" section to the plan document:

```markdown
---
## Implementation Log

### Phase 1: [Phase Name] - [Date]
**Status**: ✅ Complete | 🚧 In Progress | ⏸️ Blocked

**Changes Made**:
- [file:line] - [description of change]
- [file:line] - [description of change]

**Decisions Made**:
- [decision and rationale]

**Issues Encountered**:
- [issue] - [solution]

**Verification Results**:
- ✅ `moon run api:test` - all tests pass
- ✅ `moon run hospital:tests` - Cypress tests pass
- ✅ Manual verification complete

**Notes**: [any additional context for future reference]

---
```

### Using Task Agents
Use Task agents sparingly and only for:
- **Targeted debugging** when implementation doesn't work as expected
- **Code pattern research** when you need to understand unfamiliar territory
- **Integration verification** when checking how new code fits with existing systems

Always specify exact directories:
- Backend: `services/api/app/[controllers|models|services]/`
- Frontend: `apps/hospital/src/`, `apps/mobile-*/src/`
- Database: `services/api/db/migrate/`, `services/hasura/`

## Verification Standards

### Automated Verification (must pass before proceeding)
- **Backend**: `moon run api:test` - all RSpec tests pass
- **Frontend**: `moon run [app]:tests` - all Cypress tests pass
- **Build**: `moon run [app]:build` - code compiles successfully
- **Linting**: `moon run [app]:lint` - code style and type checking pass

### Manual Verification
- **UI/UX**: Feature works as expected in browser/app
- **Performance**: Acceptable response times under normal load
- **Edge cases**: Error handling and boundary conditions work
- **Integration**: No regressions in related features

## Resuming Work

If the plan has existing checkmarks or implementation log entries:
- **Trust completed work** - don't re-implement finished phases
- **Review implementation log** - understand decisions and context
- **Pick up from first unchecked item** in current phase
- **Verify only if something seems inconsistent** with previous work

## Error Handling

### When Tests Fail
1. **Don't proceed** to next phase until all tests pass
2. **Document the failure** in implementation log
3. **Fix the issue** using platform debugging patterns
4. **Re-run verification** before continuing

### When Blocked
1. **Document the blocking issue** in plan file
2. **Mark phase as blocked** (⏸️) with explanation
3. **Present the problem clearly** with context and proposed solutions
4. **Wait for guidance** before proceeding

## Platform-Specific Patterns

### Backend Implementation (Ruby/Rails)
- **Models**: Add to `services/api/app/models/`, follow existing patterns
- **Controllers**: Add to `services/api/app/controllers/`, use services for business logic
- **Services**: Add to `services/api/app/services/`, keep methods short
- **Tests**: Mirror structure in `services/api/spec/`, use `create :model` patterns
- **Jobs**: Use `run_que_synchronously` in tests

### Frontend Implementation (React/React Native)
- **Components**: Follow existing patterns in `apps/[app]/src/`
- **Testing**: Add `data-cy` attributes, create Cypress tests
- **State**: Use existing state management patterns
- **Styling**: Follow app-specific style conventions

### Database Changes
- **Migrations**: Create in `services/api/db/migrate/`
- **Models**: Update in `services/api/app/models/`
- **GraphQL**: Update schema in `services/hasura/`
- **Testing**: Verify with `moon run api:migrate` and factory updates

## Critical Requirements
- **Progress Tracking**: Update plan document with implementation log for each phase
- **Verification**: Complete all success criteria before proceeding to next phase
- **Documentation**: Record decisions, issues, and solutions in plan file
- **Platform Consistency**: Follow Moon commands and existing code patterns
- **File Reading**: Always FULL reads (no limit/offset) before implementing
- **Forward Momentum**: Keep implementation moving while maintaining quality
- **Context Preservation**: Create detailed implementation records for future reference