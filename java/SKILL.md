---
name: java-dev
description: Use this skill to develop or debug Java applications
---

# Java Development

## Correctness

Prevent resource/classloader leaks, deadlocks, and silent data corruption.
Report concrete existing hazards with actionable technical context in a
`CRITICAL SAFETY WARNING` block. Runtime warnings should explain recovery.

Prefer deterministic bootstrap/build behavior, such as sorted collections.
In hot paths, prioritize efficiency; security and correctness take precedence.
Document tradeoffs when requirements conflict.

## Implementation

- Reuse existing helpers before adding utilities.
- Prefer thread confinement or event loops over shared state. Document
  single-thread assumptions and minimize critical sections.
- Prefer `final` for new variables and parameters unless mutation is needed.
- In hot paths, minimize allocations, streams, and boxing; prefer primitive
  collections or arrays. Prefer method handles over reflection and lambdas,
  except where lambdas improve examples or API tests.
- Keep changes focused: preserve existing formatting and signatures unless
  the task requires changing them. Do not add `@author` tags.
- Comment non-obvious reasons and tradeoffs; keep Javadoc brief.

## Build and Test

Prefer `mvnd`; fall back to `./mvnw` or `mvn`.
Include automated integration coverage; consult the user when impractical. Add unit tests for uncovered complex logic, avoiding tests
that duplicate integration coverage or tightly couple to implementation.
