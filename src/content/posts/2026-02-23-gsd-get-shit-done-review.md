---
title: gsd(get-shit-done) 사용 후기
pubDatetime: 2026-02-23T09:00:00+09:00
description: Orchestration-Subagent 패턴 기반 프로젝트 관리 프레임워크 GSD의 동작 방식, 명령어 흐름, 에이전트 구성과 직접 써본 장단점을 정리한다.
tags:
  - ai
---

## 개요

주말동안 GSD(get-shit-done)를 사용해보고 동작 방식 및 사용 후기에 대해 작성해봤습니다.

아래 내용은 조금 상세하다보니 안읽고 넘어가셔도 되고 TL;DR만 확인하셔도 됩니다.

## TL;DR

- **Orchestration-Subagent 패턴**을 사용할 경우 컨텍스트 오염이 줄어들어 더 정확한 결과물을 얻을 수 있다.
- 개발하기 전 다음과 같은 고려사항을 claude가 알 경우 좀 더 다양한 시각에서 내 요청(질문)을 보완해주고 결과적으로 좋은 결과물을 도출시켜준다
  - 예시
    - 리서치
      - code-base 리서치
        - 기술 스택 (언어, 프레임워크, 의존성)
        - 아키텍처 (모듈 구조, 레이어, 패턴)
        - 코드 컨벤션 (스타일, 네이밍, 패턴)
        - 우려사항 (기술 부채, 안티패턴, 리스크)
      - 프로젝트 리서치
        - 도메인/시장 조사
        - 기술 스택 생태계
        - 유사 프로젝트/패턴
        - 리스크/제약사항
      - 리서치 취합
        - 위 내용을 취합하여 재정리
- 목표를 관리하는 에이전트, 검증하는 에이전트가 존재할 경우 claude가 확장된 범위를 수정하지 않고 명확한 방향으로 나아간다.
  - 예시
    - 로드맵
      - 피쳐의 결과물 목표 제시
      - 프로젝트의 목적, 용도 작성
    - 검증
      - 작성한 코드를 바탕으로 앞서 확인한 (리서치, 로드맵)에 부합하는 결과물을 도출했는지 검사
      - 품질 검증

#### 개인의견

Orchestration-Subagent 패턴 좋아보인다. 다만 초반 에이전트 설계 및 구조를 만드는데 비용이 들 것 같다.

모든 개발을 gsd와 같은걸로 할 필요는 없어보인다 (리소스 낭비로 느껴진다.) 복잡도에 맞게 사용하는걸 추천한다.

프로젝트에서 태스크까지 관리하는 느낌이 들었다. 이제 점점 설계 문서(claude와의 대화)도 git에서 관리하는 시대가 오지 않을까 생각이 문득 든다.

![GSD가 생성한 .planning 디렉토리 구조](/images/gsd-get-shit-done-review/img-01.png)

**Orchestration-Subagent 패턴**을 활용해 우리가 진행하는 프로젝트에 알맞는 에이전트를 만들어 관리하면 좀 더 명확한 결과물을 얻을 수 있지 않을까? 하는 생각이 든다.

## GSD(get-shit-done)이란?

GSD(Get-Shit-Done)은 "Claude도 사람처럼 **생각 → 계획 → 실행 → 검토** 사이클로 일할 때 더 좋은 결과를 낸다"는 철학에서 출발한 AI-native 프로젝트 관리 프레임워크입니다.

기존에 출시된 Task-master, BMAD, Speckit와 같은 SDD(spec-driven development) tool들은 생각보다 너무 복잡하거나 시스템적으로 큰 그림에 대한 이해도가 부족하다는 느낌이 있는데 이 부분을 개선하고자 단순하지만, 내부적으로는 정교하게 설계된 시스템을 만들었다 합니다.

#### 링크

