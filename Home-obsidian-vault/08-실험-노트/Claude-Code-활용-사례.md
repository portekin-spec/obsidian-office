# Claude Code 활용 사례

> 프로젝트 `claudeclaw-setup-telegram` 기반 실전 운영에서 검증된 패턴 모음.
> 날짜: 2026-04-21 | 운영자: khantechman

---

## 1. 텔레그램 채널 연동 Claude Code 하네스

### 개요
Claude Code CLI를 PM2로 상시 실행하고, Telegram Bot을 통해 모바일에서 명령을 보내면 Claude가 실제 코드 작업을 수행하는 구조.

### 핵심 구성
```
Telegram → plugin:telegram → Claude Code (--channels)
                ↓
          PM2 ecosystem.config.cjs
                ↓
          run-claude.ps1 → claude-session.ps1
```

### 실행 명령
```powershell
claude --channels "plugin:telegram@claude-plugins-official" --dangerously-skip-permissions
```

### 검증된 효과
- 모바일에서 실시간으로 코드 수정·배포 지시 가능
- PM2 자동 재시작으로 세션 안정성 확보
- `edit_message` 로 긴 작업 진행률 실시간 확인

---

## 2. 하네스 엔지니어링 5-역할 패턴

### 개요
복잡한 작업을 5개 역할로 분리해 품질을 높이는 Claude Code 워크플로우.

### 역할 구조
| 역할 | 역할 |
|------|------|
| 오케스트레이터 | 전체 조율 + edit_message 진행 업데이트 |
| 플래너 | Checkpoint JSON 생성 + Pre-flight 검증 |
| 생성자 | Phase별 구현 + Checkpoint 갱신 |
| 평가자 | 검증 기준 대조 + 완료 이동 |
| QA | 실제 실행 테스트 + 최종 reply |

### 적용 조건
- 파일 생성·수정·삭제 포함 작업
- 3단계 이상 다단계 구현
- `만들어줘` / `구현해줘` / `수정해줘` 류 요청

### 제외 조건 (직접 처리)
- 질문·조회 (읽기 전용)
- 단일 파일 1줄 수정
- 하트비트 / STARTUP 트리거

---

## 3. 음성 메시지 처리 파이프라인

### 개요
Telegram 음성 메시지 → Groq Whisper 전사 → Claude API → 자동 reply.

### 흐름
```
Telegram 음성 → download_attachment
  → bun voice-pipeline.mjs <파일> <chat_id>
  → Groq Whisper STT → 전사 원문
  → Claude API (슬라이딩 윈도우 20턴 히스토리)
  → Telegram reply (자동)
```

### 핵심 설계
- 전사 원문 손상 없음 — 그대로 user turn 처리
- 시스템 프롬프트에 메모리 주입 (Anthropic 캐시 최적화)
- 히스토리: `~/.claude/channels/telegram/voice-history.json`

### 환경변수
- `GROQ_API_KEY` → `~/.claude/channels/telegram/.env`

---

## 4. Knowledge Pipeline (세션 종료 → Obsidian 자동 저장)

### 개요
Claude 세션 종료 시 두 경로로 자동 저장.

### 흐름 1 — 세션 원본 (llm-wiki)
```
Claude Stop 훅 → save-session.ps1
  → JSONL 파싱 → Markdown
  → LLM-Wiki/raw/YYYY-MM-DD/NNN-title.md
```

### 흐름 2 — AI 분석 (orchestrator)
```
Claude Stop 훅 → stop-hook.ps1
  → POST http://127.0.0.1:3100/ingest
  → analyze_skill (Haiku) → structure_skill → memory_skill
  → LLM-Wiki/raw/{learning|success|failure}/YYYYMMDDHHMM.md
```

### Vault 분류
| 폴더 | 내용 |
|------|------|
| `raw/YYYY-MM-DD/` | 세션 원본 (불변) |
| `raw/learning/` | AI 분류: 질문·참조 |
| `raw/success/` | AI 분류: 일반 지식 |
| `raw/failure/` | AI 분류: 무시·에스컬레이션 |

---

## 5. Context Snapshot (세션 이어가기)

### 개요
`/clear` · `/restart` 직전에 현재 작업 상태를 파일로 보존.

### 동작
```
/clear 또는 /restart 수신
  → skills/context-snapshot/SKILL.md 실행
  → ~/.claude/task_YYYY-MM-DD.md 생성 (또는 섹션 추가)
  → 활성 작업 목록 + 세션 요약 저장
  → 원래 reply + restart 진행
```

### 파일 위치
`C:\Users\dhkim\.claude\task_YYYY-MM-DD.md`

---

## 6. 승인 게이트 (이중 승인)

### 개요
위험 작업(git push, pm2 stop, .env 수정 등) 자동 차단 + 텔레그램 승인 대기.

### 동작
```
위험 Bash 명령 감지 → approval-gate.ps1
  → 텔레그램 알림 + 터미널 차단
  ↓
사용자 "승인" 전송
  → Claude: approvals/approved.signal 파일 생성
  → 원래 작업 자동 재시도
```

### 승인 유효 시간
10분 (이후 만료 → 재승인 필요)

---

## 7. Dead Man's Switch (DMS) 생존 감시

### 개요
Claude 세션이 살아있는지 주기적으로 감시, 비정상 종료 시 텔레그램 알림.

### 구성
- **alive-writer**: 30초마다 `heartbeat/alive.txt` 갱신
- **dms-check.ps1**: 1분마다 alive.txt 타임스탬프 확인
- **임계값**: 5분 무응답 → 텔레그램 경보

### Task Scheduler 등록
```
ClaudeAliveWriter — alive.txt 갱신 (시스템 시작 시 자동)
ClaudeDMS        — 생존 감시 (시스템 시작 시 자동)
```

---

## 참고 링크
- 프로젝트 루트: `D:/project/claudeclaw-setup-telegram/`
- 스킬 디렉터리: `skills/`
- Obsidian Vault: `D:/PROJECT/Khan-odsidian-vault/Home-obsidian-vault/`
