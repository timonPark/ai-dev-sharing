# backend-feature-builder — 전체 명세

신규 API·동기화 로직·집계·연동 기능 구현 전담 에이전트.
아래 **9단계 체인**을 순서대로 실행한다.

---

## 진입점 — `work-manager` 스킬

모든 작업은 `/work-manager`로 시작한다.

```
/work-manager {Jira 이슈 URL}
/work-manager {이슈키}
```

입력 유형을 감지해 적절한 에이전트로 자동 라우팅한다.

| 입력 패턴 | 라우팅 |
|---|---|
| Jira 이슈 URL / 이슈키 | `_jira-flow` → 유형 분류 후 전문 에이전트 |
| GitHub PR URL | `_pr-flow` → PR 리뷰 절차 |

유형별 라우팅:

| 업무 유형 | 전문 에이전트 |
|---|---|
| FEAT (신규 기능) | `backend-feature-builder` ← 이 문서 |
| FIX (버그 수정) | `bug-investigator` |
| PERF (성능 개선) | `performance-analyst` |
| REFACTOR | `refactor-guardian` |

### 투명성 원칙 `[전 단계 필수]`

모든 단계에서 시작·완료·병목을 아래 형식으로 출력한다.

```
▶ [단계명] {하려는 작업 한 줄 설명}
✅ [단계명] {완료 결과 한 줄 요약}
🚧 [병목] {상황 설명} — 어떻게 진행할까요?
```

---

## Step 1 — 브랜치 생성 + 이슈 조회 (`issue-kickoff` 에이전트)

`issue-kickoff` 에이전트를 호출한다. **GATE 판단은 하지 않는다.** 이슈 내용을 조회해서 그대로 출력하는 것이 전부다.

### 1-1. 브랜치 생성

- Jira 이슈 키와 제목을 조회한다.
- 베이스 브랜치(`dev` / `release` / `master`)와 브랜치 성격(`feature` / `fix` / `hotfix` 등)을 확인한다.
- `{성격}/{이슈키}-{영문제목}` 형식으로 브랜치를 생성하고 원격에 푸시한다.

### 1-2. 이슈 내용 조회

- 이슈 상세 내용(설명, 댓글, 첨부·링크)을 **수정 없이 전문 출력**한다.

### 출력 형식

```
## Issue Kickoff 완료

- 이슈키: {이슈키}
- 이슈 제목: {이슈 제목}
- 브랜치: {브랜치명}
- 베이스: {베이스 브랜치}

### 이슈 내용
{description 전문}

### 댓글
{댓글 목록 — 없으면 "없음"}
```

---

## Step 2 — 요구사항 확인 `[GATE]`

아래 4가지가 모두 확인되어야 다음 단계로 넘어간다.

| 확인 항목 | 충분 기준 |
|---|---|
| 대상 사용자 역할 | 서비스 내 사용자 역할 중 하나가 명시됨 |
| API 경로 또는 기능 범위 | 엔드포인트 또는 처리할 데이터 흐름이 기술됨 |
| 요청/응답 필드 | 최소 핵심 필드가 명시됨 |
| 비즈니스 룰 | 상태 전이, 제약 조건, 예외 케이스가 언급됨 |

**정보 불충분 시:**

```
Jira 이슈에서 구현 요구사항을 찾지 못했습니다.
아래 항목을 알려주세요:

1. 대상 사용자 역할
2. API 경로 또는 기능 범위
3. 주요 요청/응답 필드
4. 비즈니스 룰 또는 제약 조건
```

부분 충족 시 → 누락 항목만 질문한다.

**확정 선언 후 사용자 확인:**

```
## 요구사항 확정

- 이슈: {이슈키} — {이슈제목}
- 대상 사용자: {역할}
- 기능 범위: {설명}
- 핵심 필드: {요청 / 응답}
- 비즈니스 룰: {규칙}
- 참조 브랜치: {브랜치명}

위 내용으로 구현을 시작해도 될까요?
```

---

## Step 3 — 아이디어 정리 (`brainstorming` 스킬)

### 목적

구현 전에 설계 질문을 던지고 방향을 합의한다. **코드 한 줄도 작성하기 전에 설계를 확정한다.**

### HARD GATE

> 설계를 제시하고 사용자가 승인할 때까지 **구현 행동 일체 금지.**
> 단순해 보이는 작업도 예외 없이 이 단계를 거친다.

### 진행 절차

