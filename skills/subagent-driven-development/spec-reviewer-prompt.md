# Spec Compliance Review Worker Prompt Template

Use this template when dispatching a spec compliance reviewer worker.

**Purpose:** Verify the implementation matches what was requested — nothing more, nothing less.

````text
Task tool (general-purpose):
description: "Review spec compliance for Task N"
prompt: |
  You are reviewing whether an implementation matches its specification.

  ## What Was Requested
  [FULL TEXT of task requirements]

  ## Authoritative Context
  - Discovery log / raw user input: [PASTE relevant excerpt and/or path]
  - Approved design: [PASTE relevant excerpt and/or path]
  - Approved plan: [PASTE relevant excerpt and/or path]
  - Accumulated discoveries: [PASTE relevant excerpt if relevant]

  ## What Implementer Claims They Built
  [From the implementation worker's report]

  ## CRITICAL: Do Not Trust the Report
  The worker may be incomplete, inaccurate, or optimistic. Verify independently.

  ## Do Not
  - Take their word for what they implemented
  - Trust their completeness claims
  - Ignore conflicts with the discovery log or design

  ## Do
  - Read the actual code they wrote
  - Compare implementation to requirements line by line
  - Check for missing pieces
  - Check for extra work
  - Flag conflicts between task text and authoritative context

  ## Your Job
  Verify:
  - missing requirements
  - extra or overbuilt work
  - misunderstandings
  - conflicts with authoritative user answers or design intent

  Report one of:
  - ✅ Spec compliant
  - ❌ Issues found: [list specifically what's missing, extra, or conflicting, with file:line references]
````
