# Team Workflow

Spec-driven development process for this team. All projects follow this workflow.

## SPEC.md Structure

Each project repository contains a `SPEC.md` with these sections:

```
## Requirements
Kirin's requirements verbatim.

## Technical Design
Roy's technical specification: architecture, API contracts, tech stack, data model.

## Success Criteria
Hawkeye's verifiable checklist derived directly from Requirements.
Each criterion includes a test scenario: "Given X, when Y, then Z."
Accumulates across iterations — never removed, only added or marked complete.

## Open Issues
- [ ] [Advisory] Description — iteration N
- [x] [Advisory] Description — resolved in iteration N+1

## Changelog
- v0.1: Initial spec
- v0.2: One-line summary (details in git commit)
```

History is tracked via git (`git log SPEC.md`). No separate history file.

## Iteration Cycle

```
Kirin → requirements
  ↓
Roy → create/update SPEC.md (Requirements + Technical Design)
     → review and resolve Open Issues before starting (accept / defer / reject)
  ↓
Hawkeye → add Success Criteria to SPEC.md
  ↓
Roy → assign tasks to Breda (referencing SPEC.md)
  ↓
Breda → implement + write tests (Playwright for frontend, API test tool for backend)
       → run all tests, include results in completion report to Roy
  ↓
Hawkeye → evaluate against ALL Success Criteria in SPEC.md (new + regression)
         → [Blocking] issues: added to SPEC.md Open Issues, posted in channel — Kirin decides
         → [Advisory] issues: added to SPEC.md Open Issues, posted in channel — Roy decides
  ↓
Roy → if passing: report completion to Kirin
      if blocking: coordinate resolution with Kirin's final decision
```

## SPEC.md Scale Management

- Keep SPEC.md focused on current state. Changelog stays brief (one-liner per iteration).
- For large projects, split by module: `SPEC-api.md`, `SPEC-frontend.md`, etc.
- Project-specific rules go inside SPEC.md or a separate rules file referenced from SPEC.md.
