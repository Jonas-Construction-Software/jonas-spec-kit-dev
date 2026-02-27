---
description: Generate an actionable, dependency-ordered tasks.md for the feature based on available design artifacts.
handoffs: 
  - label: Analyze For Consistency
    agent: speckit.analyze
    prompt: Run a project analysis for consistency
    send: true
  - label: Implement Project
    agent: speckit.implement
    prompt: Start the implementation in phases
    send: true
scripts:
  sh: scripts/bash/check-prerequisites.sh --json
  ps: scripts/powershell/check-prerequisites.ps1 -Json
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Outline

1. **Setup**: Run `{SCRIPT}` from repo root and parse FEATURE_DIR and AVAILABLE_DOCS list. All paths must be absolute. For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

   **Multi-Repository Workspace Detection**:
   - Detect if running in a multi-repository workspace by checking for multiple `.git` directories in sibling folders or a workspace configuration file
   - If multi-repo workspace detected:
     - Identify which repository contains the current feature branch based on `FEATURE_DIR` path
     - Load `project-context.md` from the active repository if it exists
     - Check `plan.md` for cross-repository dependencies or integration points
     - Scan for `project-context.md` files in sibling repositories mentioned in the plan
     - Note any cross-repository task dependencies that must be sequenced

2. **Load design documents**: Read from FEATURE_DIR:
   - **Required**: plan.md (tech stack, libraries, structure), spec.md (user stories with priorities)
   - **Optional**: data-model.md (entities), contracts/ (interface contracts), research.md (decisions), quickstart.md (test scenarios)
   - Note: Not all projects have all documents. Generate tasks based on what's available.
   
   **Architectural Context Integration** (if multi-repo workspace):
   - If `plan.md` references cross-repository integration:
     - Load relevant `project-context.md` files to understand existing contracts
     - Identify shared APIs or contracts that tasks must maintain compatibility with
     - Note repository boundaries that affect task organization
   - If `contracts/` directory exists:
     - Check if contracts are consumed by other repositories
     - Validate contract changes don't break existing integrations
     - Add contract versioning tasks if breaking changes are needed

3. **Execute task generation workflow**:
   - Load plan.md and extract tech stack, libraries, project structure
   - Load spec.md and extract user stories with their priorities (P1, P2, P3, etc.)
   - If data-model.md exists: Extract entities and map to user stories
   - If contracts/ exists: Map interface contracts to user stories
     - **Multi-Repo**: Identify contracts consumed by other repositories
     - **Multi-Repo**: Add contract validation tasks for cross-repo dependencies
   - If research.md exists: Extract decisions for setup tasks
   - Generate tasks organized by user story (see Task Generation Rules below)
   - Generate dependency graph showing user story completion order
     - **Multi-Repo**: Include cross-repository dependencies in the graph
   - Create parallel execution examples per user story
   - Validate task completeness (each user story has all needed tasks, independently testable)
     - **Multi-Repo**: Validate cross-repo integration points have explicit tasks

4. **Generate tasks.md**: Use `templates/tasks-template.md` as structure, fill with:
   - Correct feature name from plan.md
   - Phase 1: Setup tasks (project initialization)
   - Phase 2: Foundational tasks (blocking prerequisites for all user stories)
     - **Multi-Repo**: Include cross-repository contract compatibility checks if needed
   - Phase 3+: One phase per user story (in priority order from spec.md)
   - Each phase includes: story goal, independent test criteria, tests (if requested), implementation tasks
   - Final Phase: Polish & cross-cutting concerns
     - **Multi-Repo**: Verify cross-repository contract compatibility
   - All tasks must follow the strict checklist format (see Task Generation Rules below)
   - Clear file paths for each task
   - Dependencies section showing story completion order
     - **Multi-Repo**: Document cross-repository task dependencies
   - Parallel execution examples per story
   - Implementation strategy section (MVP first, incremental delivery)

5. **Report**: Output path to generated tasks.md and summary:
   - Total task count
   - Task count per user story
   - Parallel opportunities identified
   - Independent test criteria for each story
   - Suggested MVP scope (typically just User Story 1)
   - Format validation: Confirm ALL tasks follow the checklist format (checkbox, ID, labels, file paths)
   
   **Cross-Repository Task Summary** (if multi-repo workspace):
   - List repositories affected by tasks in this feature
   - Identify tasks that require changes in multiple repositories
   - Note cross-repository integration tasks and their sequencing requirements
   - Document contract compatibility tasks
   - Recommend coordination points between repositories during implementation

Context for task generation: {ARGS}

The tasks.md should be immediately executable - each task must be specific enough that an LLM can complete it without additional context.

## Task Generation Rules

**CRITICAL**: Tasks MUST be organized by user story to enable independent implementation and testing.

**Tests are OPTIONAL**: Only generate test tasks if explicitly requested in the feature specification or if user requests TDD approach.

### Checklist Format (REQUIRED)

Every task MUST strictly follow this format:

```text
- [ ] [TaskID] [P?] [Story?] [Repo?] Description with file path
```

**Format Components**:

1. **Checkbox**: ALWAYS start with `- [ ]` (markdown checkbox)
2. **Task ID**: Sequential number (T001, T002, T003...) in execution order
3. **[P] marker**: Include ONLY if task is parallelizable (different files, no dependencies on incomplete tasks)
4. **[Story] label**: REQUIRED for user story phase tasks only
   - Format: [US1], [US2], [US3], etc. (maps to user stories from spec.md)
   - Setup phase: NO story label
   - Foundational phase: NO story label  
   - User Story phases: MUST have story label
   - Polish phase: NO story label
