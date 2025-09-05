## PRD Tasks Creation Prompt (Template)

> Copy this prompt, replace the placeholders, and run it to generate task files for an existing PRD.

### Inputs (fill before running)
- PRD: <<<PRD_PATH_AND_NAME>>>  (absolute path and name of the PRD file)
- Tasks output directory: <<<TASKS_DIR>>>  (absolute path to the folder for created task files)
- Tasks to create (user-defined list):
<<<TASKS_LIST>>>

### Role & Objective
You are the Agent working in this repository. Your objective is to create a complete set of task definition files for the provided PRD, ensuring each task adheres strictly to the project policy and the task template.

### Mandatory Rules
- Use `@template_task.md` as the canonical template for each task file.
- Follow all constraints and workflow rules from `@policy.md`.
- Transfer all necessary context from the PRD into each specific task. Do not rely on out-of-band knowledge; each task must be self-sufficient.
- Do not create code or tests. Only create task files.

### What to Produce
For every item in <<<TASKS_LIST>>>:
1) Create a task file under <<<TASKS_DIR>>> using the filename pattern:
   - `<PRD-ID>-<TASK-ID>-<short-name>.md`
   - If `<PRD-ID>` is not explicitly given, infer it from <<<PRD_PATH_AND_NAME>>> (filename prefix) or PRD header.
   - `<TASK-ID>` is sequential within the PRD starting from 1.
2) Structure and content MUST follow `@template_task.md` exactly (all sections present and populated).
3) Context migration from PRD:
   - "# Description": Summarize the task’s purpose and tie it to the PRD’s Overview/Scope.
   - "# Requirements and DOD": Derive clear, testable requirements from PRD Conditions of Satisfaction relevant to this task.
   - "# Implementation Plan": Provide concrete steps limited to this task’s scope; reference relevant repo paths and components as needed.
   - "# Test Plan": Provide proportional plan per `Task Validation Criteria Proportionality` in `@policy.md` (Simple/Moderate/Complex). No tests are implemented here.
   - "# Verification and Validation" subsections: Specify validation appropriate to the declared complexity.
   - Link back to the PRD and list any dependencies explicitly.
4) Ensure task scope is focused (no scope creep) and aligned with the PRD.
5) Ensure naming consistency and clarity. Keep each task concise but self-contained.

### Constraints & Compliance
- Adhere to:
  - Task-driven development and scope rules (see `@policy.md`).
  - File naming and location conventions. Use exactly <<<TASKS_DIR>>> as the output directory.
  - Proportional validation criteria per complexity level.
- Do not introduce new files outside the tasks directory without explicit instruction.

### Output Requirements
- Save each created task file to <<<TASKS_DIR>>>.
- After saving all tasks, return a concise summary including:
  - A checklist of created tasks with file paths
  - Any assumptions made (if unavoidable)
  - Any blockers that require Architect input

### Quality Bar
- Every task file must be self-sufficient: a reviewer should not need to open the PRD to understand purpose, scope, and validation for that task.
- All requirements must be measurable and testable.
- Keep language clear and action-oriented.

### Example placeholder for <<<TASKS_LIST>>>
- [1] Short name: <short-name-1>
  - Description: <one-line goal>
  - Complexity: <Simple|Moderate|Complex>
  - Notes: <optional constraints or dependencies>
- [2] Short name: <short-name-2>
  - Description: <one-line goal>
  - Complexity: <Simple|Moderate|Complex>
  - Notes: <optional constraints or dependencies>

(Replace all placeholders before running.)
