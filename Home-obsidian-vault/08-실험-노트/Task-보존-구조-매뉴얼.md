# Task 보존 구조 매뉴얼

> Claude Code 하네스에서 세션 종료·재시작·컨텍스트 초기화 시
> 작업 상태를 잃지 않고 이어가는 3-레이어 구조.
> 다른 프로젝트에서도 동일하게 설정 가능.

---

## 전체 구조 한눈에 보기

```
상황                        레이어                저장 위치
────────────────────────────────────────────────────────────
복잡한 작업 중 각 단계 완료  → Layer 1 Checkpoint  tasks/active/{id}.json
/clear 또는 /restart 수신   → Layer 2 Snapshot    ~/.claude/task_YYYY-MM-DD.md
Claude 세션 자체 종료        → Layer 3 Pipeline    Obsidian vault / JSONL
```

세 레이어는 독립적으로 작동하며, 각각 다른 상황에서 복구 기반이 된다.

---

## Layer 1 — Checkpoint (tasks/active/*.json)

### 언제 작동하나
복잡한 작업(파일 생성·수정·다단계 구현) 시 하네스 엔지니어링 5-역할 절차 내에서 자동 생성.

### 흐름
```
플래너 역할
  → tasks/active/{YYYYMMDD-HHmm-슬러그}.json 생성
  → 모든 step: pending, status: in_progress

생성자 역할 (각 step 완료 시)
  → step: done, 다음 step: in_progress
  → updated_at, changed_files, last_step_output 갱신

평가자 역할 (전체 완료 시)
  → tasks/active/ → tasks/completed/ 이동
  → completed/ 10개 초과 시 오래된 것 삭제

오류 발생 시
  → step: failed, status: failed
  → tasks/active/ 에 유지 (삭제 안 함)
```

### JSON 스키마
```json
{
  "id": "20260421-1430-context-snapshot-build",
  "title": "context-snapshot 스킬 생성",
  "created_at": "2026-04-21T14:30:00",
  "updated_at": "2026-04-21T14:45:00",
  "status": "in_progress",
  "trigger_message": "context-snapshot 스킬 만들어줘",
  "chat_id": "689642569",
  "plan": {
    "steps_total": 3,
    "steps": [
      {"n": 1, "desc": "스킬 파일 생성", "status": "done"},
      {"n": 2, "desc": "CLAUDE.md 트리거 수정", "status": "in_progress"},
      {"n": 3, "desc": "핵심 경로 테이블 추가", "status": "pending"}
    ]
  },
  "context": {
    "changed_files": ["skills/context-snapshot/SKILL.md"],
    "last_step_output": "SKILL.md 생성 완료",
    "notes": ""
  }
}
```

### 복구 방법
STARTUP 트리거(`[STARTUP-TRIGGER]`) 수신 시 `tasks/active/` 자동 감지 → 텔레그램 알림 → "이어서 진행해줘" 응답 시 재개.

---

## Layer 2 — Context Snapshot (task_YYYY-MM-DD.md)

### 언제 작동하나
`/clear` 또는 `/restart` 수신 시, reply 전에 자동 실행.

### 흐름
```
/clear or /restart 수신
  → skills/context-snapshot/SKILL.md 실행
  → 오늘 날짜로 파일명 결정: ~/.claude/task_YYYY-MM-DD.md
  → tasks/active/ 활성 작업 수집
  → 현재 세션 작업 1~3줄 요약
  → 파일 없음: 새로 생성 (헤더 + 섹션)
  → 파일 있음: 섹션만 추가 (덮어쓰기 금지)
  → reply: "💾 컨텍스트 저장됨: task_YYYY-MM-DD.md"
  → 원래 재시작 진행
```

### 파일 형식
```markdown
# 작업 이어가기 — 2026-04-21

---

## 14:30 — /clear 스냅샷

> 트리거: /clear
> 프로젝트: D:/project/claudeclaw-setup-telegram

### 활성 작업
- 20260421-1430-context-snapshot-build (2/3단계 완료)

### 이번 세션 컨텍스트
context-snapshot 스킬 생성 및 CLAUDE.md 트리거 연동 작업 진행 중.
SKILL.md 생성 완료, CLAUDE.md 수정 완료, 핵심 경로 테이블 추가 완료.

---

## 16:00 — /restart 스냅샷
...
```

### 복구 방법
새 세션에서 `~/.claude/task_오늘날짜.md` 파일을 열어 마지막 섹션부터 이어서 진행.

---

## Layer 3 — Knowledge Pipeline (세션 종료 시)

### 언제 작동하나
Claude Code 세션이 종료(stop)될 때 전역 stop hook 자동 실행.