- [https://github.com/gsd-build/get-shit-done](https://github.com/gsd-build/get-shit-done)
- [https://github.com/gsd-build/get-shit-done/blob/main/docs/USER-GUIDE.md](https://github.com/gsd-build/get-shit-done/blob/main/docs/USER-GUIDE.md)

#### 주요 사용 패턴

GSD에서 사용한 패턴은 **Orchestration-Subagent 패턴**입니다.

총 11개의 전문화된 에이전트를 가지고 있으며 이를 Orchestrator를 통해 에이전트를 조율하는 방식입니다.

```text
[Orchestration-Subagent 방식]
Orchestrator: "어떤 에이전트가 뭘 해야 하는가"를 결정
Subagent-1:   연구 조사 (신선한 200k 컨텍스트)
Subagent-2:   계획 수립 (신선한 200k 컨텍스트)
Subagent-3:   코드 실행 (신선한 200k 컨텍스트)
Subagent-4:   결과 검증 (신선한 200k 컨텍스트)
```

### GSD 생명주기

GSD는 프로젝트의 시작부터 완료까지 전체 수명주기를 관리합니다

```text
/gsd:new-project
  → 4개 gsd-project-researcher 병렬 실행 (도메인/기술/패턴/리스크 조사)
  → gsd-research-synthesizer로 종합
  → gsd-roadmapper로 PROJECT.md + REQUIREMENTS.md + ROADMAP.md 생성

[Phase Loop: N번 반복]
  /gsd:plan-phase N   → PLAN.md
  /gsd:execute-phase N → SUMMARY.md, VERIFICATION.md

/gsd:audit-milestone
  → gsd-integration-checker로 크로스-페이즈 통합 검증
  → MILESTONE-AUDIT.md

/gsd:complete-milestone v1.0
  → milestones/ 아카이브
  → Git tag 생성
  → PROJECT.md 진화

/gsd:new-milestone → 다음 버전 반복
```

#### 명령어별 flow (detail)

**GSD Command Flow 레퍼런스**

> 각 명령의 **내부 동작 과정**과 **커맨드 간 연결 흐름**을 정리한 참고 문서.
> 에이전트 상세 → `gsd-agents.md` / plan-execute 상세 → `gsd-plan-execute.md` / 세션 관리 → `gsd-session-management.md`

---

##### 전체 커맨드 흐름 지도

```text
[신규 프로젝트]                    [기존 코드베이스]
/gsd:new-project                   /gsd:map-codebase
      │                                   │
      │                                   ▼
      │                            /gsd:new-project
      │                              (또는 new-milestone)
      ▼
   PROJECT.md
   REQUIREMENTS.md      ◄──────────────────────────────────────┐
   ROADMAP.md                                                   │
      │                                                         │
      ▼ [Phase Loop]                                            │
┌─────────────────────────────────────────────────────┐         │
│                                                     │         │
│  (선택) /gsd:discuss-phase N                         │         │
│         ↓ CONTEXT.md                               │         │
│                                                     │         │
│  /gsd:plan-phase N                                  │         │
│         ↓ PLAN.md                                   │         │
│                                                     │         │
│  /gsd:execute-phase N                               │         │
│         ↓ SUMMARY.md, VERIFICATION.md              │         │
│                                                     │         │
│  (선택) /gsd:verify-work N                          │         │
│         ↓ UAT.md                                   │         │
│                                                     │         │
└─────────────────────────────────────────────────────┘         │
      │                                                         │
      ▼                                                         │
/gsd:audit-milestone                                            │
      ↓ MILESTONE-AUDIT.md                                      │
      │                                                         │
      ├── 갭 발견 → /gsd:plan-milestone-gaps ──────────────────┘
      │
      ▼
/gsd:complete-milestone [v1.0]
      ↓ milestones/ 아카이브 + Git tag
      │
      ▼
/gsd:new-milestone  ← 다음 버전 반복
```

---

##### 프로젝트 초기화 커맨드

###### `/gsd:new-project`

신규 프로젝트를 처음 시작할 때. 대화형 질문 → 조사 → 요구사항 → 로드맵.

```text
/gsd:new-project
      │
      ▼
 [대화형 질문] Orchestrator가 직접 수행
      │ 프로젝트 목적, 기술 스택, 제약사항, 성공 기준
      │
      ▼
 [병렬 리서치] 4개 gsd-project-researcher 동시 실행
      ├── researcher-1: 도메인/시장 조사
      ├── researcher-2: 기술 스택 생태계
      ├── researcher-3: 유사 프로젝트/패턴
      └── researcher-4: 리스크/제약사항
      │
      ▼
 gsd-research-synthesizer
      │ → .planning/research/ 파일들 종합
      │
      ▼
 gsd-roadmapper
      │ → REQUIREMENTS.md (REQ-ID 포함)
      │ → ROADMAP.md (페이즈별 목표)
      │ → PROJECT.md (장기 메모리)
      │
      ▼
 STATE.md + config.json 초기화
      │
      ▼
 완료: Phase 1 계획 시작 안내
```

**플래그:**

- `-auto @document.md` : 기존 PRD 문서 자동 처리 (질문 생략)

---

###### `/gsd:map-codebase`

기존 코드베이스에 GSD를 적용하기 전에 코드 파악. `new-project` 또는 `new-milestone` 전에 실행.

```text
/gsd:map-codebase
      │
      ▼
 [병렬 탐색] 4개 gsd-codebase-mapper 동시 실행
      ├── mapper-1: 기술 스택 (언어, 프레임워크, 의존성)
      ├── mapper-2: 아키텍처 (모듈 구조, 레이어, 패턴)
      ├── mapper-3: 코드 컨벤션 (스타일, 네이밍, 패턴)
      └── mapper-4: 우려사항 (기술 부채, 안티패턴, 리스크)
      │
      ▼
 .planning/codebase/ 문서 생성
      ├── tech-stack.md
      ├── architecture.md
      ├── conventions.md
      └── concerns.md
      │
      ▼
 완료: new-project 또는 new-milestone 실행 안내
```

> 모델: Haiku (비용 최소화, 읽기만 수행)

---

###### `/gsd:new-milestone [name]`

이전 마일스톤 완료 후 다음 버전 시작.

```text
/gsd:new-milestone
      │
      ▼
 PROJECT.md 갱신 (이전 마일스톤 완료 기록)
      │
      ▼
 (선택) map-codebase 실행 안내
      │
      ▼
 새 ROADMAP.md 생성 (다음 버전 페이즈 계획)
      │
      ▼
 STATE.md 초기화
      │
      ▼
 Phase 1 계획 시작 안내
```

---

##### 페이즈 관리 커맨드

###### `/gsd:discuss-phase [N]`

plan-phase 전에 구현 방향을 미리 결정. 아키텍처 결정사항을 PLAN에 반영하기 위한 대화.

```text
/gsd:discuss-phase N
      │
      ▼
 ROADMAP.md에서 페이즈 N 목표 로드
      │
      ▼
 Orchestrator가 대화형 질문
      │ - 기술적 접근 방식 선택
      │ - 특별한 제약사항
      │ - 의존성/통합 포인트
      │ - 성능/보안 요구사항
      │
      ▼
 .planning/phases/NN-name/CONTEXT.md 생성
      │ (plan-phase 시 gsd-planner가 참조)
      │
      ▼
 plan-phase N 실행 안내
```

---

###### `/gsd:add-phase`

스코프가 늘었을 때 마일스톤 끝에 새 페이즈 추가.

```text
/gsd:add-phase
      │
      ▼
 ROADMAP.md 현재 페이즈 목록 확인
      │
      ▼
 Orchestrator가 새 페이즈 내용 질문
      │ - 목표, 범위, 산출물
      │
      ▼
 ROADMAP.md 끝에 새 페이즈 추가
      ▼
 완료
```

---

###### `/gsd:insert-phase [N]`

긴급 작업을 기존 페이즈 사이에 삽입. 소수점 번호 사용 (예: 3→3.1→4).

```text
/gsd:insert-phase 3
      │
      ▼
 ROADMAP.md에서 페이즈 3 이후 확인
      │
      ▼
 새 페이즈를 3.1로 넘버링
      │ phases/03.1-urgent-fix/ 디렉토리
      │
      ▼
 ROADMAP.md 갱신 (3.1 삽입, 기존 4,5... 유지)
      ▼
 discuss-phase 3.1 실행 안내
```

---

###### `/gsd:remove-phase [N]`

계획에서 미래 페이즈 제거 후 자동 재번호.

```text
/gsd:remove-phase N
      │
      ▼
 ROADMAP.md에서 페이즈 N 존재 확인
      │
      ▼
 실행 전 페이즈인지 확인 (실행된 페이즈는 제거 불가)
      │
      ▼
 ROADMAP.md에서 페이즈 N 삭제
      │
      ▼
 이후 페이즈 번호 재정렬 (N+1 → N, N+2 → N+1, ...)
      │
      ▼
 phases/ 디렉토리 정리 (해당 페이즈 폴더 삭제)
      ▼
 완료
```

---

###### `/gsd:list-phase-assumptions [N]`

plan-phase 실행 전 Claude가 어떤 방식으로 구현할지 미리 확인.

```text
/gsd:list-phase-assumptions N
      │
      ▼
 ROADMAP.md + PROJECT.md 읽기
      │
      ▼
 Orchestrator가 직접 분석하여 가정 목록 출력
      │ - 기술 선택 가정
      │ - 아키텍처 가정
      │ - 의존성 가정
      │ - 리스크 가정
      │
      ▼
 사용자가 가정 확인 → discuss-phase 또는 plan-phase 실행
```

> 코드 실행 없음, Orchestrator가 직접 추론하여 표시

---

##### 실행 & 검증 커맨드

###### `/gsd:plan-phase [N]`

→ 상세 플로우: `gsd-plan-execute.md`

요약:

```text
ROADMAP 확인 → (조사) → gsd-planner(PLAN.md 생성) → gsd-plan-checker(검증, 최대 3회) → 완료
```

---

###### `/gsd:execute-phase [N]`

→ 상세 플로우: `gsd-plan-execute.md`

요약:

```text
PLAN 목록 수집 → 의존성 분석 → 웨이브별 병렬 실행 → gsd-verifier(VERIFICATION.md) → 완료
```

---

###### `/gsd:verify-work [N]`

실행 후 수동 UAT. 실제 사용자 시나리오 기준으로 검증.

```text
/gsd:verify-work N
      │
      ▼
 PLAN.md의 success_criteria 로드
      │
      ▼
 Orchestrator가 대화형 UAT 진행
      │ - 각 기능을 사용자 관점에서 테스트
      │ - 엣지 케이스 확인
      │ - 실제 데이터로 시나리오 검증
      │
      ├── 문제 발견 → gsd-debugger 자동 진단 실행
      │              → 수정 계획 제안
      │
      ▼
 UAT.md 생성 (pass/fail 항목 목록)
      │
      ├── 모두 통과 → 다음 페이즈 안내
      └── 실패 존재 → /gsd:plan-phase N --gaps 안내
```

---

###### `/gsd:quick [description]`

Ad-hoc 소규모 작업 (버그 수정, 설정 변경 등). 전체 plan/execute 사이클 없이 GSD 보장.

```text
/gsd:quick "로그인 버튼 색상 변경"
      │
      ▼
 작업 범위 빠른 분석 (Orchestrator)
      │
      ▼
 gsd-executor 단일 실행
      │ - 자체 PLAN 미니 생성 (파일로 저장 안 함)
      │ - 코드 변경 실행
      │ - atomic commit
      │
      ▼
 빠른 검증 (gsd-verifier 없이 자체 확인)
      │
      ▼
 완료 (SUMMARY.md 간략 생성)
```

> plan-phase/execute-phase의 연구/검증 단계 생략 → 빠르고 가벼움

---

##### 마일스톤 완료 커맨드

###### `/gsd:audit-milestone`

모든 페이즈 완료 후 마일스톤 전체를 원래 요구사항과 대조 검증.

```text
/gsd:audit-milestone
      │
      ▼
 모든 페이즈 SUMMARY.md + REQUIREMENTS.md 로드
      │
      ▼
 gsd-integration-checker 실행
      │ 크로스-페이즈 통합 검증
      │ - 페이즈 간 이음새(wiring) 확인
      │ - E2E 플로우 유효성
      │ - REQ-ID 별 커버리지 확인
      │ - 스텁 코드 / TODO 탐지
      │ - 기술 부채 집계
      │
      ▼
 MILESTONE-AUDIT.md 생성
      │ ├── 요구사항 커버리지 매트릭스
      │ ├── 미충족 요구사항 목록
      │ ├── 발견된 갭/버그
      │ └── E2E 플로우 검증 결과
      │
      ├── 갭 있음 → /gsd:plan-milestone-gaps 안내
      └── 모두 통과 → /gsd:complete-milestone 안내
```

---

###### `/gsd:plan-milestone-gaps`

audit에서 발견된 갭을 새 페이즈로 계획.

```text
/gsd:plan-milestone-gaps
      │
      ▼
 MILESTONE-AUDIT.md 갭 목록 로드
      │
      ▼
 갭을 페이즈로 그룹화 (관련 갭 묶음)
      │
      ▼
 ROADMAP.md에 갭 수정 페이즈 추가
      │ (기존 페이즈 뒤에 N+1, N+2, ...)
      │
      ▼
 각 갭 페이즈 plan-phase 실행 안내
```

---

###### `/gsd:complete-milestone [version]`

마일스톤을 공식 완료하고 아카이브.

```text
/gsd:complete-milestone v1.0
      │
      ▼
 audit-milestone 완료 여부 확인
      │
      ▼
 milestones/ 아카이브
      │ ├── milestones/v1.0-ROADMAP.md
      │ ├── milestones/v1.0-REQUIREMENTS.md
      │ └── milestones/v1.0-SUMMARY.md
      │
      ▼
 Git tag 생성
      │ git tag v1.0 -m "Milestone v1.0 complete"
      │
      ▼
 PROJECT.md 갱신 (완료 마일스톤 기록)
      │
      ▼
 new-milestone 실행 안내
```

---

##### 디버깅 커맨드

###### `/gsd:debug [description]`

버그를 과학적 방법으로 체계적으로 조사.

```text
/gsd:debug "로그인 후 세션이 바로 만료됨"
      │
      ▼
 .planning/debug/[slug].md 생성 (조사 상태 추적)
      │
      ▼
 gsd-debugger 에이전트 실행
      │
      ▼
 [조사 사이클]
      │  1. 증상 기록 (예상 vs 실제 동작)
      │  2. 가설 형성 (원인 후보 목록)
      │  3. 가설 검증 (코드 탐색, 로그 분석)
      │  4. 근본 원인 특정
      │  └── 5. 수정 계획 작성
      │
      ├── 체크포인트: 사용자에게 중간 결과 보고
      │   (사용자 확인 후 계속 조사 또는 수정 시작)
      │
      ▼
 debug/[slug].md 최종 업데이트
      │ - 근본 원인
      │ - 수정 방법 (PRE/DURING/POST)
      │ - 재발 방지 방법
      │
      ▼
 수정 실행 → /gsd:quick 또는 수동 수정 안내
```

---

##### 유틸리티 커맨드

###### `/gsd:health [--repair]`

.planning/ 디렉토리 무결성 검사.

```text
/gsd:health
      │
      ▼
 필수 파일 존재 확인
      │ PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md, config.json
      │
      ▼
 STATE.md 유효성 검사
      │ - 참조하는 페이즈 번호가 ROADMAP과 일치하는가
      │
      ▼
 config.json 파싱 검사
      │
      ▼
 결과 출력 (에러/경고/정상)
      │
      ├── --repair 플래그 있음
      │    └── 자동 수정 가능한 문제 복구
      │        (STATE.md 재생성, config.json 기본값 복구)
      │
      └── --repair 없음 → 문제 목록만 출력
```

→ 에러 코드 상세: `gsd-session-management.md`

---

###### `/gsd:add-todo [description]`

아이디어나 할 일을 나중에 처리하기 위해 저장.

```text
/gsd:add-todo "인증 모듈에 rate limiting 추가 고려"
      │
      ▼
 .planning/todos/ 에 새 항목 저장
      │ timestamp + description + 현재 페이즈 컨텍스트
      │
      ▼
 완료 (check-todos로 나중에 확인)
```

---

###### `/gsd:check-todos`

저장된 todo 목록 확인 및 실행할 항목 선택.

```text
/gsd:check-todos
      │
      ▼
 .planning/todos/ 모든 항목 로드
      │
      ▼
 목록 출력 (날짜, 컨텍스트, 내용)
      │
      ▼
 사용자가 항목 선택
      ├── 페이즈로 전환 → add-phase 또는 insert-phase 안내
      ├── 즉시 실행 → /gsd:quick 안내
      └── 제거 → 해당 항목 삭제
```

---

###### `/gsd:settings`

워크플로우 설정 편집 (`.planning/config.json`).

```text
/gsd:settings
      │
      ▼
 config.json 현재 값 표시
      │
      ▼
 대화형 설정 변경
      │ - model_profile (quality/balanced/budget)
      │ - model_overrides (에이전트별 개별 모델)
      │ - worktree 사용 여부
      │ - 검증 게이트 활성화 여부
      │
      ▼
 config.json 저장
```

---

###### `/gsd:set-profile <profile>`

빠른 모델 프로필 전환.

```text
/gsd:set-profile balanced
      │
      ▼
 config.json의 model_profile 값 변경
      │ quality  → 전체 Opus 중심
      │ balanced → Opus 계획 + Sonnet 실행 (기본값)
      │ budget   → Sonnet/Haiku 전체
      │
      ▼
 완료 (다음 plan/execute부터 적용)
```

---

##### 네비게이션 커맨드

→ `/gsd:progress`, `/gsd:resume-work`, `/gsd:pause-work` 상세: `gsd-session-management.md`

요약:

| 커맨드 | 언제 사용 | 핵심 동작 |
| --- | --- | --- |
| `/gsd:progress` | 세션 시작, 현황 파악 | 상태 읽기 → 다음 액션 6가지 중 하나 안내 |
| `/gsd:resume-work` | 이전 세션에서 중단 후 재개 | .continue-here.md 또는 STATE.md로 컨텍스트 복원 |
| `/gsd:pause-work` | 세션 중간에 중단 | .continue-here.md 생성 + WIP 커밋 |

---

##### 커맨드 선택 치트시트

```text
신규 프로젝트 시작?
  ├── 빈 코드베이스 → /gsd:new-project
  └── 기존 코드베이스 → /gsd:map-codebase → /gsd:new-project

다음 단계를 모르겠음? → /gsd:progress

이전 세션 이어서 작업? → /gsd:resume-work

페이즈 계획 전 방향 조율? → /gsd:discuss-phase N

페이즈 계획 수립? → /gsd:plan-phase N

페이즈 실행? → /gsd:execute-phase N

실행 결과 확인? → /gsd:verify-work N

소규모 즉시 작업? → /gsd:quick "description"

버그 조사? → /gsd:debug "증상"

마일스톤 검수? → /gsd:audit-milestone

마일스톤 완료? → /gsd:complete-milestone v1.0

다음 버전 시작? → /gsd:new-milestone

중간에 급한 작업 삽입? → /gsd:insert-phase N

.planning/ 깨진 것 같음? → /gsd:health --repair
```

#### 주요 에이전트 목록

| 에이전트 | 역할 | 모델(balanced 프로필) |
| --- | --- | --- |
| `gsd-planner` | 페이즈를 PLAN.md로 분해 | **Opus** (가장 중요) |
| `gsd-executor` | PLAN.md를 읽고 코드 작성 | Sonnet |
| `gsd-phase-researcher` | 도메인 조사, 패턴 발견 | Sonnet |
| `gsd-plan-checker` | PLAN.md 품질 검증 | Sonnet |
| `gsd-verifier` | 목표 달성 여부 검증 | Sonnet |
| `gsd-debugger` | 과학적 버그 조사 | Sonnet |
| `gsd-codebase-mapper` | 기존 코드베이스 탐색 | **Haiku** (비용 절감) |
| `gsd-integration-checker` | 크로스-페이즈 통합 검증 | Sonnet |
| `gsd-roadmapper` | 로드맵 구조화 | Sonnet |
| `gsd-project-researcher` | 프로젝트 전체 도메인 조사 | Sonnet |
| `gsd-research-synthesizer` | 병렬 조사 결과 종합 | Sonnet |

## 사용 후기

Bmad는 안써봤으나 쓰면 쓸수록 살짝 지치는? 느낌을 받았습니다. 다만 결과물은 진짜 잘 도출시키고 필요한 내용을 구체적으로 많이 물어봐서 괜찮은 경험이었습니다.

#### 진빠지는 순서

**GSD > superpower > plan mode 순서**

#### 장점

- 내가 생각 못한 부분까지 상세하게 물어봄
- 포인트를 잘 잡음
- 개발 결과물이 좋음

#### 단점

- 생각보다 추론 및 개발 시간 소요가 큼
- 토큰 소모량도 큼
- 물어보는게 너무 많아서 병렬로 claude를 사용하기 어려울 것 같음
