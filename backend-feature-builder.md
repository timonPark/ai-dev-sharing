# backend-feature-builder — Full Specification

A dedicated agent for implementing new APIs, sync logic, aggregations, and integrations.
Execute the **9-step chain** below in order.

---

## Entry Point — `work-manager` skill

Every task starts with `/work-manager`.

```
/work-manager {Jira issue URL}
/work-manager {issue key}
```

It detects the input type and automatically routes to the appropriate agent.

| Input pattern | Routing |
|---|---|
| Jira issue URL / issue key | `_jira-flow` → classify type, then specialized agent |
| GitHub PR URL | `_pr-flow` → PR review procedure |

Routing by type:

| Work type | Specialized agent |
|---|---|
| FEAT (new feature) | `backend-feature-builder` ← this document |
| FIX (bug fix) | `bug-investigator` |
| PERF (performance improvement) | `performance-analyst` |
| REFACTOR | `refactor-guardian` |

### Transparency Principle `[Required at every step]`

At every step, output start, completion, and blockers in the format below.

```
▶ [Step name] {one-line description of what you're doing}
✅ [Step name] {one-line summary of completion result}
🚧 [Blocker] {situation description} — How should we proceed?
```

---

## Step 1 — Create branch + fetch issue (`issue-kickoff` agent)

Invoke the `issue-kickoff` agent. **Do not make GATE decisions.** Its sole job is to fetch and print the issue content as-is.

### 1-1. Create branch

- Fetch the Jira issue key and title.
- Confirm the base branch (`dev` / `release` / `master`) and branch type (`feature` / `fix` / `hotfix`, etc.).
- Create a branch in the format `{type}/{issue-key}-{english-title}` and push it to the remote.

### 1-2. Fetch issue content

- Print the issue details (description, comments, attachments/links) **verbatim without modification**.

### Output format

```
## Issue Kickoff Complete

- Issue key: {issue key}
- Issue title: {issue title}
- Branch: {branch name}
- Base: {base branch}

### Issue description
{full description}

### Comments
{comment list — "None" if there are no comments}
```

---

## Step 2 — Requirements check `[GATE]`

All four items below must be confirmed before moving to the next step.

| Check item | Sufficient criterion |
|---|---|
| Target user role | One of the service's user roles is specified |
| API path or feature scope | Endpoint or data flow to handle is described |
| Request/response fields | At least the core fields are specified |
| Business rules | State transitions, constraints, exception cases are mentioned |

**When information is insufficient:**

```
Implementation requirements could not be found in the Jira issue.
Please provide the following:

1. Target user role
2. API path or feature scope
3. Key request/response fields
4. Business rules or constraints
```

If partially satisfied → ask only about the missing items.

**Confirmation declaration followed by user confirmation:**

```
## Requirements Confirmed

- Issue: {issue key} — {issue title}
- Target user: {role}
- Feature scope: {description}
- Core fields: {request / response}
- Business rules: {rules}
- Reference branch: {branch name}

Shall we begin implementation with the above?
```

---

## Step 3 — Idea organization (`brainstorming` skill)

### Purpose

Ask design questions before implementing and agree on direction. **Finalize the design before writing a single line of code.**

### HARD GATE

> **No implementation activity whatsoever** until the design is presented and the user approves.
> Even tasks that look simple must go through this step without exception.

### Procedure

1. **Explore project context** — related files, recent commits, existing patterns
2. **Clarification questions** — one at a time, identifying purpose, constraints, success criteria
3. **Propose 2–3 implementation directions** — include trade-offs and the recommended direction
4. **Present the design** — architecture / layer structure / DTO·Type location / error design
5. **Save the design document** → `docs/{issue-key}/brainstorming.md`
6. **Self-review** — check for omissions, contradictions, ambiguity; revise
7. **Request user review** — after approval, move on to `writing-plans`

### Items to explore

- Implementation direction · design options · trade-offs
- Layer structure (Router → Controller → Service → Repository)
- DTO · Type locations
- Error design (error codes · messages · trigger conditions)
- Caching strategy (if needed)

### Core principles

- **One question at a time** — do not ask multiple questions at once.
- **Strict YAGNI** — do not include features not needed right now in the design.
- **Existing patterns first** — explore existing code before introducing new patterns.

---

## Step 4 — Write implementation plan (`writing-plans` skill)

### Purpose

Based on the Step 3 design, write an actionable per-layer implementation blueprint.

### HARD GATE (no commits)

> **Never perform commits or pushes** while writing or executing the plan.
> Do not include a commit step in plan tasks.
> Do not auto-commit on the basis that "the plan had a commit step."

### Save path

`docs/{issue-key}/implementation-plan.md`

### Plan header format

