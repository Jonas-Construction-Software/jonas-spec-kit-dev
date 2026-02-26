---
description: >-
  Create or update the feature specification. Optionally import from Jira user story 
  or provide a natural language feature description. Supports Jira dev-task-based branch naming.
handoffs: 
  - label: Build Technical Plan
    agent: speckit.plan
    prompt: Create a plan for the spec. I am building with...
  - label: Clarify Spec Requirements
    agent: speckit.clarify
    prompt: Clarify specification requirements
    send: true
scripts:
  sh: scripts/bash/create-new-feature.sh --json "{ARGS}"
  ps: scripts/powershell/create-new-feature.ps1 -Json "{ARGS}"
---

## Pre-processing: Optional Jira User Story Import (Step 0)

Before treating the user's message after `/speckit.specify` as the feature description, you **MUST** ask:

> **Do you want to import an existing Jira user story for this feature? (Yes/No)**

### If the user answers **Yes**

1. Ask the user for the Jira user story key:  
   **"Please provide the Jira user story number/key (e.g., PROJ-123)."**

2. **Retrieve Atlassian Cloud ID**:
   
   Call `mcp_com_atlassian_getAccessibleAtlassianResources()` to get the cloud ID.
   
   **Error handling**:
   - If the call fails or returns no resources:
     ```
     ❌ Error: Cannot connect to Atlassian services. 
     
     Possible causes:
     - Atlassian MCP server not installed or not running
     - No Atlassian account connected in VS Code
     - Network connectivity issues
     
     To resolve:
     1. Install the Atlassian MCP server extension in VS Code
     2. Authenticate with your Atlassian account
     3. Try again
     
     Would you like to continue without importing from Jira? (Yes/No)
     ```
   - If user says **Yes**: Skip to Step 0 "If the user answers **No**" section
   - If user says **No**: Exit with error message

3. **Fetch the Jira issue**:
   
   Use `mcp_com_atlassian_getJiraIssue()` with:
   - `cloudId`: from step 2
   - `issueIdOrKey`: provided by user
   - `fields`: `["summary", "description", "issuetype", "status", "subtasks", "customfield_*"]`
   
   **Error handling**:
   - If issue not found (404):
     ```
     ❌ Error: Issue {ISSUE_KEY} not found.
     
     Please verify:
     - The issue key is correct (e.g., PROJ-123)
     - You have permission to view this issue
     - The issue exists in your Jira instance
     
     Would you like to:
     A) Try a different issue key
     B) Continue without importing from Jira
     ```
   - If permission denied (403):
     ```
     ❌ Error: Access denied to issue {ISSUE_KEY}.
     
     You don't have permission to view this issue. Please:
     - Request access from your Jira administrator
     - Verify you're authenticated with the correct account
     
     Would you like to:
     A) Try a different issue key
     B) Continue without importing from Jira
     ```
   - If network error or timeout:
     ```
     ❌ Error: Cannot connect to Jira.
     
     Network error: {ERROR_MESSAGE}
     
     Please check your internet connection and try again.
     
     Would you like to:
     A) Retry fetching the issue
     B) Continue without importing from Jira
     ```
   - If any other error:
     ```
     ❌ Error: Failed to fetch Jira issue.
     
     Error details: {ERROR_MESSAGE}
     
     Would you like to:
     A) Try a different issue key
     B) Continue without importing from Jira
     ```
   
   Handle user's choice accordingly (retry, try different key, or skip import).

4. **Present the retrieved content**:
   
   ```markdown
   ## Jira Story: {ISSUE_KEY} - {SUMMARY}
   
   **Type**: {ISSUETYPE}
   **Status**: {STATUS}
   
   ### Description
   {DESCRIPTION}
   
   ### Acceptance Criteria
   {ACCEPTANCE_CRITERIA or "Not specified"}
   
   ### Sub-tasks
   {LIST_OF_SUBTASKS or "None"}
   
   ---
   
   Would you like to:
   A) Use this content as-is for the feature specification
   B) Use this content and add additional requirements
   C) Reject this content and continue without importing
   
   **Your choice**:
   ```

