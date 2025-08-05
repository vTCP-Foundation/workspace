# Project Policy

This policy provides a single, authoritative, machine-readable source of truth for AI coding agents and humans, ensuring that all work is governed by clear, unambiguous rules and workflows.

It aims to eliminate ambiguity, reduce supervision needs, and facilitate automation while maintaining accountability and compliance with best practices.


# Introduction
> **Rationale:** Sets the context, actors, and compliance requirements for the policy, ensuring all participants understand their roles and responsibilities.

## Actors
> **Rationale:** Defines who is involved in the process and what their roles are.

- **User**: The individual responsible for defining requirements, prioritising work, approving changes, and ultimately accountable for all code modifications.

- **Agent**: The delegate responsible for executing the user's instructions precisely as defined by PRDs and tasks.


# Fundamental Principles
> **Rationale:** Lists the essential guiding principles for all work, such as task-driven development, user authority, and prohibition of unapproved changes.


## Authority and Responsibility

- **User Authority and Responsibility**: The User is the sole decider for the scope and design of ALL work. Responsibility for all code changes remains with the User, regardless of whether the AI Agent performed the implementation.

- **Data Sense-Checking**: All data must be sense-checked for consistency and accuracy.


## Workflow
> **Rationale:** Defines stages of work that are expected to be followed by the Actors, with clear step-by-step guidance for Agents

### Feature Development Workflow

#### Phase 1: PRD Planning and Preparation
1. **PRD Creation**: Create PRD using [Template PRD](/template_prd.md)
2. **PRD Validation**: Validate PRD with [PRD validation prompt](/prompts/prd_validation_prompt.md)
3. **PRD Decomposition**: Break down PRD into tasks using [Template Task](/template_task.md)
4. **Task Complexity Assessment**: Determine validation criteria level for each task (Simple/Moderate/Complex) as per "Task Validation Criteria Proportionality"
5. **User Approval**: Get explicit User approval for the complete task list and validation criteria before proceeding

#### Phase 2: Individual Task Implementation (Repeat for Each Task)

**Step 1: Pre-Implementation Setup**
- **Task Selection**: Agent identifies next task from approved task list
- **Scope Verification**: Confirm task scope aligns with PRD and no scope creep
- **Branch Creation**: Create or use PRD-dedicated feature branch named `prd/<PRD-ID>-<short-description>`. Before creation check if the branch already exists. If it does, then use that branch
- **Implementation Plan Review**: Review and confirm implementation approach with User if needed

**Step 2: Core Implementation**
- **Code Implementation**: Execute task requirements strictly within defined scope
- **Documentation Updates**: Update any required technical documentation inline with changes
- **Implementation Documentation**: Document any deviations from planned approach in task file

**Step 3: Demo Creation and Validation**
> **Critical Gate**: No code commits or test implementation shall proceed until demo is validated

- **Demo Creation**: Agent creates a demonstration that proves the implemented functionality works as expected and covers ALL task requirements
  - **Demo Purpose**: The demo serves as immediate, tangible proof that implementation meets requirements before investing time in formal testing
  - **Demo Types**: Choose appropriate format based on task complexity:
    - Simple tasks: Basic script or command demonstration
    - Moderate tasks: Interactive demonstration or test harness
    - Complex tasks: Comprehensive walkthrough covering all integration points
  - **Demo Scope**: Must demonstrate ALL requirements and DOD criteria listed in task
  - **Demo Documentation**: Include clear instructions for User to reproduce demo results

- **User Demo Validation**: User reviews and approves demo
  - **Validation Criteria**: Demo must prove all task requirements are met
  - **Failure Protocol**: If demo fails, Agent must fix implementation before proceeding
  - **Scope Validation**: Confirm no scope creep occurred during implementation

**Step 4: Test Implementation**
- **Test Development**: Implement automated tests according to task's test plan
- **Test Scope**: Tests must align with task complexity level and validation criteria
- **Test Coverage**: Ensure tests cover all functionality demonstrated in the demo
- **Test Documentation**: Update test documentation and ensure tests are properly integrated

