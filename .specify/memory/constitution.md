<!--
SYNC IMPACT REPORT
==================
Version change: 0.0.0 (uninitialized) → 1.0.0
Modified principles: N/A — initial ratification; all principles are new
Added sections:
  - Core Principles (4 principles: Code Quality, Testing Standards, UX Consistency, Performance)
  - Quality Gates
  - Development Workflow
  - Governance
Removed sections: N/A (first version)
Templates requiring updates:
  ✅ .specify/templates/plan-template.md — Constitution Check gates reflect these 4 principles
  ✅ .specify/templates/spec-template.md — Success Criteria must include UX + performance measurables
  ✅ .specify/templates/tasks-template.md — Task phases cover quality-gate, performance, and UX tasks
  ✅ .specify/templates/agent-file-template.md — Agent-agnostic; no CLAUDE-only references present
Follow-up TODOs: None — all placeholders resolved.
-->

# investment intel agents Constitution

## Core Principles

### I. Code Quality (NON-NEGOTIABLE)

Every piece of code merged to the main branch MUST meet the following standards:

- Code MUST pass all configured linter and static-analysis checks with zero warnings before merging.
- Functions and methods MUST have a single, clearly stated responsibility; side effects MUST be explicit
  and documented.
- Cyclomatic complexity per function MUST NOT exceed 10; any exception requires written justification
  in the PR description.
- All public APIs MUST include documentation comments (docstrings, JSDoc, or equivalent) describing
  purpose, parameters, return values, and thrown errors.
- Dead code, commented-out blocks, and unused imports MUST be removed before merging.
- Dependencies MUST be pinned to exact versions in lock files; unpinned ranges are not permitted in
  production manifests.

**Rationale**: Consistent quality reduces cognitive load during review and maintenance, prevents
accumulation of technical debt, and makes onboarding faster. These rules are machine-verifiable and
therefore enforceable without subjectivity.

### II. Testing Standards (NON-NEGOTIABLE)

Shipping untested code is a constitutional violation:

- Test-Driven Development (TDD) MUST be followed: tests are written and approved before implementation
  begins. The Red-Green-Refactor cycle is mandatory.
- Unit test coverage for all new or modified code MUST be ≥ 80% line coverage; critical paths (auth,
  payments, data mutations) MUST achieve ≥ 95%.
- Every public API contract change MUST be accompanied by a contract test update before the change ships.
- Integration tests MUST cover all inter-service communication boundaries and shared data schemas.
- Tests MUST be deterministic and isolated; flaky tests MUST be fixed or quarantined within one sprint
  of detection.
- Test files MUST be co-located with source or in a mirrored `tests/` tree; tests embedded in production
  modules are not permitted.

**Rationale**: Tests are the primary mechanism for verifying that the system behaves as specified.
Without enforceable coverage thresholds the constitution's intent cannot be validated automatically.

### III. User Experience Consistency

All user-facing surfaces MUST deliver a coherent, predictable experience:

- Every user interaction MUST follow the established design system (tokens, components, interaction
  patterns); one-off deviations MUST be approved by a UX review before implementation.
- Error messages MUST be human-readable, actionable, and free of raw stack traces or internal identifiers.
- Keyboard navigation and screen-reader accessibility (WCAG 2.1 AA) MUST be validated for every new
  or modified UI component.
- Copy and labels MUST use consistent terminology aligned with the project glossary; conflicting
  terminology requires a glossary update as part of the same PR.
- Loading, empty, and error states MUST be explicitly designed and implemented — they are not optional.

**Rationale**: Inconsistency erodes user trust and increases support burden. Explicit state coverage
prevents the common failure mode of shipping happy-path-only UIs.

### IV. Performance Requirements

Performance is a feature and MUST be treated as a first-class concern:

- Every feature that introduces a new user-facing interaction MUST define a p95 latency budget in its
  spec before implementation begins.
- API endpoints MUST respond within 200 ms p95 under baseline load; endpoints exceeding this threshold
  MUST document the justification and a remediation plan.
- Front-end pages MUST achieve Largest Contentful Paint (LCP) ≤ 2.5 s and Cumulative Layout Shift
  (CLS) ≤ 0.1 on a median device profile.
- Performance regressions > 10% vs. the established baseline MUST block merging until resolved or
  formally excepted with dual principal-engineer approval.
- Background jobs and batch processes MUST define throughput and memory-ceiling SLAs in the relevant
  spec.

**Rationale**: Performance budgets defined at spec time prevent expensive retrofitting of optimizations
after ship. Hard thresholds make automated enforcement feasible.

## Quality Gates

All pull requests MUST satisfy these gates before merge approval is granted:

1. **Lint + Static Analysis**: Zero errors; zero suppressed warnings without inline justification comment.
2. **Test Suite**: All tests pass; coverage thresholds met (see Principle II).
3. **Contract Tests**: Updated and passing if any API surface changed.
4. **Performance Baseline**: No regression > 10% on benchmark suite; p95 budget defined in spec.
5. **UX Review**: Required for any change touching user-facing copy, components, or interaction flows.
6. **Documentation**: Public API docs updated; CHANGELOG entry added for every user-visible change.

These gates are enforced in CI. A bypass requires dual approval from two principal engineers and MUST
be logged as a known exception in the relevant issue tracker.

## Development Workflow

- **Spec before code**: Every feature MUST have an approved `spec.md` before implementation begins.
  A `plan.md` MUST exist before the first line of production code is written.
- **Branch naming**: `###-short-description` where `###` is the issue number.
- **Commit messages**: Follow Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`, `test:`, `perf:`).
- **PR size**: Pull requests SHOULD target < 400 changed lines; larger PRs require decomposition rationale.
- **Review turnaround**: Reviewers MUST respond within one business day; blocked PRs escalate to the
  team lead.
- **Definition of Done**: Code merged + deployed to staging + acceptance criteria verified + monitoring
  confirmed healthy for 24 hours.

## Governance

This constitution supersedes all other documented practices. Where conflict exists, the constitution wins.

**Amendment procedure**:
1. Open a constitution-amendment issue describing the change, motivation, and impact on existing principles.
2. Obtain approval from at least two principal engineers and the project lead.
3. Update this file, increment the version per the semantic versioning policy below, and update the
   Sync Impact Report comment at the top of this file.
4. Propagate changes to all affected templates and guidance files in the same PR.

**Versioning policy**:
- MAJOR: Principle removal, redefinition, or backward-incompatible governance change.
- MINOR: New principle or section added; materially expanded guidance.
- PATCH: Clarifications, wording improvements, typo fixes, non-semantic refinements.

**Compliance review**: The constitution MUST be reviewed at the start of each quarter. The outcome
(no change / amendment / deprecation) MUST be recorded as a comment on the quarterly milestone.

**Version**: 1.0.0 | **Ratified**: 2026-03-18 | **Last Amended**: 2026-03-18
