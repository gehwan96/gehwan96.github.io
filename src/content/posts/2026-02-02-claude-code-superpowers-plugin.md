---
title: claude code superpower plugin
pubDatetime: 2026-02-02T09:00:00+09:00
description: Claude Code Superpowers 플러그인의 14개 스킬을 요약하고 brainstorming 스킬로 설계 문서를 뽑아본 예시를 소개한다.
tags:
  - ai
---

### superpower brainstorming 사용 예시: Dooray 태스크 AI 요약 및 분할 기능 설계

> 작성일: 2026-02-02

#### 개요

Dooray 태스크 생성 시 AI를 활용하여 본문 요약 및 태스크 분할 기능을 제공한다.

#### 요구사항 정리

| 항목 | 내용 |
| --- | --- |
| **워크플로우** | 요약 → 분할 여부 결정 (순차적) |
| **AI 처리** | 연결된 AI 에이전트 활용 (Claude Code, Gemini 등) |
| **Dooray 연동** | MCP 서버로 태스크 생성 |
| **분할 방식** | AI 제안 + 사용자 조정 |
| **태스크 관계** | 하위 태스크(Subtask), 부모 없으면 부모 먼저 생성 |
| **요약 적용** | 미리보기 후 사용자가 적용 방식 선택 |
| **UI 위치** | 기존 CreateDoorayTaskDialog에 버튼 + 확장 패널 추가 |

#### UI/UX 흐름

##### 기본 흐름

```text
[기존 CreateDoorayTaskDialog]
     │
     ├── 제목 입력
     ├── 본문 입력 (textarea)
     │
     └── [🤖 AI 요약/분할] 버튼  ← 새로 추가
              │
              ▼
        ┌─────────────────────────────┐
        │  확장 패널 (접힘/펼침)        │
        │                             │
        │  [요약 중...] 로딩 표시       │
        │         ▼                   │
        │  ┌─ 요약 결과 ─┐            │
        │  │ (미리보기)   │            │
        │  └─────────────┘            │
        │                             │
        │  적용 방식:                  │
        │  ○ 본문 대체                 │
        │  ○ 본문 상단에 추가           │
        │  ○ 제목으로 사용              │
        │  [적용] [취소]               │
        │                             │
        │  ─────────────────          │
        │  [📋 태스크 분할] 버튼        │
        │         ▼                   │
        │  ┌─ 분할 제안 ─┐            │
        │  │ □ 태스크 1 (수정 가능)    │
        │  │ □ 태스크 2 (수정 가능)    │
        │  │ □ 태스크 3 (수정 가능)    │
        │  │ [+ 추가] [🗑 선택 삭제]   │
        │  └─────────────┘            │
        └─────────────────────────────┘
              │
              ▼
     [생성] → 부모 태스크 + 하위 태스크 일괄 생성
```

##### 핵심 인터랙션

- 요약과 분할은 **선택적** - 버튼을 누르지 않으면 기존 방식대로 단일 태스크 생성
- 분할된 각 태스크는 **인라인 편집** 가능
- 체크박스로 생성할 태스크 선택/해제

#### 컴포넌트 구조

##### 새로 생성할 컴포넌트

```text
frontend/src/components/dialogs/
├── CreateDoorayTaskDialog.tsx        (기존 - 수정)
│
├── dooray-ai/                        (새 폴더)
│   ├── AiSummaryPanel.tsx            # 요약 UI 패널
│   ├── AiSplitPanel.tsx              # 분할 UI 패널
│   ├── SplitTaskItem.tsx             # 분할된 개별 태스크 항목
│   └── types.ts                      # 타입 정의
```

##### 컴포넌트 역할

**AiSummaryPanel**

- AI 요약 요청 및 결과 표시
- 적용 방식 라디오 버튼 (대체/상단추가/제목)
- 적용/취소 버튼
- 로딩/에러 상태 처리

**AiSplitPanel**

- AI 분할 요청 및 제안 목록 표시
- 분할 태스크 추가/삭제 관리
- 전체 선택/해제 체크박스

**SplitTaskItem**