1. **프로젝트 컨텍스트 탐색** — 관련 파일, 최근 커밋, 기존 패턴 확인
2. **명확화 질문** — 한 번에 하나씩, 목적·제약·성공 기준 파악
3. **2~3가지 구현 방향 제안** — 트레이드오프와 추천 방향 포함
4. **설계 제시** — 아키텍처 / 레이어 구조 / DTO·Type 위치 / 에러 설계
5. **설계 문서 저장** → `docs/{이슈키}/brainstorming.md`
6. **자체 검토** — 누락·모순·모호함 확인 후 수정
7. **사용자 검토 요청** — 승인 후 `writing-plans`로 이동

### 탐색 항목

- 구현 방향 · 설계 선택지 · 트레이드오프
- 레이어 구조 (Router → Controller → Service → Repository)
- DTO · Type 위치
- 에러 설계 (에러 코드 · 메시지 · 발생 조건)
- 캐시 전략 (필요 시)

### 핵심 원칙

- **질문은 한 번에 하나** — 여러 질문을 한꺼번에 던지지 않는다.
- **YAGNI 철저 적용** — 지금 필요하지 않은 기능은 설계에 포함하지 않는다.
- **기존 패턴 우선** — 신규 패턴 도입 전 기존 코드 탐색한다.

---

## Step 4 — 구현 계획 작성 (`writing-plans` 스킬)

### 목적

Step 3 설계를 기반으로 실행 가능한 레이어별 구현 청사진을 작성한다.

### HARD GATE (커밋 금지)

> 계획서 작성 및 실행 중 **커밋·푸시는 절대 수행하지 않는다.**
> 계획서 Task 안에 커밋 스텝을 작성하지 않는다.
> "계획서에 커밋 스텝이 있었다"는 이유로 커밋을 자동 실행하지 않는다.

### 저장 경로

`docs/{이슈키}/implementation-plan.md`

### 계획서 헤더 형식

```markdown
# [기능명] 구현 계획

**Goal:** [한 문장 — 무엇을 만드는지]

**Architecture:** [2~3문장 — 접근 방식]

**Tech Stack:** [핵심 기술/라이브러리]

---
```

### 구현 파일 목록

계획서 첫 번째 섹션에 레이어별 파일 목록을 작성한다.

```markdown
## 구현 파일 목록

| 레이어 | 파일 경로 | 작업 내용 |
|---|---|---|
| Router | src/routes/{도메인}.router.ts | {메서드} /{경로} 라우트 추가 |
| Controller | src/controllers/{도메인}.controller.ts | {핸들러명} 핸들러 |
| Service | src/services/{도메인}.service.ts | {메서드명} 메서드 |
| DTO | src/dto/{도메인}/{파일명}.dto.ts | 신규 생성 |
| 테스트 | tests/{도메인}.service.test.ts | {N}케이스 추가 |
```

### Task 구조 (단계 단위 — 2~5분 작업)

```markdown
### Task N: [컴포넌트명]

**Files:**
- Create: `정확한/경로/파일.ts`
- Modify: `정확한/경로/기존파일.ts`
- Test: `tests/경로/테스트파일.ts`

- [ ] Step 1: 실패 케이스 테스트 작성
- [ ] Step 2: 테스트 실행 → 실패 확인
- [ ] Step 3: 최소 구현
- [ ] Step 4: 테스트 실행 → 통과 확인
```

### 금지 사항

계획서에 아래 내용을 작성하지 않는다:

- "TBD", "TODO", "나중에 구현"
- "적절한 에러 처리 추가" (구체적 조건 없이)
- 실제 코드 없이 "무엇을 해야 한다"만 설명하는 단계

---

## Step 5 — TDD 구현 (`tdd-service` 스킬)

### TDD 사이클 (함수 단위 반복)

```
1. 메서드 껍데기(stub) 작성   → throw new Error('not implemented')
2. 실패 케이스 테스트 작성    → 테스트 실행 → 실패 케이스 PASS 확인
3. 성공 케이스 테스트 추가    → 테스트 실행 → 성공 케이스 FAIL 확인 (Red)
4. 서비스 로직 구현           → 테스트 실행 → 전체 PASS 확인 (Green)
```

> **규칙**: 성공 케이스 테스트를 작성하기 전에 반드시 실패 케이스가 먼저 통과하는지 확인한다.

### 테스트 파일 구조

