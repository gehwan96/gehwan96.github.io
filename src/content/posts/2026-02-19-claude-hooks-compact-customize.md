---
title: Claude Hooks - compact customize 하기
pubDatetime: 2026-02-19T09:00:00+09:00
description: Claude Code compact의 동작과 한계를 짚고, PreCompact/SessionStart hook으로 요약 품질을 높이고 HANDOFF.md로 맥락을 보존하는 방법을 정리한다.
tags:
  - ai
---

**대상**: Claude Code를 이미 사용 중인 개발자

**범위**: compact가 언제/왜 발생하는지 + PreCompact hook으로 요약 품질을 높이고 HANDOFF.md로 맥락을 보존하는 방법

**비범위**: context window 상세 이론, API 서버 측 compaction 베타 기능

## 1. Compact란 무엇인가

### 왜 필요한가

Claude Code의 대화는 턴마다 토큰이 선형적으로 쌓인다. 사용자 메시지, 어시스턴트 응답, 도구 호출 결과, 읽어들인 파일 내용이 모두 컨텍스트에 누적된다. 기본 컨텍스트 윈도우 한도는 **200K 토큰**이며, Claude Sonnet 3.7 이후 모델은 한도 초과 시 자동으로 잘라내는 대신 **오류를 반환**한다.

compact는 이 한도에 도달하기 전에 전체 대화를 요약으로 압축하여 컨텍스트를 확보하는 메커니즘이다.

> 참고: extended thinking 블록은 다음 턴에서 API가 자동으로 제거한다. Claude Sonnet 4.6/4.5, Haiku 4.5는 남은 토큰 예산을 실시간으로 추적하는 "컨텍스트 인식" 기능을 갖추고 있다.

### Claude 제공 Compact 종류

| 구분 | Claude Code (클라이언트) | API compact_20260112 (서버 베타) |
| --- | --- | --- |
| 위치 | CLI 내장 | Anthropic Messages API |
| 지원 모델 | 모든 모델 | Claude Opus 4.6만 |
| 트리거 | `/compact` 수동 또는 자동 | 입력 150K 토큰 초과 시 (기본값) |
| Hook 지원 | PreCompact, SessionStart 등 | 없음 |
| 이 문서 범위 | **해당** | 참고만 |

**이 문서는 Claude Code CLI의 클라이언트 측 compact만 다룬다.**

---

## 2. Compact 기본 동작과 한계

### 기본 동작

- `/compact` 수동 입력 또는 컨텍스트 한계 접근 시 자동 실행
- Claude가 전체 대화를 스스로 요약
- 요약 결과가 새로운 대화 시작점이 됨

### 기본 동작의 한계

- **요약 품질 불일관**: 어떤 정보를 보존할지 지침 없이 Claude가 자의적으로 판단
- **Git 상태 유실**: compact 시점의 브랜치, 변경 파일 등이 요약에서 누락될 수 있음
- **요약이 파일로 남지 않음**: compact 후 세션 종료 시 요약도 소멸
- **인수인계 불가**: 팀원에게 작업을 넘기려면 대화를 처음부터 설명해야 함

---

## 3. Hook으로 Compact 개선하기

### 전체 동작 흐름

```text
/compact 입력 또는 자동 트리거
  │
  ▼
[1] PreCompact hook — compact-guide.sh
      • 요약 지침 5개 항목을 Claude에 주입 (stdout → 요약 지침으로 사용됨)
      • 현재 Git 상태 (브랜치, 최근 커밋, 변경 파일) 주입
  │
  ▼
[2] Claude compact 수행
      • 주입된 지침에 따라 구조화된 요약 생성
      • 원래 요청, 최근 작업, 파일 목록, 미완료 작업, 결정 사항 포함
  │
  ▼
[3] SessionStart hook — write-handoff.sh (matcher: "compact")
      • compact 완료 후 새 세션 시작 시 Claude에게 HANDOFF.md 작성 지시
  │
  ▼
[4] 결과
      • 구조화된 요약이 새 대화 컨텍스트로 설정
      • <프로젝트 디렉토리>/HANDOFF.md 파일로 영구 보존
```

