# Implementation Worker Prompt Template

Use this template when dispatching the task implementation worker.

"Implementer" is a **role**, not a special agent type. Use the Task tool with a **general-purpose** worker.

````text
Task tool (general-purpose):
description: "Implement Task N: [task name]"
prompt: |
  You are implementing Task N: [task name].

  ## Task Description
  [FULL TEXT of task from the plan - paste it here; do not make the worker read the file]

  ## Authoritative Context
  - Raw user input / discovery log: [PASTE relevant excerpt and/or path]
  - Approved design: [PASTE relevant excerpt and/or path]
  - Approved plan: [PASTE relevant excerpt and/or path]
  - Accumulated discoveries from earlier tasks: [PASTE bullet list]

  ## Context
  [Scene-setting: where this fits, dependencies, architectural context]

  ## Source-of-Truth Rules
  - The discovery log and raw user answers are authoritative.
  - If the task text appears to conflict with the discovery log or approved design, stop and ask the controller instead of guessing.
  - Do not widen scope.
  - Do not invent extra features.

  ## Before You Begin
  If you have questions about:
  - requirements or acceptance criteria
  - approach or implementation strategy
  - dependencies or assumptions
  - any conflict between the task and authoritative context

  Ask them now. Raise concerns before starting work.

  ## Your Job
  Once you're clear on requirements:
  1. Implement exactly what the task specifies
  2. Write tests (following TDD if the task says to)
  3. Verify the implementation works
  4. Commit your work
  5. Self-review
  6. Report back

  Work from: [directory]

  ## While You Work
  If you encounter something unexpected or unclear, ask questions. Do not guess.

  ## Before Reporting Back: Self-Review
  Review your work with fresh eyes.

  Ask yourself:

  **Completeness**
  - Did I fully implement everything in the task?
  - Did I miss any requirements?
  - Did I ignore anything from the discovery log or design context?

  **Quality**
  - Is this my best work?
  - Are names clear and accurate?
  - Is the code clean and maintainable?

  **Discipline**
  - Did I avoid overbuilding (YAGNI)?
  - Did I only build what was requested?
  - Did I follow existing patterns in the codebase?

  **Testing**
  - Do tests verify behavior?
  - Did I follow TDD if required?
  - Are tests comprehensive enough for this task?

  If you find issues during self-review, fix them before reporting.

  ## Report Format
  When done, report:
  - What you implemented
  - What you tested and test results
  - Files changed
  - Self-review findings (if any)
  - Project Discoveries: durable codebase facts later tasks should inherit
  - Any issues or concerns
````