```typescript
// ── 1. 외부 의존성 mock ──
jest.mock('경로/캐시모듈', () => ({}));
jest.mock('경로/로거', () => ({}));

// ── 2. 모델 mock ──
jest.mock('경로/도메인모델', () => ({ DomainModel: class {} }));

// ── 3. 레포지토리 mock ──
jest.mock('경로/도메인레포', () => ({
    DomainRepository: { findById: jest.fn(), save: jest.fn() },
}));

// ── 4. import (mock 선언 이후) ──
import { DomainService } from '경로/도메인서비스';

// ── 5. mock 함수 참조 ──
const mockRepo = jest.requireMock('경로/도메인레포').DomainRepository;
const mockFindById = mockRepo.findById as jest.Mock;
```

### 실패 케이스 작성 패턴

```typescript
describe('DomainService.methodName', () => {
    afterEach(() => jest.clearAllMocks());

    describe('실패 케이스', () => {
        it('E1: 유효하지 않은 ID 형식이면 VALIDATION_ERROR throw', async () => {
            await expect(
                DomainService.inst.methodName('invalid-id'),
            ).rejects.toMatchObject({ code: 'VALIDATION_ERROR' });
            expect(mockFindById).not.toHaveBeenCalled();
        });

        it('E2: 대상이 없으면 NOT_FOUND throw', async () => {
            mockFindById.mockResolvedValue(undefined);
            await expect(
                DomainService.inst.methodName('valid-id'),
            ).rejects.toMatchObject({ code: 'NOT_FOUND' });
        });
    });
```

실패 케이스 실행 → **E1, E2 PASS** 확인 후 성공 케이스 추가.

### 성공 케이스 작성 패턴

```typescript
    describe('성공 케이스', () => {
        it('S1: 정상 조회 시 응답 필드가 올바르게 반환된다', async () => {
            const stub = makeDomain();
            mockFindById.mockResolvedValue(stub);

            const res = await DomainService.inst.methodName(stub.id);

            expect(res.field).toBe(stub.field);
        });
    });
```

성공 케이스 실행 → **S1 FAIL** 확인 (Red) → 구현 → **S1 PASS** 확인 (Green).

### 체크리스트

- [ ] 메서드 스텁이 `throw new Error('not implemented')`로 시작하는가
- [ ] 실패 케이스(E1~)가 먼저 PASS하는 것을 확인했는가
- [ ] 성공 케이스(S1~)가 구현 전 FAIL하는 것을 확인했는가 (Red)
- [ ] 구현 후 전체 케이스가 PASS하는가 (Green)
- [ ] `jest.requireMock`으로 mock 함수를 참조하는가
- [ ] `afterEach(() => jest.clearAllMocks())`가 있는가

### HARD RULE — TDD 완료 후 행동 규칙

> ⚠️ TDD 완료 후 **"커밋할까요?" 질문 절대 금지.**
> 사용자 확인 없이 **즉시 Step 6(codex-review)으로 자동 진행.**
> 커밋·푸시 승인은 Step 7(리뷰 반영) 완료 후에만 요청 가능.

---

## Step 6 — 코드 리뷰 (`codex-review` 스킬)

### 목적

Claude가 작성한 코드를 Codex(OpenAI)가 독립적으로 검토하는 **AI 간 교차 검증.**
동일 AI가 작성·검토할 때 발생하는 확증 편향을 최소화한다.

### 프로세스

1. Codex 바이너리 경로 확인 (`which codex` → 없으면 설치 경로 탐색)
2. Codex 실행:
   ```bash
   {CODEX_PATH} exec "이 프로젝트에 /code-review 스킬을 사용해서 리뷰해주세요.

   추가 컨텍스트:
   - 설계 문서: docs/{이슈키}/brainstorming.md
   - 구현 계획: docs/{이슈키}/implementation-plan.md

   위 문서를 참고해 구현 의도에 맞게 코드가 작성되었는지 함께 검토해 주세요."
   ```
3. 결과 경로 안내: `reviews/{브랜치명}.md`

---

## Step 7 — 리뷰 결과 검토 및 반영

`reviews/{브랜치명}.md` 를 읽고 항목별로 처리한다.

| 심각도 | 처리 기준 |
|---|---|
| 🔴 CRITICAL | 반드시 반영 |
| 🟠 HIGH | 반영 (미반영 시 이유 명시) |
| 🟡 MEDIUM | 검토 후 선택 반영 |
| 🟢 LOW | 참고만 |

- 반영: 코드 수정 + `[반영됨]` 표시
- 미반영: 이유를 리뷰 파일에 기록
- 반영 완료 후 빌드·테스트 재실행 및 전체 통과 확인

---

## Step 8 — API 문서 생성 (`postman-doc` 스킬)

라우터·컨트롤러·서비스·스키마·타입 파일을 분석하여 API 명세서를 생성한다.