### 흐름
```
Claude 세션 종료
  → stop-hook.ps1 (전역 hooks 등록)
  → stdin에서 session_id 추출
  → ~/.claude/projects/*/{session_id}.jsonl 탐색
  → 첫 번째 user 메시지 추출
  → POST http://127.0.0.1:3100/ingest
  → orchestrator: analyze → structure → memory
  → Obsidian vault 저장:
      learning/  → 질문·참조
      success/   → 일반 지식
      failure/   → 무시·에스컬레이션
```

### 복구 방법
Obsidian vault에서 날짜·태그로 검색. 세션 원본은 `LLM-Wiki/raw/YYYY-MM-DD/` 에 보존.

---

## 다른 프로젝트에 설정하기

### 최소 설정 (Layer 1 + 2만)

#### 1단계: 디렉터리 생성
```powershell
mkdir tasks/active
mkdir tasks/completed
```

#### 2단계: 스킬 파일 복사
```
skills/checkpoint/SKILL.md      ← Layer 1
skills/context-snapshot/SKILL.md ← Layer 2
skills/startup/SKILL.md          ← STARTUP 복구
skills/preflight/SKILL.md        ← 위험 작업 감지 (선택)
```

원본 위치: `D:/project/claudeclaw-setup-telegram/skills/`

#### 3단계: 프로젝트 CLAUDE.md에 추가

```markdown
## 트리거 처리

| 트리거 | 동작 |
|--------|------|
| `[STARTUP-TRIGGER]` | `skills/startup/SKILL.md` 읽어서 즉시 실행 |
| 복잡한 작업 요청 | 하네스 엔지니어링 5-역할 절차 적용 |
| `/restart` | `skills/context-snapshot/SKILL.md` 실행 → reply → 재시작 |
| `/clear`   | `skills/context-snapshot/SKILL.md` 실행 → reply → 재시작 |

## 핵심 경로

| 용도 | 경로 |
|------|------|
| Checkpoint 스킬 | `skills/checkpoint/SKILL.md` |
| Context Snapshot 스킬 | `skills/context-snapshot/SKILL.md` |
| 컨텍스트 스냅샷 저장 위치 | `~/.claude/task_YYYY-MM-DD.md` |
| 작업 체크포인트 | `tasks/active/` / `tasks/completed/` |
```

#### 4단계: 하네스 엔지니어링 절차 선언

CLAUDE.md에 5-역할 정의 포함 (아래 최소 버전):

```markdown
## 하네스 엔지니어링

파일 생성·수정·삭제, 다단계 구현 시 적용:

1. 오케스트레이터 — edit_message 시작 알림
2. 플래너 — tasks/active/{id}.json 생성 + 단계 목록
3. 생성자 — Phase별 구현 + Checkpoint 갱신
4. 평가자 — 기준 대조 + tasks/completed/ 이동
5. QA — 실행 테스트 + 최종 reply
```

---

### 전체 설정 (Layer 3 포함)

Layer 3은 orchestrator 서비스와 Obsidian vault가 있어야 작동.

#### 추가 필요 구성
| 항목 | 설명 |
|------|------|
| `D:/project/orchestrator/` | Node.js orchestrator 서버 (port 3100) |
| `D:/project/runtime/hooks/stop-hook.ps1` | 세션 종료 훅 |
| Obsidian vault | `LLM-Wiki/raw/` 구조 |

#### 전역 settings.json에 stop hook 등록
```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "pwsh -File D:\\project\\runtime\\hooks\\stop-hook.ps1"
          }
        ]
      }
    ]
  }
}
```
위치: `C:\Users\{username}\.claude\settings.json`

---

## 레이어별 비교

| 항목 | Layer 1 Checkpoint | Layer 2 Snapshot | Layer 3 Pipeline |
|------|-------------------|-----------------|-----------------|
| 트리거 | 복잡한 작업 시작 | /clear /restart | Claude 세션 종료 |
| 저장 형식 | JSON | Markdown | Markdown + JSONL |
| 저장 위치 | 프로젝트/tasks/ | ~/.claude/ | Obsidian vault |
| 복구 방법 | STARTUP 자동 감지 | 파일 수동 참조 | vault 검색 |
| 설정 난이도 | ★★☆ | ★☆☆ | ★★★ |
| 필수 여부 | 복잡한 작업 시 필수 | 권장 | 선택 |

---

## 트러블슈팅

| 증상 | 원인 | 조치 |
|------|------|------|
| STARTUP에서 active 작업 미감지 | tasks/active/ 없음 | 디렉터리 수동 생성 |
| Snapshot 파일 미생성 | skills/context-snapshot 없음 | 스킬 파일 복사 |
| Snapshot 내용 비어있음 | 세션 요약 미작성 | SKILL.md 절차 재확인 |
| stop-hook 미작동 | settings.json 등록 누락 | 전역 hooks 섹션 추가 |
| orchestrator 연결 실패 | PM2 미실행 | pm2 start ecosystem.config.cjs |
