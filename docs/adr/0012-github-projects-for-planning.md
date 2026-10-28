# ADR-0012: GitHub Projects instead of Trello

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

The course values Jira-style user stories and visibility of each member's contribution. Jira is not used. Trello was considered.

## Decision

Use **GitHub Issues + GitHub Projects**: epics/stories/tasks as issues with sub-issues, custom fields (Type, Area, Priority, Size, Sprint iteration), board and roadmap views, automation linking PRs (`Closes #nn`) to issues. Process in [11](../11-process.md).

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Trello | Very simple | No link between cards and commits/PRs/reviews; contribution evidence scattered |
| Jira | Industry standard | Excluded by the team; separate tool |

## Consequences

Planning, code, reviews and CI in one place; contributions visible in Insights. Team must keep issues up to date.
