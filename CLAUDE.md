# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 개요

이 저장소는 Claude Code 설정(에이전트, 스킬)을 공유하기 위한 템플릿 레포지토리입니다. 빌드/테스트/린트 명령어가 없으며, 모든 파일은 Markdown 형식입니다.


## 저장소 구조

```
.claude/
  agents/          # Claude Code 서브 에이전트 정의
    code-reviewer.md
    prd-to-roadmap.md
    sprint-close.md
    sprint-planner.md
  skills/          # Claude Code 스킬 정의
    karpathy-guidelines/
    writing-plans/
docs/
  PRD.md           # 제품 요구사항 문서 (사용하는 프로젝트에서 생성)
  ROADMAP.md       # 프로젝트 로드맵 (prd-to-roadmap 에이전트가 생성)
  plans/           # 구현 계획 문서 (YYYY-MM-DD-<feature-name>.md)
  sprint/          # 스프린트 문서 및 검증 보고서
    sprint{N}.md
    sprint{N}/     # 스크린샷, Playwright 보고서
README.md          # 저장소 소개 및 사용 방법
CLAUDE.md          # Claude Code 설정 파일
```

## 에이전트 파일 형식 (`.claude/agents/*.md`)

각 에이전트 파일은 YAML frontmatter로 시작합니다:

```yaml
---
name: agent-name
description: 에이전트 설명
model: inherit | opus | sonnet | haiku
color: red | blue | green | ...
memory: project   # 프로젝트 메모리 자동 주입
---
```

**중요:** 에이전트 파일에 절대 경로(`/Users/...`)를 하드코딩하지 않습니다. `memory: project`가 런타임에 올바른 경로를 자동 주입합니다.

## 스킬 파일 형식 (`.claude/skills/<name>/SKILL.md`)

```yaml
---
name: skill-name
description: 스킬 설명
---
```

## 스프린트 워크플로우

1. **prd-to-roadmap** 에이전트: `docs/PRD.md` → `docs/ROADMAP.md` 생성
2. **sprint-planner** 에이전트: ROADMAP 기반으로 `docs/sprint/sprint{N}.md` 생성
3. 구현 (writing-plans 스킬로 세부 계획 수립 → 실행)
4. **sprint-close** 에이전트: ROADMAP 업데이트 → PR 생성 → 코드 리뷰 → Playwright 검증 → 검증 보고서 저장

## 핵심 에이전트 역할

| 에이전트 | 역할 | 주요 입력 | 주요 출력 |
|----------|------|-----------|-----------|
| `prd-to-roadmap` | PRD → 로드맵 변환 | `docs/PRD.md` | `docs/ROADMAP.md` |
| `sprint-planner` | 스프린트 계획 수립 | `ROADMAP.md` | `docs/sprint/sprint{N}.md` |
| `sprint-close` | 스프린트 마무리 | 현재 브랜치 | PR, 검증 보고서 |
| `code-reviewer` | 코드 리뷰 | 구현 완료 단계 | 이슈 분류 보고 (Critical/Important/Suggestion) |

## Playwright MCP 검증

`sprint-close` 및 `prd-to-roadmap` 에이전트는 Playwright MCP 도구(`browser_navigate`, `browser_snapshot`, `browser_click`, `browser_console_messages`, `browser_network_requests` 등)를 사용하여 `npm run dev` 실행 상태에서 UI를 직접 검증합니다. 검증 결과는 `docs/sprint/sprint{N}/playwright-report.md`에 저장합니다.

## 언어 및 커뮤니케이션 규칙

- 기본 응답 언어: 한국어
- 코드 주석: 한국어로 작성
- 커밋 메시지: 한국어로 작성
- 문서화: 한국어로 작성
- 변수명/함수명: 영어 (코드 표준 준수)

## 개발시 유의해야할 사항

- sprint 관련 문서 구조:
  - 스프린트 계획/완료 문서: `docs/sprint/sprint{n}.md`
  - 스프린트 첨부 파일 (스크린샷, 보고서 등): `docs/sprint/sprint{n}/`
- sprint 개발이 plan 모드로 진행될 때는 다음을 꼭 준수합니다.
  - karpathy-guidelines skill을 준수하세요.
  - sprint 가 새로 시작될 때는 새로 branch를 sprint{n} 이름으로 생성하고 해당 브랜치에서 작업해주세요. (worktree 사용하지 말아주세요)
  - 다음과 같이 agent를 활용합니다.
    1. sprint-planner agent가 계획 수립 작업을 수행하도록 해주세요.
    2. 구현/검증 단계에서는 각 task의 내용에 따라 적절한 agent가 있는지 확인 한 후 적극 활용해주세요.
    3. 스프린트 구현이 완료되면 sprint-close agent를 사용하여 마무리 작업(ROADMAP 업데이트, PR 생성, 코드 리뷰, 자동 검증)을 수행해주세요.

- 스프린트 검증 원칙 — **자동화 가능한 항목은 sprint-close 시점에 직접 실행**:
  - ✅ **자동 실행**: `docker compose exec backend pytest -v` — 백엔드 통합 테스트
  - ✅ **자동 실행**: API 동작 검증 (curl/httpx) — Docker 컨테이너가 실행 중인 경우 sprint-close agent가 직접 실행
  - ✅ **자동 실행**: 데모 모드 API 검증 — 마찬가지로 서버 실행 중이면 자동 실행
  - ❌ **수동 필요**: `docker compose up --build` — 새 코드 반영을 위한 Docker 재빌드 (타이밍을 사용자가 결정)
  - ❌ **수동 필요**: `alembic upgrade head` — prod DB 스키마 변경 (되돌릴 수 없으므로 사용자가 직접 실행)
  - ❌ **수동 필요**: 브라우저 UI 시각적 확인 (프론트엔드 렌더링, 버튼 동작 등)
  - sprint-close agent는 자동 실행 항목을 실행하고 결과를 deploy.md에 기록해야 합니다.
  - deploy.md에는 "자동 검증 완료" 항목과 "수동 검증 필요" 항목을 명확히 구분하여 기재합니다.

- 사용자가 직접 수행해야 하는 작업은 deploy.md 파일을 생성하거나 기존에 존재하는 deploy.md에 수행해야하는 작업을 자세히 정리해주세요.
- 체크리스트 작성 형식:
  - 완료 항목: `- ✅ 항목 내용`
  - 미완료 항목: `- ⬜ 항목 내용`
  - GFM `[x]`/`[ ]` 대신 이모지를 사용하여 마크다운 미리보기에서 시각적 구분을 보장합니다.