5. **Handle the user's response**:
   
   - **Choice A**: Use Jira content as the feature description
   - **Choice B**: Ask user "Please provide your additional requirements:" and combine with Jira content
   - **Choice C**: Skip Jira import and proceed to "If the user answers **No**" section

6. **Set feature description variable**:
   
   Based on the choice, set `FEATURE_DESCRIPTION` to the appropriate content for use in subsequent steps.

### If the user answers **No**

1. Ask the user to provide the feature description:
   
   **"Please provide a detailed description of the feature you want to build:"**

2. Wait for user's response

3. Set `FEATURE_DESCRIPTION` to the user's provided description

---

## User Input

```text
$ARGUMENTS
```

> **Note**: The `$ARGUMENTS` field is optional. After Step 0, the "feature description" is either:  
> (a) Jira story content (optionally augmented with additional requirements), or  
> (b) The user's natural-language description provided when prompted in the "No" path
> 
> If `$ARGUMENTS` is provided with the initial command, it can be used as a default/fallback, but Step 0 takes precedence.

---

## Outline

The finalized feature description (from Step 0) is now available. You have either:
- Imported it from Jira (with optional additional requirements), or
- Received it directly from the user when they chose not to import from Jira

Do not ask the user to repeat the feature description - you already have it from Step 0.

Given that finalized feature description (from Step 0), do this:

1. **Generate a concise short name** (2-4 words) for the branch:
   - Analyze the `FEATURE_DESCRIPTION` (set in Step 0) and extract the most meaningful keywords
   - Create a 2-4 word short name that captures the essence of the feature
   - Use action-noun format when possible (e.g., "add-user-auth", "fix-payment-bug")
   - Preserve technical terms and acronyms (OAuth2, API, JWT, etc.)
   - Keep it concise but descriptive enough to understand the feature at a glance
   - Examples:
     - "I want to add user authentication" → "user-auth"
     - "Implement OAuth2 integration for the API" → "oauth2-api-integration"
     - "Create a dashboard for analytics" → "analytics-dashboard"
     - "Fix payment processing timeout bug" → "fix-payment-timeout"

