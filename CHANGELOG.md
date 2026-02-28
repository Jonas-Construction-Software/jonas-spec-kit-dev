# Changelog

<!-- markdownlint-disable MD024 -->

Recent changes to the Specify CLI and templates are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.12] - 2026-02-27

### Added

- **Repository Branch Setup Scripts**: Added automated scripts for creating feature branches across multi-repository workspaces
  - `scripts/bash/setup-repo-branches.sh`: Bash implementation for Unix-like systems
  - `scripts/powershell/setup-repo-branches.ps1`: PowerShell implementation for Windows
  - Automatically detects multi-repo workspaces and creates matching feature branches
  - Parses `tasks.md` for `[repo-name]` labels to identify affected repositories
  - Handles both single-repo and multi-repo scenarios seamlessly

### Changed

- **Analyze Command Template**: Enhanced multi-repository awareness
  - Added comprehensive handoff templates for post-analysis workflow transitions
  - Improved documentation of handoff patterns and checklist generation
  - Enhanced cross-repository constraint validation

- **Implement Command Template**: Added explicit no-commit policy
  - Added **DO NOT commit changes** instruction in completion validation (step 9)
  - Added **DO NOT commit changes in any repository** for multi-repo scenarios (step 10)
  - Added prominent final reminder that all changes must remain uncommitted for user review
  - Prevents automatic commits after implementation, requiring explicit user approval

- **Tasks Command Template**: Improved task format validation
  - Enhanced `[Repo]` label requirement documentation
  - Clarified checklist format with better examples
  - Added explicit repository label examples for single-repo and multi-repo scenarios

- **Tasks Template**: Updated task structure documentation
  - Refined phase organization and task ordering guidelines
  - Enhanced parallel task execution rules
  - Improved repository label format documentation

## [0.1.10] - 2026-02-27

### Added

- **Multi-Repository Workspace Support in Plan and Tasks Workflows**: Extended multi-repo capabilities from clarify workflow to plan and tasks workflows
  - **Plan Workflow (`plan.md`)**:
    - Added multi-repository detection to identify workspace structure and active repository location
    - Loads `project-context.md` files from multiple repositories to understand system-wide architecture
    - Pre-populates technical context with known constraints from architectural documentation
    - Validates plan against cross-repository architectural constraints
    - Generates cross-repository impact summary listing affected repositories and integration points
    - Documents shared contracts, data models, and service boundaries that span repositories
    - Includes research tasks for cross-repository integration patterns
    - Ensures contracts align with existing repository contracts when cross-repo dependencies exist
  - **Tasks Workflow (`tasks.md`)**:
    - Detects multi-repository workspaces and identifies active repository from `FEATURE_DIR` path
    - Loads architectural context from `project-context.md` files in relevant repositories
    - Maps cross-repository dependencies from plan to task organization
    - Validates contract changes don't break existing integrations in other repositories
    - Adds `[Repo]` label to task format for multi-repo tasks (e.g., `- [ ] [TaskID] [P1] [S1] [repo-name] Description`)
    - Includes cross-repository dependencies in task dependency graph
    - Generates contract validation and compatibility check tasks for shared APIs
    - Documents cross-repository task summary with coordination requirements
  - Maintains full backward compatibility with single-repository projects

- **Jira Development Task Integration in Specify Workflow**: Enhanced feature branch creation with optional Jira task number support
  - **Specify Workflow (`specify.md`)**:
    - Added upfront prompt for Jira development task number before branch creation (e.g., FT-53, DEV-142, PROJ-789)
    - Validates Jira key format using pattern `[A-Z]+-\d+`
    - Supports two branch naming modes:
      1. **Jira mode**: Uses dev task key as prefix (e.g., `FT-53-user-auth`)
      2. **Auto-numbering mode**: Uses sequential numbers (e.g., `001-user-auth`)
    - Ensures consistent branch and directory naming based on selected mode
    - Moved Jira prompt to beginning of workflow for better UX (before any resource creation)
  - **Feature Creation Scripts**:
    - Added `--custom-prefix` option to `create-new-feature.sh` (Bash)
    - Added `-CustomPrefix` parameter to `create-new-feature.ps1` (PowerShell)
    - Validates custom prefix format and provides clear error messages
    - Updated help documentation and usage examples for both scripts
  - **Benefits**:
    - Provides clear feature branch traceability back to Jira development tasks
    - Enables teams to quickly identify which Jira task a feature branch implements
    - Supports existing auto-numbering workflow for teams not using Jira
    - Improves project management integration and developer workflow