**Step 5: Final Validation and Integration**
- **User Test Validation**: User reviews and approves implemented tests
- **Validation Criteria Check**: Confirm all proportional validation criteria are met
- **ADR Synchronization**: Update Architecture Decision Records if implementation required architectural decisions
- **Final Documentation**: Complete any remaining documentation updates

**Step 6: Version Control Integration**
- **Commit Creation**: Create commits with proper format `[<PRD-ID>-<TASK-ID>] <description>`
- **Pull Request Decision**: 
  - If current task is the last task in the PRD: Create PR with PRD linkage and implementation summary for all tasks
  - If current task is NOT the last task in the PRD: Only create commit (no PR)
- **User PR Review**: User approves pull request (only if PR was created)
- **Merge to Main**: Complete merge using squash merge strategy (only if PR was created)

### Testing-Focused Development Workflow

#### For Dedicated Testing Tasks (Unit/Integration/E2E Test Development)

**Modified Phase 2 for Testing Tasks:**

**Step 1-2**: Same as Feature Development (Pre-Implementation Setup and Core Implementation)

**Step 3: Test Demonstration**
- **Test Execution Demo**: Run implemented tests to demonstrate they work correctly
- **Test Coverage Demo**: Show what functionality/scenarios the tests cover
- **Test Results Demo**: Demonstrate that tests can detect both success and failure scenarios
- **Integration Demo**: Show how tests integrate with existing test infrastructure

**Step 4**: Skip separate test implementation (tests ARE the implementation)

**Step 5-6**: Same as Feature Development (Final Validation and Integration)

#### Demo Importance and Timing Rationale

> **Why Demos are Critical Before Commits/Tests:**

1. **Early Validation**: Demos provide immediate feedback on whether implementation meets requirements before time is invested in formal testing
2. **Scope Control**: Demos help identify scope creep early in the process when it's easier to correct
3. **User Confidence**: Users can see tangible progress and provide course corrections before work is "locked in" through commits
4. **Quality Gate**: Demos ensure functionality actually works in practice, not just in theory
5. **Documentation Verification**: Demos prove that implementation matches documented requirements
6. **Risk Mitigation**: Catching implementation issues at demo stage prevents cascading problems in tests and downstream work

#### Workflow Error Handling

**Scope Creep Detection:**
- Agent must immediately stop work and notify User if scope exceeds task boundaries
- All scope conflicts resolved through explicit User decision and task document updates

**Blocker Management:**
- Agent must identify and report blockers immediately
- Work stops until blockers are resolved through User intervention

**Validation Failures:**
- Demo validation failure: Return to Step 2 (Core Implementation)
- Test validation failure: Return to Step 4 (Test Implementation)
- Maximum 3 iteration cycles before escalating to User for guidance


## Task-Driven Development
- **Task-Driven Development**: No code shall be created or changed unless there is a task explicitly authorizing that change.

- **Task Granularity**: Tasks must be defined to be as small as practicable while still representing a cohesive, testable unit of work. Large or complex features should be broken down into multiple smaller tasks.

- **PRD Alignment**: Each task must be explicitly defined as a part of PRD (Product Requirements Document)

## Changes Approving Policy
- **Prohibition of Unapproved Changes**: Any changes outside the explicit scope of an agreed task are EXPRESSLY PROHIBITED: could be proposed by the Agent during the task implementation as options to go with, but should never lead to the implementation of changes outside the scope of the task.

- **Controlled File Creation**: The Agent shall not create any files, including standalone documentation files, that are outside the explicitly defined structures for PRDs (see "Default Project Layout"), tasks (see "Task Documentation and Process"), or source code, unless the User has given explicit prior confirmation for the creation of each specific file. This principle is to prevent the generation of unrequested or unmanaged documents.