### settings.json 설정

`~/.claude/settings.json`의 `"hooks"` 섹션:

```json
{
  "hooks": {
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/compact-guide.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/write-handoff.sh"
          }
        ]
      }
    ]
  }
}
```

| 키 | 설명 |
| --- | --- |
| `PreCompact` | compact 직전에 실행. hook의 **stdout**이 Claude의 요약 지침으로 주입됨 |
| `SessionStart` + `matcher: "compact"` | compact 완료 후 새 세션이 시작될 때만 실행 (일반 세션 시작과 구분) |
| `type: "command"` | 셸 명령어 실행 방식 |

### compact-guide.sh — PreCompact hook

```bash
#!/bin/bash
# PreCompact hook: compact 요약 지침과 Git 상태를 Claude에 주입한다.
# stdout 출력이 compact 과정에서 Claude의 요약 지침으로 사용된다.

CWD=$(pwd)

# Compact 요약 지침
cat <<'EOF'
## [COMPACT 요약 지침]

아래 항목을 compact 요약에 **반드시** 포함하세요:

1. **사용자의 원래 요청/질문** - 정확하게 보존
2. **가장 최근 수행한 작업과 결과** - 구체적인 파일 경로, 명령어 결과 포함
3. **수정/생성/삭제한 파일 목록** - 경로 포함
4. **미완료 작업** - 진행 중이거나 남은 작업 목록
5. **중요한 결정/판단 사항** - 특정 방향을 선택한 이유

EOF

# Git 상태 출력
if git -C "$CWD" rev-parse --is-inside-work-tree &>/dev/null; then
  echo "## Git State (compact 시점)"
  echo ""
  echo "브랜치: $(git -C "$CWD" branch --show-current 2>/dev/null)"
  echo ""
  echo "최근 커밋:"
  git -C "$CWD" log --oneline -5 2>/dev/null | sed 's/^/  /'
  echo ""

  STATUS=$(git -C "$CWD" status --short 2>/dev/null)
  if [ -n "$STATUS" ]; then
    echo "변경 파일:"
    TOTAL=$(echo "$STATUS" | wc -l | tr -d ' ')
    echo "$STATUS" | head -20 | sed 's/^/  /'
    if [ "$TOTAL" -gt 20 ]; then
      echo "  ... 외 $((TOTAL - 20))개"
    fi
  fi
fi

exit 0
```

**핵심 포인트**

- `cat <<'EOF' ... EOF`: Claude에게 전달할 요약 지침 5개 항목. compact 결과물의 품질을 결정하는 핵심
- `git log --oneline -5`: 최근 커밋 이력으로 작업 맥락 보존
- `git status --short`: 변경/미커밋 파일 목록으로 작업 상태 보존
- Git 저장소가 아닌 디렉토리에서는 Git 섹션이 자동으로 생략됨

### write-handoff.sh — SessionStart hook

```bash
#!/bin/bash
# SessionStart (compact) hook: compact 완료 후 Claude에게 HANDOFF.md 작성을 지시한다.
# stdout 출력이 Claude의 새 세션 시작 시 컨텍스트로 주입된다.

CWD=$(pwd)

cat <<EOF
## [Compact 완료 - HANDOFF.md 작성 요청]

compact가 완료되었습니다.
위의 compact 요약 내용을 \`$CWD/HANDOFF.md\` 파일로 저장해주세요.

저장 후 사용자 응답을 기다려주세요.
EOF

exit 0
```

**핵심 포인트**

- compact 후 새 세션이 시작되면 이 스크립트의 stdout이 Claude에게 첫 번째 지시로 전달됨
- `$CWD`: compact가 실행된 프로젝트 디렉토리에 HANDOFF.md를 생성하도록 경로 지정
- Claude가 HANDOFF.md를 작성한 후 사용자 응답을 기다리므로, 세션이 자동으로 이어짐