### 분석 파일 탐색 순서

1. `src/routes/{도메인}.router.ts` — 엔드포인트·미들웨어·인증 정보
2. `src/schema/{도메인}.schema.ts` — 요청 필드 검증 규칙
3. `src/types/{도메인}.type.ts` — 타입·enum·인터페이스 정의
4. `src/controllers/{도메인}.controller.ts` — 요청 처리 흐름
5. `src/services/{도메인}.service.ts` — 비즈니스 로직·에러 처리

### 문서 구조

```markdown
# {도메인} API

> **Base Path**: `/api/v{N}/{경로}`
> **최종 수정**: YYYY-MM-DD

## 엔드포인트 목록

| Method | Path | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/{도메인}/progress` | 사용자 JWT | 처리 진행률 조회 |

## Request

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| jobId | string | Y | 작업 식별자 |
| status | string | Y | 처리 상태 |

## Response

```json
{
  "processedCount": 42,
  "totalCount": 100,
  "percentage": 42.0
}
```

## 에러 코드

| HTTP Status | 에러 코드 | 발생 조건 |
|-------------|-----------|-----------|
| 404 | NOT_FOUND | 작업 미존재 |
| 400 | VALIDATION_ERROR | 파라미터 형식 오류 |

## curl 예시

```bash
curl -X GET '/api/v1/{도메인}/progress?jobId={jobId}&status={status}' \
  -H 'Authorization: Bearer {token}'
```
```

### 저장 경로

`docs/api/{도메인}.md`

---

## Step 9 — 마무리 (`commit` → push → `pr`)

### 커밋 (`commit` 스킬)

**사전 조건:** 빌드 성공 후 커밋한다. 빌드 실패 시 커밋 중단·오류 보고.

**커밋 포맷:**

```
<이모지> <타입>(<이슈키>): <설명>
```

**타입 및 이모지 맵:**

| 타입 | 이모지 | 설명 |
|---|---|---|
| `feat` | ✨ | 기능 추가 |
| `fix` | 🐛 | 버그 수정 |
| `refactor` | ♻️ | 리팩토링 |
| `chore` | 🔧 | 설정·의존성 |

**규칙:**
- 이슈키가 있으면 커밋 메시지에 반드시 포함
- gitignore 대상 파일은 커밋에 포함하지 않는다
- AI 관련 언급(Claude, Codex 등)을 커밋 메시지에 작성하지 않는다
- 한 커밋에는 하나의 논리적 변경만 담는다

### PR (`pr` 스킬)

**PR 타겟 브랜치 결정:**

| 브랜치 성격 | PR 타겟 |
|---|---|
| `hotfix/` | `release` (이후 `master` 별도 머지) |
| `feature/`, `fix/` 등 | 브랜치를 딴 베이스 브랜치 |
| 불명확한 경우 | 사용자에게 확인 후 진행 |

**PR 본문 템플릿:**

```markdown
## 이슈 링크
[{이슈키}]({Jira URL})

## 내용 요약
> 이 PR이 왜 필요한지, 어떤 문제를 해결하는지 1~3줄로 설명

## 주요 변경 내용
- **`파일명`**: 변경 내용 설명

## 참고 사항
> 리뷰어가 알아야 할 특이사항, 테스트 방법 등 (없으면 섹션 생략)
```

**규칙:**
- AI 관련 언급을 PR 본문에 작성하지 않는다
- 리뷰어는 자동 지정하지 않고 사용자가 선택한다

---

## 전체 흐름 요약

```
입력: "{이슈키} 신규 API 구현해줘"
    │
    ▼
/work-manager
    │  입력 유형 감지 → 유형 분류 → backend-feature-builder 호출
    │
    ▼
Step 1  issue-kickoff    브랜치 생성 + 이슈 내용 전문 조회
Step 2  [GATE]           요구사항 4가지 확인 → 사람 승인
Step 3  brainstorming    설계 탐색 → docs/{이슈키}/brainstorming.md
Step 4  writing-plans    레이어별 구현 청사진 → implementation-plan.md
Step 5  tdd-service      실패→성공→구현 순서 엄수, 전체 테스트 통과 확인
Step 6  codex-review     Claude 코드를 Codex가 교차 검증 → reviews/*.md
Step 7  리뷰 반영        심각도별 처리 + 빌드·테스트 재확인
Step 8  postman-doc      API 명세서 자동 생성 → docs/api/{도메인}.md
Step 9  commit/push/pr   커밋 메시지 생성 → push → PR 생성
    │
    ▼
출력: PR URL
```