2. **Check for existing branches before creating new one**:

   a. First, fetch all remote branches to ensure we have the latest information:

      ```bash
      git fetch --all --prune
      ```

   b. Find the highest feature number across all sources for the short-name:
      - Remote branches: `git ls-remote --heads origin | grep -E 'refs/heads/[0-9]+-<short-name>$'`
      - Local branches: `git branch | grep -E '^[* ]*[0-9]+-<short-name>$'`
      - Specs directories: Check for directories matching `specs/[0-9]+-<short-name>`

   c. Determine the next available number:
      - Extract all numbers from all three sources
      - Find the highest number N
      - Use N+1 for the new branch number

   d. Run the script `{SCRIPT}` with the calculated number and short-name:
      - **Generate a brief summary** from `FEATURE_DESCRIPTION`:
        * Create a concise summary (maximum 35 characters)
        * Capture the key essence of the feature in a few words
        * Examples:
          - "As a user, I want to authenticate using OAuth2 so that I can securely access my account with social login" → "Add OAuth2 authentication"
          - Long Jira story with multiple paragraphs → "User dashboard with analytics"
      - Pass `--number N+1`, `--short-name "your-short-name"`, and the **brief summary** (not the full FEATURE_DESCRIPTION)
      - Bash example: `{SCRIPT} --json --number 5 --short-name "user-auth" "Add OAuth2 authentication"`
      - PowerShell example: `{SCRIPT} -Json -Number 5 -ShortName "user-auth" "Add OAuth2 authentication"`
      - **IMPORTANT**: Keep the summary under 35 characters to avoid command-line length issues

   **IMPORTANT**:
   - Check all three sources (remote branches, local branches, specs directories) to find the highest number
   - Only match branches/directories with the exact short-name pattern
   - If no existing branches/directories found with this short-name, start with number 1
   - You must only ever run this script once per feature
   - The JSON is provided in the terminal as output - always refer to it to get the actual content you're looking for
   - The JSON output will contain BRANCH_NAME and SPEC_FILE paths
   - For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot")

   e. **⏸ HOLD POINT - Ask for Jira Development Task Number (Optional)**:
   
      Immediately **after** the script completes successfully (branch and directory created), **PAUSE** and ask:
      
      > **Do you have a Jira development task number to use as the branch prefix (e.g., FT-53, DEV-142)? (Yes/No)**
      
      - **If Yes**:
        1. Ask: **"Please provide the Jira development task key (e.g., FT-53):"**
        2. Validate format: Must match pattern `[A-Z]+-\d+` (e.g., FT-53, DEV-142, PROJ-789)
        3. If format invalid, ask again with example
        4. Parse script JSON output to get current branch and directory names
        5. Extract the short-name portion from the current branch (everything after the number prefix)
        6. **Rename the Git branch**:
           
           For Bash:
           ```bash
           OLD_BRANCH="{parsed BRANCH_NAME from JSON}"
           NEW_BRANCH="{DEV_TASK_KEY}-{short-name}"  # e.g., FT-53-user-auth
           
           # Rename local branch
           git branch -m "$OLD_BRANCH" "$NEW_BRANCH"
           
           # If remote tracking exists, update it
           if git rev-parse --abbrev-ref "$NEW_BRANCH@{upstream}" >/dev/null 2>&1; then
               git push origin -u "$NEW_BRANCH"
               git push origin --delete "$OLD_BRANCH" 2>/dev/null || true
           fi
           ```
           
           For PowerShell:
           ```powershell
           $oldBranch = "{parsed BRANCH_NAME from JSON}"
           $newBranch = "{DEV_TASK_KEY}-{short-name}"  # e.g., FT-53-user-auth
           
           # Rename local branch
           git branch -m $oldBranch $newBranch
           
           # If remote tracking exists, update it
           try {
               git rev-parse --abbrev-ref "$newBranch@{upstream}" 2>$null | Out-Null
               if ($LASTEXITCODE -eq 0) {
                   git push origin -u $newBranch
                   git push origin --delete $oldBranch 2>$null | Out-Null
               }
           } catch { }
           ```
        
        7. **Rename the feature directory**:
           
           For Bash:
           ```bash
           OLD_DIR="{specs/XXX-short-name from JSON}"
           NEW_DIR="specs/{DEV_TASK_KEY}-{short-name}"  # e.g., specs/FT-53-user-auth
           
           # Rename directory using git mv (preserves history)
           git mv "$OLD_DIR" "$NEW_DIR"
           
           # Update path variables for subsequent steps
           FEATURE_DIR="$NEW_DIR"
           SPEC_FILE="$NEW_DIR/spec.md"
           ```
           
           For PowerShell:
           ```powershell
           $oldDir = "{specs/XXX-short-name from JSON}"
           $newDir = "specs/{DEV_TASK_KEY}-{short-name}"  # e.g., specs/FT-53-user-auth
           
           # Rename directory using git mv (preserves history)
           git mv $oldDir $newDir
           
           # Update path variables for subsequent steps
           $featureDir = $newDir
           $specFile = "$newDir/spec.md"
           ```
        
        8. Confirm to user:
           ```
           ✅ Branch and directory renamed:
              Old: {OLD_BRANCH} → New: {NEW_BRANCH}
              Directory: {NEW_DIR}
           
           Updated paths:
           - BRANCH_NAME: {NEW_BRANCH}
           - FEATURE_DIR: {NEW_DIR}
           - SPEC_FILE: {NEW_DIR}/spec.md
           ```
        
        9. **Update environment variable**:
           ```bash
           export SPECIFY_FEATURE="{NEW_BRANCH}"
           ```
           or
           ```powershell
           $env:SPECIFY_FEATURE = "{NEW_BRANCH}"
           ```
        
        10. Continue to step 3 using the updated paths
      
      - **If No**: Continue with auto-numbered branch (001-, 002-, etc.) and proceed to step 3

3. Load `templates/spec-template.md` to understand required sections.

