# Code Quality Reviewer Prompt Template

Use this template when dispatching a code quality reviewer worker.

**Purpose:** Verify implementation is well-built (clean, tested, maintainable).

**Only dispatch after spec compliance review passes.**

````text
Task tool (general-purpose):
description: "Review code quality for Task N"
prompt: |
  You are reviewing whether an implementation is well-built.

  ## What Was Implemented
  [From the implementer's report — what they claim they built]

  ## Task Requirements
  [FULL TEXT of task requirements from the plan]

  ## CRITICAL: Do Not Trust the Report
  The worker may be incomplete, inaccurate, or optimistic. Read the actual code.

  ## What to Check

  **Structure**
  - Does each file have one clear responsibility with a well-defined interface?
  - Are units decomposed so they can be understood and tested independently?
  - Is the implementation following the file structure from the plan?
  - Did this change create new files that are already large, or significantly grow existing files?
    (Don't flag pre-existing file sizes — focus on what this change contributed.)

  **Quality**
  - Are names clear and accurate?
  - Is the code clean and maintainable?
  - Are there unnecessary abstractions or overengineering?
  - Does the code follow existing patterns in the codebase?

  **Error Handling**
  - Are error cases handled appropriately?
  - Are there silent failures or swallowed exceptions?

  **Testing**
  - Do tests verify behavior, not implementation details?
  - Are edge cases covered?
  - Are tests readable and maintainable?

  ## Severity Definitions
  - Critical: The code has a serious defect, security issue, or architectural problem
  - Important: A meaningful quality issue that should be fixed before moving on
  - Minor: Good to improve, but does not block progress

  ## Your Job
  Report:
  - **Strengths:** What was done well
  - **Issues:** List with severity (Critical/Important/Minor), file:line references, and specific fix suggestions
  - **Assessment:** One of:
    - ✅ Code quality acceptable
    - ❌ Issues found: [list blocking issues]
````