- 개별 분할 태스크의 체크박스 + 편집 UI
- 제목/설명 인라인 수정
- 삭제 버튼

##### 상태 관리

```ts
// CreateDoorayTaskDialog 내부 상태
interface DialogState {
  // 기존 상태
  title: string;
  body: string;

  // 새로 추가
  aiPanelOpen: boolean;
  summary: SummaryResult | null;
  summaryApplyMode: 'replace' | 'prepend' | 'title' | null;
  splitTasks: SplitTask[];
  selectedTaskIds: Set<string>;
}
```

#### AI 에이전트 연동

##### 연동 방식

Vibe Kanban에 연결된 AI 에이전트(Claude Code, Gemini 등)를 활용하여 요약/분할 요청을 처리한다.

```text
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Frontend   │────▶│  Vibe Kanban │────▶│  AI Agent       │
│  (Dialog)   │     │  Backend     │     │  (Claude Code)  │
└─────────────┘     └──────────────┘     └─────────────────┘
```

##### 새로운 Backend API 엔드포인트

```text
// crates/server/src/routes/dooray.rs에 추가

POST /dooray/ai/summarize
  Request:  { body: string, max_length?: number }
  Response: { summary: string, key_points: string[] }

POST /dooray/ai/split
  Request:  { title: string, body: string, context?: string }
  Response: {
    parent_title: string,
    tasks: [{ title: string, description: string }]
  }
```

##### 프롬프트 템플릿

```text
[요약 프롬프트]
다음 태스크 본문을 간결하게 요약해주세요.
핵심 포인트를 불릿으로 정리해주세요.
---
{body}

[분할 프롬프트]
다음 태스크를 논리적인 단위로 분할해주세요.
각 하위 태스크는 독립적으로 수행 가능해야 합니다.
---
제목: {title}
본문: {body}
```

#### Dooray MCP 연동

##### create_task 파라미터

```ts
{
  subject: string,              // 태스크 제목 (필수)
  body: {
    mime_type: "text/x-markdown" | "text/html",
    content: string             // 본문 내용
  },
  parent_post_id?: string,      // 하위 태스크 생성 시 부모 ID
  users?: {
    to: MemberRef[],            // 담당자
    cc: MemberRef[]             // 참조자
  },
  due_date?: string,            // ISO8601 형식
  milestone_id?: string,
  tag_ids?: string[],
  priority?: "none" | "highest" | "high" | "normal" | "low" | "lowest"
}
```

##### 생성 흐름

```text
[생성 버튼 클릭]
     │
     ▼
┌────────────────────────────────────────────┐
│  1. 부모 태스크 생성                        │
│                                            │
│  dooray-mcp: create_task({                 │
│    subject: "부모 태스크 제목",             │
│    body: {                                 │
│      mime_type: "text/x-markdown",         │
│      content: "요약된 본문 + 원본(선택)"    │
│    },                                      │
│    tag_ids: [...],                         │
│    priority: "normal"                      │
│  })                                        │
│  → 반환: { id: "parent_task_id", ... }     │
│                                            │
├────────────────────────────────────────────┤
│  2. 하위 태스크들 순차 생성                  │
│                                            │
│  for each splitTask:                       │
│    dooray-mcp: create_task({               │
│      subject: splitTask.title,             │
│      body: {                               │
│        mime_type: "text/x-markdown",       │
│        content: splitTask.description      │
│      },                                    │
│      parent_post_id: "parent_task_id"      │
│    })                                      │
│                                            │
└────────────────────────────────────────────┘
```

##### 추가 옵션

```ts
interface CreateOptions {
  inheritAssignees: boolean;   // 부모 담당자를 하위에도 적용
  inheritTags: boolean;        // 부모 태그를 하위에도 적용
  inheritPriority: boolean;    // 부모 우선순위 상속
}
```

#### 에러 처리

##### 에러 시나리오별 처리