4. Follow this execution flow:

    1. Parse the `FEATURE_DESCRIPTION` (set in Step 0)
       If empty or missing: ERROR "No feature description provided from Step 0"
    2. Extract key concepts from description
       Identify: actors, actions, data, constraints
    3. For unclear aspects:
       - Make informed guesses based on context and industry standards
       - Only mark with [NEEDS CLARIFICATION: specific question] if:
         - The choice significantly impacts feature scope or user experience
         - Multiple reasonable interpretations exist with different implications
         - No reasonable default exists
       - **LIMIT: Maximum 3 [NEEDS CLARIFICATION] markers total**
       - Prioritize clarifications by impact: scope > security/privacy > user experience > technical details
    4. Fill User Scenarios & Testing section
       If no clear user flow: ERROR "Cannot determine user scenarios"
    5. Generate Functional Requirements
       Each requirement must be testable
       Use reasonable defaults for unspecified details (document assumptions in Assumptions section)
    6. Define Success Criteria
       Create measurable, technology-agnostic outcomes
       Include both quantitative metrics (time, performance, volume) and qualitative measures (user satisfaction, task completion)
       Each criterion must be verifiable without implementation details
    7. Identify Key Entities (if data involved)
    8. Return: SUCCESS (spec ready for planning)

5. Write the specification to SPEC_FILE using the template structure, replacing placeholders with concrete details derived from the `FEATURE_DESCRIPTION` (set in Step 0) while preserving section order and headings.

6. **Specification Quality Validation**: After writing the initial spec, validate it against quality criteria:

   a. **Create Spec Quality Checklist**: Generate a checklist file at `FEATURE_DIR/checklists/requirements.md` using the checklist template structure with these validation items:

      ```markdown
      # Specification Quality Checklist: [FEATURE NAME]
      
      **Purpose**: Validate specification completeness and quality before proceeding to planning
      **Created**: [DATE]
      **Feature**: [Link to spec.md]
      
      ## Content Quality
      
      - [ ] No implementation details (languages, frameworks, APIs)
      - [ ] Focused on user value and business needs
      - [ ] Written for non-technical stakeholders
      - [ ] All mandatory sections completed
      
      ## Requirement Completeness
      
      - [ ] No [NEEDS CLARIFICATION] markers remain
      - [ ] Requirements are testable and unambiguous
      - [ ] Success criteria are measurable
      - [ ] Success criteria are technology-agnostic (no implementation details)
      - [ ] All acceptance scenarios are defined
      - [ ] Edge cases are identified
      - [ ] Scope is clearly bounded
      - [ ] Dependencies and assumptions identified
      
      ## Feature Readiness
      
      - [ ] All functional requirements have clear acceptance criteria
      - [ ] User scenarios cover primary flows
      - [ ] Feature meets measurable outcomes defined in Success Criteria
      - [ ] No implementation details leak into specification
      
      ## Notes
      
      - Items marked incomplete require spec updates before `/speckit.clarify` or `/speckit.plan`
      ```

   b. **Run Validation Check**: Review the spec against each checklist item:
      - For each item, determine if it passes or fails
      - Document specific issues found (quote relevant spec sections)

   c. **Handle Validation Results**:

      - **If all items pass**: Mark checklist complete and proceed to step 6

      - **If items fail (excluding [NEEDS CLARIFICATION])**:
        1. List the failing items and specific issues
        2. Update the spec to address each issue
        3. Re-run validation until all items pass (max 3 iterations)
        4. If still failing after 3 iterations, document remaining issues in checklist notes and warn user

      - **If [NEEDS CLARIFICATION] markers remain**:
        1. Extract all [NEEDS CLARIFICATION: ...] markers from the spec
        2. **LIMIT CHECK**: If more than 3 markers exist, keep only the 3 most critical (by scope/security/UX impact) and make informed guesses for the rest
        3. For each clarification needed (max 3), present options to user in this format:

           ```markdown
           ## Question [N]: [Topic]
           
           **Context**: [Quote relevant spec section]
           
           **What we need to know**: [Specific question from NEEDS CLARIFICATION marker]
           
           **Suggested Answers**:
           
           | Option | Answer | Implications |
           |--------|--------|--------------|
           | A      | [First suggested answer] | [What this means for the feature] |
           | B      | [Second suggested answer] | [What this means for the feature] |
           | C      | [Third suggested answer] | [What this means for the feature] |
           | Custom | Provide your own answer | [Explain how to provide custom input] |
           
           **Your choice**: _[Wait for user response]_
           ```

        4. **CRITICAL - Table Formatting**: Ensure markdown tables are properly formatted:
           - Use consistent spacing with pipes aligned
           - Each cell should have spaces around content: `| Content |` not `|Content|`
           - Header separator must have at least 3 dashes: `|--------|`
           - Test that the table renders correctly in markdown preview
        5. Number questions sequentially (Q1, Q2, Q3 - max 3 total)
        6. Present all questions together before waiting for responses
        7. Wait for user to respond with their choices for all questions (e.g., "Q1: A, Q2: Custom - [details], Q3: B")
        8. Update the spec by replacing each [NEEDS CLARIFICATION] marker with the user's selected or provided answer
        9. Re-run validation after all clarifications are resolved

   d. **Update Checklist**: After each validation iteration, update the checklist file with current pass/fail status