5. **[Repo] label**: REQUIRED for multi-repo tasks only
   - Format: [repo-name] where repo-name is the repository folder name
   - Include ONLY when task affects a different repository than the current one
   - Helps identify cross-repository coordination requirements
6. **Description**: Clear action with exact file path

**Examples**:

- ✅ CORRECT (single-repo): `- [ ] T001 Create project structure per implementation plan`
- ✅ CORRECT (single-repo): `- [ ] T005 [P] Implement authentication middleware in src/middleware/auth.py`
- ✅ CORRECT (single-repo): `- [ ] T012 [P] [US1] Create User model in src/models/user.py`
- ✅ CORRECT (single-repo): `- [ ] T014 [US1] Implement UserService in src/services/user_service.py`
- ✅ CORRECT (multi-repo): `- [ ] T015 [US1] [shared-contracts] Update TaskDto in shared-contracts/src/models/task.ts`
- ✅ CORRECT (multi-repo): `- [ ] T020 [P] [US2] [api-gateway] Add task endpoint proxy in api-gateway/src/routes/tasks.ts`
- ❌ WRONG: `- [ ] Create User model` (missing ID and Story label)
- ❌ WRONG: `T001 [US1] Create model` (missing checkbox)
- ❌ WRONG: `- [ ] [US1] Create User model` (missing Task ID)
- ❌ WRONG: `- [ ] T001 [US1] Create model` (missing file path)

### Task Organization

1. **From User Stories (spec.md)** - PRIMARY ORGANIZATION:
   - Each user story (P1, P2, P3...) gets its own phase
   - Map all related components to their story:
     - Models needed for that story
     - Services needed for that story
     - Interfaces/UI needed for that story
     - If tests requested: Tests specific to that story
   - Mark story dependencies (most stories should be independent)
   - **Multi-Repo**: If story spans repositories, group tasks by repository within the story phase
   - **Multi-Repo**: Add sequencing constraints for cross-repo dependencies (e.g., "shared-contracts updates must complete before api-service updates")

2. **From Contracts**:
   - Map each interface contract → to the user story it serves
   - If tests requested: Each interface contract → contract test task [P] before implementation in that story's phase
   - **Multi-Repo**: Identify contracts consumed by other repositories
   - **Multi-Repo**: Add contract compatibility validation tasks for cross-repo consumers
   - **Multi-Repo**: If breaking changes needed, add contract versioning and migration tasks

3. **From Data Model**:
   - Map each entity to the user story(ies) that need it
   - If entity serves multiple stories: Put in earliest story or Setup phase
   - Relationships → service layer tasks in appropriate story phase
   - **Multi-Repo**: Identify entities shared across repositories
   - **Multi-Repo**: Document which repository owns each entity's definition
   - **Multi-Repo**: Add data synchronization tasks if entities are replicated

4. **From Setup/Infrastructure**:
   - Shared infrastructure → Setup phase (Phase 1)
   - Foundational/blocking tasks → Foundational phase (Phase 2)
     - These MUST complete before ANY user story work begins
     - Examples: DB schema, auth framework, base API structure, shared utilities
     - **Multi-Repo**: Include cross-repository contract setup
     - **Multi-Repo**: Add repository connection/discovery setup if needed
   - Story-specific setup → within that story's phase

5. **From Cross-Repository Dependencies** (if multi-repo workspace):
   - Identify tasks that require coordinated changes in multiple repositories
   - Sequence tasks to ensure contracts are updated before consumers
   - Add validation tasks to verify cross-repo integration after changes
   - Document repository dependencies in the Dependencies section

### Phase Structure

- **Phase 1**: Setup (project initialization)
- **Phase 2**: Foundational (blocking prerequisites - MUST complete before user stories)
  - **Multi-Repo**: Include cross-repository contract compatibility checks
- **Phase 3+**: User Stories in priority order (P1, P2, P3...)
  - Within each story: Tests (if requested) → Models → Services → Endpoints → Integration
  - Each phase should be a complete, independently testable increment
  - **Multi-Repo**: Group cross-repository tasks within story phases
  - **Multi-Repo**: Add cross-repo contract compatibility checkpoints
- **Final Phase**: Polish & Cross-Cutting Concerns
  - **Multi-Repo**: Verify cross-repository contract compatibility

## Cross-Repository Task Patterns (Multi-Repo Only)

When tasks span multiple repositories, apply these patterns:

### Pattern 1: Contract Updates
```markdown
- [ ] T015 [US1] [shared-contracts] Update interface in shared-contracts/src/api/task.ts
- [ ] T016 [US1] [shared-contracts] Version bump in shared-contracts/package.json
- [ ] T017 [US1] Consume updated contract in src/services/task-client.ts
- [ ] T018 [US1] Validate contract compatibility in src/services/task-client.ts
```

### Pattern 2: Parallel Repository Changes
```markdown
- [ ] T025 [P] [US2] [api-service] Add endpoint in api-service/src/routes/users.ts
- [ ] T026 [P] [US2] [frontend] Add API client in frontend/src/api/users.ts
```

### Pattern 3: Sequential Cross-Repo Dependencies
```markdown
- [ ] T030 [US3] [data-models] Define schema in data-models/src/schemas/order.ts
- [ ] T031 [US3] [data-models] Build and publish package
- [ ] T032 [US3] [order-service] Install updated data-models package
- [ ] T033 [US3] [order-service] Implement order service using new schema
```