| 시나리오 | 처리 방식 |
| --- | --- |
| AI 에이전트 미연결 | "AI 에이전트를 먼저 연결해주세요" 메시지 + 버튼 비활성화 |
| 요약 요청 실패 | 재시도 버튼 표시, 원본 본문 유지 |
| 분할 요청 실패 | 재시도 버튼 표시, 수동 분할 옵션 제공 |
| 부모 태스크 생성 실패 | 전체 생성 중단, 에러 메시지 표시 |
| 하위 태스크 일부 실패 | 성공한 태스크 목록 표시 + 실패 항목 재시도 옵션 |
| 본문이 너무 짧음 | "요약할 내용이 충분하지 않습니다" 안내 |
| Dooray API 권한 오류 | 프로젝트 권한 확인 안내 |

##### 로딩 상태 UI

```text
[요약 중]     ──▶  "본문을 분석하고 있습니다..." (스피너)
[분할 중]     ──▶  "태스크를 분할하고 있습니다..." (스피너)
[생성 중]     ──▶  "태스크 생성 중... (2/5)" (진행률 표시)
```

##### 엣지 케이스

- **빈 본문으로 요약 시도**: 요약 버튼 비활성화 (본문 최소 길이 체크)
- **분할 결과가 1개인 경우**: "분할할 내용이 없습니다. 단일 태스크로 생성할까요?" 확인
- **사용자가 모든 분할 태스크 체크 해제**: 생성 버튼 비활성화 + "최소 1개 이상 선택해주세요" 안내
- **요약 적용 후 다시 요약 요청**: 이전 요약 덮어쓰기 전 확인 모달

#### 구현 범위

##### Phase 1: 기본 기능

- AiSummaryPanel 컴포넌트
- AiSplitPanel 컴포넌트
- CreateDoorayTaskDialog 수정
- Backend API 엔드포인트 추가
- AI 에이전트 연동

##### Phase 2: 고급 기능

- 담당자/태그 상속 옵션
- 기존 태스크 임포트 시 요약/분할
- 분할 히스토리 저장

## Superpowers Skills 가이드

Claude Code에서 사용 가능한 Superpowers 스킬 요약입니다.

---

### 목차

- Using Superpowers
- Systematic Debugging
- Brainstorming
- Test-Driven Development
- Writing Plans
- Executing Plans
- Dispatching Parallel Agents
- Verification Before Completion
- Using Git Worktrees
- Requesting Code Review
- Receiving Code Review
- Finishing a Development Branch
- Subagent-Driven Development
- Writing Skills

---

### 1. /using-superpowers

**스킬이 1%라도 적용될 가능성이 있다면 반드시 해당 스킬을 먼저 호출**

- 프로세스 스킬 먼저 (brainstorming, debugging)
- 구현 스킬 그 다음

---

### 2. /systematic-debugging

**핵심:** 수정 시도 전에 반드시 근본 원인 찾기

| 단계 | 활동 |
| --- | --- |
| 1. 근본 원인 조사 | 에러 읽기, 재현, 최근 변경 확인 |
| 2. 패턴 분석 | 작동하는 예시와 비교 |
| 3. 가설 테스트 | 단일 가설, 최소 테스트 |
| 4. 구현 | 테스트 작성, 수정, 검증 |

**3번 이상 수정 실패 시 아키텍처 의심**

---

### 3. /brainstorming

**핵심:** 아이디어를 설계로 발전시키는 협업 대화

- 한 번에 하나의 질문으로 아이디어 이해
- 2-3가지 접근법과 장단점 제시
- 200-300 단어씩 설계 제시, 섹션마다 확인
- `docs/plans/YYYY-MM-DD-<topic>-design.md`에 저장

---

### 4. /test-driven-development

**핵심:** 테스트 먼저 → 실패 확인 → 최소 코드 → 리팩토링

| 단계 | 활동 |
| --- | --- |
| RED | 실패하는 테스트 작성, 실패 확인 |
| GREEN | 테스트 통과하는 최소 코드 |
| REFACTOR | 테스트 유지하며 정리 |

**테스트 없이 코드 작성 금지. 먼저 작성했다면 삭제.**

---

### 5. /writing-plans

**핵심:** 컨텍스트 없는 엔지니어도 따라할 수 있는 상세 계획