7. Report completion with branch name, spec file path, checklist results, and readiness for the next phase (`/speckit.clarify` or `/speckit.plan`).

**NOTE:** The script creates and checks out the new branch and initializes the spec file before writing.

## General Guidelines

## Quick Guidelines

- Focus on **WHAT** users need and **WHY**.
- Avoid HOW to implement (no tech stack, APIs, code structure).
- Written for business stakeholders, not developers.
- DO NOT create any checklists that are embedded in the spec. That will be a separate command.

### Section Requirements

- **Mandatory sections**: Must be completed for every feature
- **Optional sections**: Include only when relevant to the feature
- When a section doesn't apply, remove it entirely (don't leave as "N/A")

### For AI Generation

When creating this spec from a user prompt:

1. **Make informed guesses**: Use context, industry standards, and common patterns to fill gaps
2. **Document assumptions**: Record reasonable defaults in the Assumptions section
3. **Limit clarifications**: Maximum 3 [NEEDS CLARIFICATION] markers - use only for critical decisions that:
   - Significantly impact feature scope or user experience
   - Have multiple reasonable interpretations with different implications
   - Lack any reasonable default
4. **Prioritize clarifications**: scope > security/privacy > user experience > technical details
5. **Think like a tester**: Every vague requirement should fail the "testable and unambiguous" checklist item
6. **Common areas needing clarification** (only if no reasonable default exists):
   - Feature scope and boundaries (include/exclude specific use cases)
   - User types and permissions (if multiple conflicting interpretations possible)
   - Security/compliance requirements (when legally/financially significant)

**Examples of reasonable defaults** (don't ask about these):

- Data retention: Industry-standard practices for the domain
- Performance targets: Standard web/mobile app expectations unless specified
- Error handling: User-friendly messages with appropriate fallbacks
- Authentication method: Standard session-based or OAuth2 for web apps
- Integration patterns: Use project-appropriate patterns (REST/GraphQL for web services, function calls for libraries, CLI args for tools, etc.)

### Success Criteria Guidelines

Success criteria must be:

1. **Measurable**: Include specific metrics (time, percentage, count, rate)
2. **Technology-agnostic**: No mention of frameworks, languages, databases, or tools
3. **User-focused**: Describe outcomes from user/business perspective, not system internals
4. **Verifiable**: Can be tested/validated without knowing implementation details

**Good examples**:

- "Users can complete checkout in under 3 minutes"
- "System supports 10,000 concurrent users"
- "95% of searches return results in under 1 second"
- "Task completion rate improves by 40%"

**Bad examples** (implementation-focused):

- "API response time is under 200ms" (too technical, use "Users see results instantly")
- "Database can handle 1000 TPS" (implementation detail, use user-facing metric)
- "React components render efficiently" (framework-specific)
- "Redis cache hit rate above 80%" (technology-specific)