---

## 4. 적용 방법 — 빠른 시작

### Step 1. 디렉토리 생성

```bash
mkdir -p ~/.claude/hooks
```

### Step 2. compact-guide.sh 생성

`~/.claude/hooks/compact-guide.sh` 경로에 저장

```bash
chmod +x ~/.claude/hooks/compact-guide.sh
```

### Step 3. write-handoff.sh 생성

`~/.claude/hooks/write-handoff.sh` 경로에 저장

```bash
chmod +x ~/.claude/hooks/write-handoff.sh
```

### Step 4. settings.json에 hooks 섹션 추가

`~/.claude/settings.json`을 열고 `"hooks"` 키를 추가한다.

```json
{
  "hooks": {
    "PreCompact": [
      {
        "hooks": [
          { "type": "command", "command": "~/.claude/hooks/compact-guide.sh" }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          { "type": "command", "command": "~/.claude/hooks/write-handoff.sh" }
        ]
      }
    ]
  }
}
```

### Step 5. .gitignore에 추가 (권장)

프로젝트 루트의 `.gitignore`에 추가:

```text
HANDOFF.md
```

## 예시

### compact 진행시

**Compact Handoff**

#### 사용자의 원래 요청

- compact 완료 후 HANDOFF.md가 자동 생성되는 흐름 확인 ("대화하면 생긴다는거지?")
- compact hooks 스크립트에서 python3 의존성이 정말 필요한지 재검토 ("python3 꼭 필요한지 다시 한 번 검토해줄래?")

#### 가장 최근 수행한 작업과 결과

##### python3 의존성 제거

`compact-guide.sh`와 `write-handoff.sh` 모두에서 stdin JSON 파싱용 python3 호출을 제거했다.

**근거:** Claude Code 공식 문서 확인 결과 "Handlers run in the current directory with Claude Code's environment" → hook 프로세스의 `pwd` == JSON의 `cwd` 필드 == 프로젝트 디렉토리. python3 파싱이 완전히 불필요.

기존 코드에도 `pwd` fallback이 이미 있었으므로 python3 파싱 자체가 중복이었음.

#### 수정/생성/삭제한 파일 목록

##### 수정

- `~/.claude/hooks/compact-guide.sh`
  - `INPUT=$(cat)` + python3 파싱 + fallback 분기 제거 → `CWD=$(pwd)` 1줄로 교체
- `~/.claude/hooks/write-handoff.sh`
  - 동일하게 python3 파싱 → `CWD=$(pwd)` 교체
- `~/.claude/hooks/COMPACT_HOOKS_GUIDE.md`
  - "python3 1회 (CWD 추출만)" → "없음" 으로 수정

##### 생성

- `<프로젝트 디렉토리>/HANDOFF.md` - compact 요약 저장

#### 미완료 작업

없음. 모든 작업 완료.

#### 중요한 결정/판단 사항

##### python3 제거 → `CWD=$(pwd)` 단일 라인

기존 코드:

```bash
INPUT=$(cat)
CWD=$(echo "$INPUT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('cwd',''))" 2>/dev/null)
if [ -z "$CWD" ]; then
  CWD=$(pwd)
fi
```

변경 후:

```bash
CWD=$(pwd)
```

#### Git 상태 (compact 시점)

- 브랜치: alpha
- 최근 커밋: `16f6dac` bot 리뷰 반영: 스타일 가이드 적용
- 변경 파일: `.claude/` (untracked), `HANDOFF.md` (untracked)

## Reference

- [Claude Code 공식 문서: hooks](https://docs.anthropic.com/ko/docs/claude-code/hooks)
- [Claude 플랫폼 문서: 컨텍스트 윈도우](https://platform.claude.com/docs/ko/build-with-claude/context-windows)
- [Claude 플랫폼 문서: 압축(Compaction)](https://platform.claude.com/docs/ko/build-with-claude/compaction)