## Documentation
- **Technical Documentation for APIs and Interfaces**: As part of completing any PRD that creates or modifies APIs, services, interfaces, or protocols, technical documentation must be created or updated explaining how to use these components. This documentation should include (but not limited to):

    - API usage examples and patterns
    - Interface contracts and expected behaviors  
    - Integration guidelines for other developers
    - Configuration options and defaults
    - Error handling and troubleshooting guidance
    - Edge cases and limitations 
    - The documentation must be created in the appropriate location (e.g., `docs/technical/` or inline code documentation) and linked from the PRD detail document.

## Best Practices
- **Don't Repeat Yourself (DRY)**: Information and code must be defined in a single location and referenced elsewhere to avoid duplication and reduce the risk of inconsistencies. Specifically:
    - Task information should be fully detailed in their dedicated task files and only referenced from other documents.
    - PRD documents should reference the task list rather than duplicating task details.
    - Any documentation that needs to exist in multiple places should be maintained in a single source of truth and referenced elsewhere.

- **Use of Constants for Repeated or Special Values**: Any value (number, string, etc.) used more than once in generated code—especially "magic numbers" or values with special significance—**must** be defined as a named constant.
    - Example: Instead of `for (let i = 0; i < 10; i++) { ... }`, define `const numWebsites = 10;` and use `for (let i = 0; i < numWebsites; i++) { ... }`.
    - All subsequent uses must reference the constant, not the literal value.
    - This applies to all code generation, automation, and manual implementation within the project.

## Scope Limitations

> **Rationale:** Prevents unnecessary work and keeps all efforts focused on agreed tasks, avoiding gold plating and scope creep.

- No gold plating or scope creep is allowed.
- All work must be scoped to the specific task at hand.
- Any identified improvements or optimizations must be proposed as separate tasks.


## Change Management Rules

> **Rationale:** Defines how changes are managed, requiring explicit association with tasks and strict adherence to scope.

- Conversation about any code change must start by ascertaining the linked Task or PRD before proceeding.
- All changes must be associated with a specific task.
- No changes should be made outside the scope of the current task.
- Any scope creep must be identified, rolled back, and addressed in a new task.
- If the User asks to make a change without referring to a task, then the Agent MUST NOT do the work and must have a conversation about it to determine if it should be associated with an existing task or if a new PRD + task should be created.


# Default Project Layout

> **Rationale:** Provides a consistent and predictable structure for the project, ensuring that all documentation and code is organized in a way that is easy to understand and maintain.

| Directory | Description |
|-----------|-------------|
| `architecture/<project-name>/adrs/` | Architecture Decision Records for a specific project. |
| `architecture/<project-name>/protocols/` | Protocols for a specific project. |
| `docs/` | General, project-agnostic documentation. |
| `workflow/prd/<project-name>/` | Product Requirements Documents for a specific project. |
| `workflow/tasks/<project-name>/` | Task definitions for a specific project. |
| `test/` | Test files for the project (Unit, Integration, E2E). |


# PRD Documentation and Structure

> **Rationale:** Defines the structure and content required for Product Requirements Documents, ensuring consistent and comprehensive requirement specification.

- **Location Pattern**: `workflow/prd/<project-name>/`
- **File Naming**: `<PRD-ID>-<SHORT-NAME>.md` (e.g., `1-user-management.md`)

| Section | Description |
|---------|-------------|
| `# [PRD-ID] [PRD-Name]` | Title of the PRD including its ID and name |
| `# Overview` | High-level description of the feature or capability |
| `# Business Justification` | Why this PRD is needed and its business value |
| `# Conditions of Satisfaction (CoS)` | Clear, testable criteria that define when the PRD is complete |
| `# Scope and Boundaries` | What is included and explicitly excluded from this PRD |
| `# Dependencies` | Other PRDs, systems, or external factors this PRD depends on |
| `# Assumptions and Constraints` | Key assumptions made and constraints that must be respected |
| `# Task List` | Reference to the task list file for this PRD |