- 각 단계는 2-5분 분량 (테스트 작성 → 실패 확인 → 구현 → 통과 확인 → 커밋)
- 정확한 파일 경로, 완전한 코드
- 저장: `docs/plans/YYYY-MM-DD-<feature-name>.md`

---

### 6. /executing-plans

**핵심:** 배치 실행 + 체크포인트 리뷰

- 플랜 로드 및 비판적 검토
- 배치 실행 (기본 3개 태스크)
- 결과 보고 → 피드백 대기
- 피드백 반영 후 다음 배치
- 완료 후 `finishing-a-development-branch` 사용

---

### 7. /dispatching-parallel-agents

**핵심:** 독립적인 문제당 하나의 에이전트

**사용 시점:** 3개 이상의 독립적 문제 (다른 테스트 파일, 다른 서브시스템)

- 독립적 도메인 식별
- 에이전트별 명확한 범위 지정
- 병렬 디스패치
- 결과 통합

---

### 8. /verification-before-completion

**핵심:** 검증 증거 없이 완료 주장 금지

```text
명령 실행 → 출력 확인 → 그 다음 주장
```

"아마 될 거예요", "수정했습니다" (검증 없이) 금지

---

### 9. /using-git-worktrees

**핵심:** 격리된 작업 공간 생성

- 기존 `.worktrees/` 또는 `worktrees/` 확인
- CLAUDE.md 설정 확인
- 없으면 사용자에게 질문
- `.gitignore` 포함 확인 필수

---

### 10. /requesting-code-review

**핵심:** 이른 리뷰, 자주 리뷰

**필수 시점:**

- 각 태스크 완료 후
- 주요 기능 완료 후
- main 병합 전

---

### 11. /receiving-code-review

**핵심:** 기술적 평가, 감정적 동의 금지

- "완전 맞습니다!", "좋은 지적이에요!" 금지
- 기술적 요구사항 재진술 또는 근거 있는 반박
- 검증 후 구현

---

### 12. /finishing-a-development-branch

**핵심:** 테스트 검증 → 옵션 제시 → 실행 → 정리

**4가지 옵션:**

- 로컬 머지
- PR 생성
- 브랜치 유지
- 작업 삭제

---

### 13. /subagent-driven-development

**핵심:** 태스크당 새 서브에이전트 + 2단계 리뷰

```text
태스크 → 구현 에이전트 → 스펙 준수 리뷰 → 코드 품질 리뷰 → 다음 태스크
```

**스펙 준수 ✅ 전에 코드 품질 리뷰 시작 금지**

---

### 14. /writing-skills

**핵심:** 스킬 작성 = 문서에 TDD 적용

| 단계 | 활동 |
| --- | --- |
| RED | 스킬 없이 시나리오 실행, 실패 문서화 |
| GREEN | 실패 해결하는 최소 스킬 작성 |
| REFACTOR | 새 합리화 발견 → 대응 추가 |

---

### 스킬 관계도

```text
brainstorming (아이디어 → 설계)
       ↓
writing-plans (설계 → 상세 계획)
       ↓
using-git-worktrees (격리된 작업공간)
       ↓
subagent-driven-development 또는 executing-plans (구현)
       ↓
finishing-a-development-branch (완료/병합)
```

**지원 스킬:**

- `systematic-debugging` - 버그 발생 시
- `test-driven-development` - 모든 구현 시
- `verification-before-completion` - 완료 주장 전
- `requesting/receiving-code-review` - 리뷰 시
- `dispatching-parallel-agents` - 독립적 문제 여럿일 때

---

### 상황별 스킬 선택

| 상황 | 스킬 |
| --- | --- |
| 새 기능 시작 | `brainstorming` → `writing-plans` |
| 버그 발견 | `systematic-debugging` |
| 코드 작성 | `test-driven-development` |
| 플랜 실행 | `subagent-driven-development` 또는 `executing-plans` |
| 여러 독립 문제 | `dispatching-parallel-agents` |
| 완료 전 | `verification-before-completion` |
| 브랜치 마무리 | `finishing-a-development-branch` |
