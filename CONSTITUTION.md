# Constitution

Overarching principles that apply to all projects by this team.

## Code Quality

- Prefer simplicity over cleverness. The simplest solution that meets requirements is the right solution.
- No speculative abstractions. Build for current needs, not hypothetical future ones.
- No unnecessary comments. Code should be self-evident; comments explain *why*, not *what*.
- Security at boundaries. Validate external input; trust internal logic and framework guarantees.
- Never hardcode secrets, credentials, or API keys in source code. Always use environment variables or `.env` files (gitignored). This applies even for local-only development — hardcoded secrets create deployment risks and bad habits.

## Testing

- Tests are written alongside implementation, not after. Never defer test writing to a later iteration.
- Frontend E2E tests use Playwright (`npx playwright`). Backend API tests use appropriate tooling per project.
- Test scenarios are defined by Hawkeye in SPEC.md as part of Success Criteria, before implementation begins.
- Breda runs all tests and includes results in the completion report to Roy.
- As projects grow, manual test execution transitions to automated CI runs — but scenarios written early remain valid.

## Communication

- Decisions made in Discord are authoritative. Ambiguity must be resolved with Kirin before proceeding.
- Report blockers immediately. Do not silently work around a blocker.

## Decision-Making

- Kirin makes the final call on all blocking issues and scope changes.
- Roy coordinates resolution but does not unilaterally decide on blocking matters.
- Hawkeye's evaluation is independent. Evaluations are based on Kirin's requirements, not Roy's interpretation alone.
