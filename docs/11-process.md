# 11 — Process: Planning, Issues, Reviews

We work in a lightweight **Scrum** style with 2-week sprints, using **GitHub Projects + Issues** as the Jira replacement ([ADR-0012](adr/0012-github-projects-for-planning.md)). Everything is traceable: issue → branch → commits → PR → review → CI → merge → Done.

## 1. Work item hierarchy

```
Epic  (E1 … E10)                 e.g. "E5 Transfers"
 └── Story  (US-nn)              e.g. "US-11 Transfer to another customer"
      └── Task (sub-issue)       e.g. "[BE] POST /transfers endpoint", "[FE] Transfer form", "[QA] TC-23…29"
Bug                              standalone, linked to a story if relevant
```

| Epic | Name | Stories |
|---|---|---|
| E1 | Foundation (repo, CI, skeletons, DB) | technical tasks |
| E2 | Authentication | US-01 … US-04 |
| E3 | Customer & accounts | US-05 … US-09 |
| E4 | Ledger core | technical tasks (ledger, system accounts, invariants) |
| E5 | Transfers | US-10, US-11, US-13 |
| E6 | ATM & history | US-12, US-14, US-15 |
| E7 | Interest | US-16 |
| E8 | Administration | US-17 … US-21, US-23, US-24 |
| E9 | Audit | US-22 |
| E10 | Quality & release | e2e, deploy, docs, demo |

Epics are issues with label `type: epic` and sub-issues; stories are sub-issues of epics; tasks are sub-issues of stories.

## 2. GitHub Project setup

**Board columns (Status field):**

| Status | Meaning |
|---|---|
| Backlog | Not planned yet |
| Ready | Meets Definition of Ready, planned for the current sprint |
| In Progress | Someone is working on it (has assignee and branch) |
| In Review | PR open |
| Done | Merged and meets Definition of Done |

**Custom fields:**

| Field | Type | Values |
|---|---|---|
| Type | single select | Epic, Story, Task, Bug |
| Area | single select | Backend, Frontend, DevOps, QA, Docs |
| Priority | single select | P0 (blocker), P1 (high), P2 (normal), P3 (low) |
| Size | single select | 1, 2, 3, 5, 8 (story points; 8 = should be split) |
| Sprint | iteration (2 weeks) | Sprint 1 … Sprint 5 |


**Views:** Board (current sprint), Backlog table (grouped by Epic), My work (filter `assignee:@me`), Roadmap (by Sprint).

**Labels:** `type: story|task|bug|epic`, `area: backend|frontend|devops|qa|docs`, `blocked`, `good first issue`, `pkg` (extension package).

**Automation:** item added → Backlog; PR opened → In Review; PR merged/issue closed → Done.

## 3. Issue conventions

| Type | Title format | Example |
|---|---|---|
| Story | `US-nn: <story name>` | `US-11: Transfer to another customer` |
| Task | `[BE] / [FE] / [QA] / [OPS] / [DOC] <action>` | `[BE] Implement POST /transfers` |
| Bug | `[BUG] <short description>` | `[BUG] History shows wrong balanceAfter for own transfer` |

A story issue contains the story text and acceptance criteria copied from [02](02-requirements.md) as a checklist, plus links to the relevant FR/BR/TC IDs. Templates are in `.github/ISSUE_TEMPLATE/`.

### 3.1 Definition of Ready (story can enter a sprint)

- [ ] Story and acceptance criteria clear and in [02](02-requirements.md)
- [ ] Endpoints/DTOs it needs exist in [05](05-api-contract.md); data in [03](03-domain-model.md)
- [ ] Split into BE/FE/QA tasks, each ≤ 5 points
- [ ] Dependencies known (e.g. "needs ledger core")

### 3.2 Definition of Done

See [09 §6](09-testing.md#6-definition-of-done).

## 4. Pull requests and code review

- One PR per task (small PRs: ideally < 400 changed lines).
- PR title = conventional commit (`feat(payment): internal transfer endpoint`), body uses the template, contains `Closes #<issue>`.
- **Reviewers:**
  - Any PR: ≥ 1 approval from another team member.
  - PR changing `docs/03-domain-model.md` or `docs/05-api-contract.md`: approvals from **both** developers.
  - PR changing CI/deploy: approval from QA/DevOps.
- Review within 24 h on working days. Reviewer checks: requirements met, tests present, no business logic in controllers, money handling, security/ownership, naming matches docs.
- Merge strategy: **squash merge** into `develop` (one commit per PR, linked to the issue).
- Author merges after approval and green CI; deletes the branch.

## 5. Ceremonies

| Event | When | Duration | Output |
|---|---|---|---|
| Sprint planning | First day of sprint | 45 min | Sprint goal, stories moved to Ready, tasks assigned |
| Sync | 2× per week (e.g. Mon, Thu) or async in the team chat | 15 min | Done / next / blockers |
| API sync | When a story needs a new/changed endpoint | 15 min | Updated [05](05-api-contract.md) via docs PR **before** implementation |
| Sprint review | Last day of sprint | 30 min | Demo of Done stories on `develop` or demo env, release tag |
| Retrospective | After review | 15 min | 1–3 improvements noted in `docs/retro.md` |

## 6. Communication

- Team chat (Discord/Telegram) for quick questions; **decisions** go to issues, PRs or ADRs — not only to chat.
- Blockers: label `blocked` + comment explaining what is needed and from whom.
- Disagreements on architecture: short discussion → if no consensus within a day, the owner of the area decides and records an ADR.

## 7. Changing the documentation

1. Open an issue (type Task, area Docs) or include the change in the feature PR.
2. Edit the Markdown; keep IDs stable; add an entry to the changelog in [docs/README.md](README.md).
3. For a significant technical decision, add an ADR (`docs/adr/NNNN-title.md`, copy of the template) and link it.

## 8. Making contributions visible

For the course assessment, contributions are visible through:

- Issue assignees and the Project board history
- Commits and PR authorship, reviews and review comments
- GitHub Insights → Contributors and Pulse
- Release notes generated from PRs per sprint
- Sprint review notes in `docs/sprints/sprint-N.md` (goal, done, not done, who did what)