```markdown
# [Feature name] Implementation Plan

**Goal:** [One sentence — what is being built]

**Architecture:** [2–3 sentences — approach]

**Tech Stack:** [key technologies/libraries]

---
```

### Implementation file list

In the first section of the plan, list files by layer.

```markdown
## Implementation File List

| Layer | File path | Work |
|---|---|---|
| Router | src/routes/{domain}.router.ts | Add {method} /{path} route |
| Controller | src/controllers/{domain}.controller.ts | {handler name} handler |
| Service | src/services/{domain}.service.ts | {method name} method |
| DTO | src/dto/{domain}/{file}.dto.ts | New file |
| Tests | tests/{domain}.service.test.ts | Add {N} cases |
```

### Task structure (step-sized units — 2–5 minute work)

```markdown
### Task N: [Component name]

**Files:**
- Create: `exact/path/file.ts`
- Modify: `exact/path/existing-file.ts`
- Test: `tests/path/test-file.ts`

- [ ] Step 1: Write failing case tests
- [ ] Step 2: Run tests → confirm failure
- [ ] Step 3: Minimum implementation
- [ ] Step 4: Run tests → confirm pass
```

### Prohibited

Do not include the following in the plan:

- "TBD", "TODO", "implement later"
- "Add appropriate error handling" (without specific conditions)
- Steps that only describe "what should be done" without actual code

---

## Step 5 — TDD implementation (`tdd-service` skill)

### TDD cycle (iterate per function)

```
1. Write method stub            → throw new Error('not implemented')
2. Write failing-case test      → run tests → confirm failing case PASSES
3. Add success-case test        → run tests → confirm success case FAILS (Red)
4. Implement service logic      → run tests → confirm ALL PASS (Green)
```

> **Rule**: Before writing a success-case test, always confirm that the failing case passes first.

### Test file structure

```typescript
// ── 1. Mock external dependencies ──
jest.mock('path/to/cache-module', () => ({}));
jest.mock('path/to/logger', () => ({}));

// ── 2. Mock models ──
jest.mock('path/to/domain-model', () => ({ DomainModel: class {} }));

// ── 3. Mock repository ──
jest.mock('path/to/domain-repo', () => ({
    DomainRepository: { findById: jest.fn(), save: jest.fn() },
}));

// ── 4. Imports (after mock declarations) ──
import { DomainService } from 'path/to/domain-service';

// ── 5. Reference the mock functions ──
const mockRepo = jest.requireMock('path/to/domain-repo').DomainRepository;
const mockFindById = mockRepo.findById as jest.Mock;
```

### Failing-case pattern

```typescript
describe('DomainService.methodName', () => {
    afterEach(() => jest.clearAllMocks());

    describe('failing cases', () => {
        it('E1: throws VALIDATION_ERROR for invalid ID format', async () => {
            await expect(
                DomainService.inst.methodName('invalid-id'),
            ).rejects.toMatchObject({ code: 'VALIDATION_ERROR' });
            expect(mockFindById).not.toHaveBeenCalled();
        });

        it('E2: throws NOT_FOUND when target does not exist', async () => {
            mockFindById.mockResolvedValue(undefined);
            await expect(
                DomainService.inst.methodName('valid-id'),
            ).rejects.toMatchObject({ code: 'NOT_FOUND' });
        });
    });
```

Run the failing cases → confirm **E1, E2 PASS** before adding success cases.

### Success-case pattern

```typescript
    describe('success cases', () => {
        it('S1: returns correct response fields on normal lookup', async () => {
            const stub = makeDomain();
            mockFindById.mockResolvedValue(stub);

            const res = await DomainService.inst.methodName(stub.id);

            expect(res.field).toBe(stub.field);
        });
    });
```

Run the success case → confirm **S1 FAIL** (Red) → implement → confirm **S1 PASS** (Green).

### Checklist

- [ ] Does the method stub start with `throw new Error('not implemented')`?
- [ ] Were the failing cases (E1~) confirmed to PASS first?
- [ ] Were the success cases (S1~) confirmed to FAIL before implementation (Red)?
- [ ] Does the entire suite PASS after implementation (Green)?
- [ ] Are mock functions referenced via `jest.requireMock`?
- [ ] Is `afterEach(() => jest.clearAllMocks())` present?

### HARD RULE — Behavior after TDD completion

> ⚠️ After TDD completion, **never ask "Shall I commit?"**
> Immediately **auto-proceed to Step 6 (codex-review)** without user confirmation.
> Approval for commit/push can only be requested after Step 7 (review application) is complete.

---

## Step 6 — Code review (`codex-review` skill)

### Purpose

**Cross-AI verification** in which Codex (OpenAI) independently reviews code written by Claude.
This minimizes the confirmation bias that occurs when the same AI writes and reviews the code.