## PRD Validation Rules

- **Core Rules**:
   - PRD IDs must be unique across the entire project
   - All Conditions of Satisfaction must be measurable and testable
   - Dependencies must reference existing PRDs or be clearly marked as external

- **Task Association**:
   - Each PRD must have an associated task list file at `workflow/tasks/<PRD-ID>/tasks.md`
   - All tasks within a PRD must contribute to achieving the PRD's Conditions of Satisfaction


# Task Documentation and Process

> **Rationale:** Specifies the structure and content required for task documentation, supporting transparency and reproducibility.

- **Location Pattern**: `workflow/tasks/<project-name>/<PRD-ID>/`
- **File Naming**: `<PRD-ID>-<TASK-ID>-<SHORT-DESCRIPTION>.md` (e.g., `1-1-create-user-table.md` for first task of PRD 1)

| Section | Description |
|---------|-------------|
| `# [Task-ID] [Task-Name]` | Title of the task including its ID and name |
| `# Links` | Links to the PRD, and other relevant documents |
| `# Description` | A short description of the task (mainly for the User to keep track of the tasks) |
| `# Requirements and DOD` | Complete and unambiguous list of requirements and DOD (Definition of Done) |
| `# Implementation Plan` | A detailed plan for the task, including the steps to be taken and the expected outcomes |
| `# Test Plan` | Testing requirements for the task |
| `# Verification and Validation` | User approval that validation criteria appropriate to task complexity have been met |
| `## Architecture integrity` | Verification of architectural compliance |
| `## Security` | Security validation |
| `## Performance` | Performance validation |
| `## Scalability` | Scalability validation |
| `## Reliability` | Reliability validation |
| `## Maintainability` | Maintainability validation |
| `## Cost` | Cost validation |
| `## Compliance` | Compliance validation |

## Task Validation Rules

> **Rationale:** Ensures all tasks adhere to required standards and workflows.

- **Core Rules**:
   - All tasks must be associated with an existing PRD
   - Task IDs must be unique within their parent PRD
   - All required documentation must be completed before marking a task as 'Done'

- **Pre-Implementation Checks**:
   - Document the task ID in all related changes.
   - List all files that will be modified.
   - Get explicit approval before proceeding with implementation.
   - Verify that validation criteria has been filled in the task file.

- **Error Prevention**:
   - If unable to access required files / resources, stop and report the issue

- **Change Management**:
   - Reference the task ID in all commit messages.
   - Ensure all changes are properly linked to the task.
   - Document any deviations from the planned implementation.

## Task Validation Criteria Proportionality

> **Rationale:** Validation criteria must be proportional to task complexity and risk, avoiding over-engineering for simple tasks while ensuring thorough validation for complex ones.

### Simple Tasks (Basic functions, configuration changes, documentation updates)
- **Functional Validation**: Core functionality works as specified
- **Integration Validation**: Changes integrate properly with existing system
- **Documentation Validation**: Any required documentation is complete and accurate.

### Moderate Tasks (API endpoints, database changes, service integrations)
- **All Simple Task criteria, plus:**
- **Security Validation**: No security vulnerabilities introduced
- **Performance Validation**: No significant performance degradation
- **Error Handling Validation**: Proper error handling and logging implemented

### Complex Tasks (Multi-service features, architectural changes, critical system components)
- **All Moderate Task criteria, plus:**
- **Architecture Integrity**: Changes align with system architecture and design principles
- **Scalability Validation**: Solution can handle expected load and growth
- **Reliability Validation**: Appropriate fault tolerance and recovery mechanisms
- **Maintainability Validation**: Code is maintainable and follows project standards
- **Cost Validation**: Resource usage is within acceptable limits
- **Compliance Validation**: All regulatory and policy requirements are met

### Validation Assignment Rules
- Task complexity level must be determined during task creation
- Validation criteria must be explicitly listed in the task document
- User must approve the validation level before task implementation begins
- Additional validation criteria may be added if risks are identified during implementation