### Changed

- **Improved Specify Workflow Branch Naming Strategy**: Restructured branch creation logic for better clarity and maintainability
  - Separated branch naming strategy determination into explicit first step
  - Made Jira integration opt-in with clear Yes/No prompt at beginning
  - Consolidated branch numbering logic to avoid duplication
  - Enhanced documentation of branch creation process with clearer examples

## [0.1.8] - 2026-02-26

### Enhanced

- **Multi-Repository Workspace Support in Clarification Workflow**: Enhanced `clarify.md` template to intelligently handle multi-repository workspaces
  - Added multi-repository detection in setup phase to identify workspace structure and locate active repository
  - Loads `project-context.md` files from repositories to understand architectural constraints
  - Added new "Multi-Repository Context" taxonomy category for coverage scanning (cross-repo dependencies, shared contracts, service boundaries)
  - Prioritizes clarification questions about cross-repository integration points and architectural alignment
  - Validates clarifications against `project-context.md` constraints during integration
  - Reports architectural alignment and cross-repository dependencies in completion summary
  - Maintains backward compatibility with single-repository projects

## [0.1.6] - 2026-02-23

### Fixed

- **Parameter Ordering Issues (#1641)**: Fixed CLI parameter parsing issue where option flags were incorrectly consumed as values for preceding options
  - Added validation to detect when `--ai` or `--ai-commands-dir` incorrectly consume following flags like `--here` or `--ai-skills`
  - Now provides clear error messages: "Invalid value for --ai: '--here'"
  - Includes helpful hints suggesting proper usage and listing available agents
  - Commands like `specify init --ai-skills --ai --here` now fail with actionable feedback instead of confusing "Must specify project name" errors
  - Added comprehensive test suite (5 new tests) to prevent regressions

## [0.1.5] - 2026-02-21

### Fixed

- **AI Skills Installation Bug (#1658)**: Fixed `--ai-skills` flag not generating skill files for GitHub Copilot and other agents with non-standard command directory structures
  - Added `commands_subdir` field to `AGENT_CONFIG` to explicitly specify the subdirectory name for each agent
  - Affected agents now work correctly: copilot (`.github/agents/`), opencode (`.opencode/command/`), windsurf (`.windsurf/workflows/`), codex (`.codex/prompts/`), kilocode (`.kilocode/workflows/`), q (`.amazonq/prompts/`), and agy (`.agent/workflows/`)
  - The `install_ai_skills()` function now uses the correct path for all agents instead of assuming `commands/` for everyone

## [0.1.4] - 2026-02-20

### Fixed

- **Qoder CLI detection**: Renamed `AGENT_CONFIG` key from `"qoder"` to `"qodercli"` to match the actual executable name, fixing `specify check` and `specify init --ai` detection failures

## [0.1.3] - 2026-02-20

### Added

- **Generic Agent Support**: Added `--ai generic` option for unsupported AI agents ("bring your own agent")
  - Requires `--ai-commands-dir <path>` to specify where the agent reads commands from
  - Generates Markdown commands with `$ARGUMENTS` format (compatible with most agents)
  - Example: `specify init my-project --ai generic --ai-commands-dir .myagent/commands/`
  - Enables users to start with Spec Kit immediately while their agent awaits formal support

## [0.0.102] - 2026-02-20

- fix: include 'src/**' path in release workflow triggers (#1646)

## [0.0.101] - 2026-02-19

- chore(deps): bump github/codeql-action from 3 to 4 (#1635)

## [0.0.100] - 2026-02-19

- Add pytest and Python linting (ruff) to CI (#1637)
- feat: add pull request template for better contribution guidelines (#1634)

## [0.0.99] - 2026-02-19

- Feat/ai skills (#1632)

## [0.0.98] - 2026-02-19

- chore(deps): bump actions/stale from 9 to 10 (#1623)
- feat: add dependabot configuration for pip and GitHub Actions updates (#1622)

## [0.0.97] - 2026-02-18

- Remove Maintainers section from README.md (#1618)

## [0.0.96] - 2026-02-17

- fix: typo in plan-template.md (#1446)

## [0.0.95] - 2026-02-12

- Feat: add a new agent: Google Anti Gravity (#1220)

## [0.0.94] - 2026-02-11

- Add stale workflow for 180-day inactive issues and PRs (#1594)