### Process

1. Verify the Codex binary path (`which codex` → if missing, search the install path)
2. Run Codex:
   ```bash
   {CODEX_PATH} exec "Please review this project using the /code-review skill.

   Additional context:
   - Design doc: docs/{issue-key}/brainstorming.md
   - Implementation plan: docs/{issue-key}/implementation-plan.md

   Please also verify that the code matches the implementation intent by referencing the above documents."
   ```
3. Guide user to the result path: `reviews/{branch-name}.md`

---

## Step 7 — Review feedback and application

Read `reviews/{branch-name}.md` and handle each item.

| Severity | Handling criterion |
|---|---|
| 🔴 CRITICAL | Must be addressed |
| 🟠 HIGH | Address (state reasons if skipped) |
| 🟡 MEDIUM | Review and address selectively |
| 🟢 LOW | For reference only |

- Addressed: fix the code + mark `[Addressed]`
- Not addressed: record the reason in the review file
- After application, re-run build/tests and verify everything passes

---

## Step 8 — Generate API documentation (`postman-doc` skill)

Analyze router, controller, service, schema, and type files to generate an API specification.

### File lookup order

1. `src/routes/{domain}.router.ts` — endpoints, middleware, auth info
2. `src/schema/{domain}.schema.ts` — request field validation rules
3. `src/types/{domain}.type.ts` — types, enums, interface definitions
4. `src/controllers/{domain}.controller.ts` — request handling flow
5. `src/services/{domain}.service.ts` — business logic, error handling

### Document structure

```markdown
# {Domain} API

> **Base Path**: `/api/v{N}/{path}`
> **Last Updated**: YYYY-MM-DD

## Endpoint list

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/{domain}/progress` | User JWT | Query processing progress |

## Request

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| jobId | string | Y | Job identifier |
| status | string | Y | Processing status |

## Response

```json
{
  "processedCount": 42,
  "totalCount": 100,
  "percentage": 42.0
}
```

## Error codes

| HTTP Status | Error code | Trigger condition |
|-------------|------------|-------------------|
| 404 | NOT_FOUND | Job does not exist |
| 400 | VALIDATION_ERROR | Invalid parameter format |

## curl example

```bash
curl -X GET '/api/v1/{domain}/progress?jobId={jobId}&status={status}' \
  -H 'Authorization: Bearer {token}'
```
```

### Save path

`docs/api/{domain}.md`

---

## Step 9 — Wrap up (`commit` → push → `pr`)

### Commit (`commit` skill)

**Precondition:** commit only after a successful build. On build failure, abort the commit and report the error.

**Commit format:**

```
<emoji> <type>(<issue-key>): <description>
```

**Type and emoji map:**

| Type | Emoji | Description |
|---|---|---|
| `feat` | ✨ | New feature |
| `fix` | 🐛 | Bug fix |
| `refactor` | ♻️ | Refactoring |
| `chore` | 🔧 | Config / dependencies |

**Rules:**
- If an issue key exists, always include it in the commit message
- Do not include gitignored files in commits
- Do not mention AI (Claude, Codex, etc.) in commit messages
- One commit contains one logical change

### PR (`pr` skill)

**Target branch resolution:**

| Branch type | PR target |
|---|---|
| `hotfix/` | `release` (merged separately into `master` afterwards) |
| `feature/`, `fix/`, etc. | The base branch the branch was cut from |
| Ambiguous | Confirm with the user before proceeding |

**PR body template:**

```markdown
## Issue link
[{issue-key}]({Jira URL})

## Summary
> Explain in 1–3 lines why this PR is needed and what problem it solves

## Key changes
- **`filename`**: description of change

## Notes
> Anything a reviewer should know, how to test, etc. (omit the section if none)
```

**Rules:**
- Do not mention AI in the PR body
- Do not auto-assign reviewers; the user selects them

---

## Overall flow summary

```
Input: "Implement new API for {issue-key}"
    │
    ▼
/work-manager
    │  Detect input type → classify → invoke backend-feature-builder
    │
    ▼
Step 1  issue-kickoff    Create branch + fetch full issue content
Step 2  [GATE]           Verify 4 requirement items → human approval
Step 3  brainstorming    Explore design → docs/{issue-key}/brainstorming.md
Step 4  writing-plans    Per-layer implementation blueprint → implementation-plan.md
Step 5  tdd-service      Fail → success → implement in order, confirm all tests pass
Step 6  codex-review     Codex cross-verifies Claude's code → reviews/*.md
Step 7  Apply review     Handle by severity + re-verify build/tests
Step 8  postman-doc      Auto-generate API spec → docs/api/{domain}.md
Step 9  commit/push/pr   Generate commit message → push → create PR
    │
    ▼
Output: PR URL
```