## Blocker Management

- **Identification**: Blockers must be identified as soon as they are encountered


# Conflict Resolution and Escalation

> **Rationale:** Provides clear procedures for resolving disagreements and handling edge cases that don't fit standard workflows.

## Scope Interpretation Conflicts

- **Agent Uncertainty**: If the Agent is uncertain about task scope or requirements, work must stop and clarification must be requested from the User
- **Scope Creep Detection**: If the Agent identifies potential scope creep during implementation, the User must be notified immediately with specific details
- **Resolution Process**: All scope conflicts must be resolved through explicit User decision and task document updates

## Emergency Procedures

- **Production Issues**: Critical production issues may bypass normal task creation process but must be documented within 24 hours
- **Security Vulnerabilities**: Security issues take precedence over normal workflow but require immediate User notification
- **Post-Emergency Documentation**: All emergency changes must be retroactively documented with proper task creation and validation

## Escalation Triggers

- Conflicts between task requirements and technical constraints
- Resource access issues preventing task completion
- Disagreements on task interpretation or scope


# Version Control and Change Management

> **Rationale:** Ensures all code changes are properly tracked, reviewed, and integrated while maintaining project history and accountability.

## Branching Strategy

- **Main Branch**: `main` branch contains production-ready code
- **PRD Feature Branches**: Each PRD must be implemented in a dedicated feature branch named `prd/<PRD-ID>-<short-description>`
- **Task Implementation**: Each task within a PRD is implemented as a commit in the PRD's feature branch
- **Branch Lifecycle**: PRD feature branches are created from `main` and merged back via pull request after all PRD tasks are completed

## Commit Standards

- **Commit Messages**: Must follow format: `[<PRD-ID>-<TASK-ID>] <description>`
- **Atomic Commits**: Each commit should represent a single logical change
- **Commit Frequency**: Commit early and often to maintain clear development history

## Pull Request Requirements

- **PR Title**: Must include PRD ID and clear description of all changes
- **PR Description**: Must link to PRD document and summarize implementation approach for all tasks
- **Review Process**: All PRs require User approval before merging
- **Merge Strategy**: Use squash merge to maintain clean history on main branch

## Change Tracking

- **Task Linkage**: All commits must be linked to their originating task
- **PRD Linkage**: All PRs must be linked to their originating PRD
- **Documentation Updates**: Any changes affecting documentation must update relevant docs in the same PR
- **Rollback Procedures**: All changes must be reversible through standard git operations


# Testing Strategy and Documentation

> **Rationale:** Ensures that testing is approached thoughtfully, appropriately scoped, and well-documented, leading to higher quality software and maintainable test suites.

## General Principles for Testing

1.  **Risk-Based Approach**: Prioritize testing efforts based on the complexity and risk associated with the features being developed.
2.  **Test Pyramid Adherence**: Strive for a healthy balance of unit, integration, and end-to-end tests. While unit tests form the base, integration tests are crucial for verifying interactions between components.
3.  **Clarity and Maintainability**: Tests should be clear, concise, and super easy to understand and maintain. Avoid overly complex test implementations.
4.  **Automation**: Automate tests wherever feasible to ensure consistent and repeatable verification.

## Test Scoping Guidelines

1.  **Unit Tests**:
    *   **Focus**: Test individual functions, methods, or classes in isolation from the code base. Specifically do not test package API methods directly they should be covered by the package tests themselves.
    *   **Mocking**: Mock all external dependencies of the unit under test.
    *   **Scope**: Verify the logic within the unit, including edge cases and error handling specific to that unit.

2.  **Integration Tests**:
    *   **Focus**: Verify the interaction and correct functioning of multiple components or services working together as a subsystem.
    *   **Example**: Testing a feature that involves an API endpoint, a service layer, database interaction, and a job queue.
    *   **Mocking Strategy**:
        *   Mock external, third-party services at the boundary of your application (e.g., external APIs like Firecrawl, Gemini).
        *   For internal infrastructure components (e.g., database, message queues like pg-boss), prefer using real instances configured for a test environment rather than deep mocking of their client libraries. This ensures tests are validating against actual behavior.
    *   **Starting Point for Complex Features**: For features involving significant component interaction, consider starting with integration tests to ensure the overall flow and orchestration are correct before (or in parallel with) writing more granular unit tests for individual components.

3.  **End-to-End (E2E) Tests**:
    *   **Focus**: Test the entire application flow from the user's perspective.
    *   **Scope**: Reserved for critical user paths and workflows.

## Test Plan Documentation and Strategy

1.  **PRD-Level Testing Strategy**:
    *   The Conditions of Satisfaction (CoS) listed in a PRD detail document (`workflow/prd/<PRD-ID-<SHORT NAME>.md`) inherently define the high-level success criteria and scope of testing for the PRD. Detailed, exhaustive test plans are generally not duplicated at this PRD level if covered by tasks.
    *   The task list for a PRD (e.g., `workflow/tasks/<PRD-ID>/tasks.md`) **must** include a dedicated task, typically named "E2E CoS Test" or similar (e.g., `<PRD-ID>-E2E-CoS-Test.md`).
    *   This "E2E CoS Test" task is responsible for holistic end-to-end testing that verifies the PRD's overall CoS are met. This task's document will contain the detailed E2E test plan, encompassing functionality that may span multiple individual implementation tasks within that PRD.

2.  **Task-Level Test Plan Proportionality**:
    *   **Pragmatic Principle**: Test plans must be proportional to the complexity and risk of the task. Avoid over-engineering test plans for simple tasks.
    *   **Basic Implementation Tasks (happy flow)** (simple functions, basic integrations):
        *   Test plan should verify core functionality and error handling patterns
        *   Example: "Function can be registered with system, basic workflow works, follows existing error patterns"
    *   **Complex Implementation Tasks (edge cases, error scenarios)** (multi-service integration, complex business logic):
        *   More detailed test plans with specific scenarios and edge cases
        *   Should still avoid excessive elaboration unless truly warranted by risk

3.  **Test Plan Documentation Requirements**:
    *   **Requirement**: Every individual task that involves code implementation or modification **must** include a test plan, but the detail level should match the task complexity.
    *   **Location**: The test plan must be documented within a "# Test Plan" section of the task's detail document.
    *   **Content for Simple Tasks**:
        *   Success criteria focused on basic functionality and compilation
        *   No elaborate scenarios unless complexity warrants it
    *   **Content for Complex Tasks**:
        *   **Objective(s)**: What the tests for this specific task aim to verify
        *   **Test Scope**: Specific components, functions, or interactions covered
        *   **Environment & Setup**: Relevant test environment details or setup steps
        *   **Mocking Strategy**: Definition of mocks used for the task's tests
        *   **Key Test Scenarios**: Scenarios covering successful paths, expected failure paths, and edge cases *relevant to the task's scope*
        *   **Success Criteria**: How test success/failure is determined for the task
    *   **Review**: Task-level test plans should be reviewed for appropriateness to task complexity
    *   **Living Document**: Test plans should be updated if task requirements change significantly
    *   **Task Completion Prerequisite**: A task cannot be marked as "Done" unless the tests defined in its test plan are passing

4.  **Test Distribution Strategy**:
    *   **Avoid Test Plan Duplication**: Detailed edge case testing and complex scenarios should be concentrated in dedicated E2E testing tasks rather than repeated across individual implementation tasks
    *   **Focus Individual Tasks**: Individual implementation tasks should focus on verifying their specific functionality works as intended within the broader system
    *   **Comprehensive Testing**: Complex integration testing, error scenarios, and full workflow validation belongs in dedicated testing tasks (typically E2E CoS tests)
