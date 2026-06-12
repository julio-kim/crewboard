# Crewboard

> **A PM agent that runs your project board — with a crew of specialized subagents doing the work.**
>
> Claude Code 위에서 동작하는 멀티 에이전트 프로젝트 수행 체계.
> PM 에이전트가 GitHub Projects 또는 GitLab 이슈 보드를 프로젝트 현황의 기준점(SSOT)으로 삼아 오케스트레이션하고,
> 기획 → 설계 → 구현 → 테스트 전 단계를 전문화된 서브 에이전트가 수행합니다.

<p align="left">
  <img src="https://img.shields.io/badge/Claude%20Code-Subagents%20%2B%20Skills-d97757?logo=anthropic&logoColor=white" alt="Claude Code" />
  <img src="https://img.shields.io/badge/version-1.4-blue" alt="Version" />
  <img src="https://img.shields.io/badge/platform-GitHub%20%7C%20GitLab-181717?logo=github&logoColor=white" alt="Platform" />
  <img src="https://img.shields.io/badge/license-MIT-green" alt="License" />
  <img src="https://img.shields.io/badge/PRs-welcome-brightgreen" alt="PRs Welcome" />
</p>

---

## 💡 어떻게 시작하나

Crewboard는 설치할 패키지가 없습니다. 설계 문서 한 장([CREWBOARD.md](CREWBOARD.md))을
새 프로젝트 폴더에 복사하고 Claude Code에게 부트스트랩을 지시하면, PM 에이전트와
서브 에이전트 크루, 보드 연동, 가드레일까지 프로젝트 수행 체계 전체가 구성됩니다.

```text
CREWBOARD.md 복사  →  claude 실행  →  "부트스트랩 프로토콜대로 구성해"  →  /kickoff
```

이후에는 PM 에이전트가 이슈 보드를 기준점 삼아 기획 → 설계 → 구현 → 테스트를
오케스트레이션하고, 사람은 요구사항 작성과 6개의 승인 게이트에서만 개입합니다.
단계별 절차는 [시작하기](#-시작하기)를 참고하세요.

## 🏗️ 아키텍처

```mermaid
flowchart TB
    H["👤 Human<br/>목표 제시 · 게이트 승인 · 방향 결정"]
    PM["🎯 PM Agent (Claude Code 메인 세션)<br/>오케스트레이션 · 검증 게이트 · 상태 전이 전권"]
    subgraph CREW["서브 에이전트 크루"]
        P[planner<br/>요구사항 정제]
        A[architect<br/>설계]
        D[developer<br/>구현]
        T[tester<br/>테스트]
        R[reviewer<br/>독립 검증]
    end
    B[("📋 Project Board / Issues<br/>프로젝트 기준점 (SSOT)")]

    H <-->|승인 게이트| PM
    PM -->|디스패치 + AC| CREW
    CREW -->|표준 REPORT 요약| PM
    PM <-->|상태 전이 · 증거 게시| B
    CREW -.->|산출물 · 코멘트| B
```

핵심 제약을 그대로 설계에 반영했습니다: Claude Code의 서브에이전트는 다른
서브에이전트를 스폰할 수 없으므로, **PM은 메인 세션 그 자체**입니다.
서브에이전트는 요약만 보고하고 상세 산출물은 파일과 이슈 코멘트로 외부화하여
PM의 컨텍스트를 보호합니다.

## 🧭 설계 원칙

Crewboard의 모든 구성은 다섯 가지 원칙 위에 서 있습니다.

- **요구사항은 받는 것, 발명하는 것이 아니다** — 사람이 작성하는 인테이크 템플릿이 모든 요구사항의 원천이며, 에이전트는 정제·구조화·검증만 합니다
- **보드가 곧 공식 기록이다** — 모든 상태·산출물·결정이 GitHub Projects/GitLab 이슈 보드에 남아, 세션이 끊겨도 보드만 읽으면 어디서든 재개됩니다
- **검증 없이는 Done이 없다** — 만든 에이전트와 검증하는 에이전트를 분리하고(builder–validator), 수용 기준(AC) 대조를 통과해야만 상태가 전이됩니다
- **규약이 아니라 권한으로 막는다** — `git push`, 머지, 보드 조작은 프롬프트 지시가 아니라 권한 설정과 훅으로 차단합니다
- **프로젝트가 끝나면 팀이 똑똑해진다** — 회고에서 검증된 해법을 스킬로 적재해, 다음 프로젝트의 에이전트가 같은 문제를 다시 풀지 않습니다

## ✨ 주요 구성 요소

| 구성 | 내용 |
|------|------|
| **에이전트 6종** | planner / architect / developer / tester / reviewer / doc-writer — 절차·산출물 예시·안티패턴·에스컬레이션 조건을 갖춘 7요소 표준 정의 |
| **프로젝트 프로파일** | 스택·규약·환경 제약은 에이전트 정의가 아닌 프로파일에 — 같은 `.claude/` 세트를 어떤 스택의 프로젝트에도 재사용. 킥오프 때 프로파일에 맞는 스택 스킬을 확정·자동 생성 |
| **요구사항 인테이크** | 템플릿 발급 → 사람 작성 → RFI 질문 루프 → 베이스라인 → 마일스톤 분할. R-ID 부터 코드까지 끊기지 않는 추적성 |
| **슬래시 커맨드 7종** | `/kickoff` `/intake` `/plan-sprint` `/run` `/verify` `/status` `/retro` |
| **검증 게이트** | reviewer의 근거 인용 판정 + tester의 독립 시나리오 + PM의 기계적 대조. 재작업 3회 시 자동 에스컬레이션 |
| **가드레일** | push/merge 권한 차단, 보드 조작 훅 통제, CODEOWNERS로 `.claude/` 변경은 사람 리뷰 강제 |
| **자기개선 루프** | `/retro` 가 검증된 해법을 `skills/learned/` 에 적재 — [agentskills.io](https://agentskills.io) 호환 |

## 🚀 시작하기

### 요구 사항

- [Claude Code](https://code.claude.com/docs) 설치 및 인증
- `gh` CLI (GitHub) 또는 `glab` CLI (GitLab) — 자가 호스팅(GHE / self-managed) 지원

### 부트스트랩 (3단계)

```bash
# 1. 새 프로젝트 폴더에 가이드 문서를 복사
mkdir my-project && cd my-project
cp /path/to/CREWBOARD.md .

# 2. Claude Code 실행
claude
```

```text
# 3. Claude 에게 지시
> 이 문서의 부트스트랩 프로토콜대로 프로젝트를 구성해
```

Claude가 환경 점검 → 질문(플랫폼·리포지토리·보드) → 26개 항목 생성 →
보드/라벨 구성 → 검증 체크리스트 보고까지 자동으로 수행합니다.
구성이 끝나면 새 세션에서 프로젝트를 시작합니다:

```text
> /kickoff 사내 교육 신청 관리 시스템 — 신청/승인/이수 관리
```

### 진행 흐름

```text
/kickoff   프로파일 인터뷰 → 스택 스킬 확정·생성 → 헌장 승인 → 인테이크 템플릿 발급
   ⏸️      (사람이 루트의 INTAKE 파일 작성 — 비동기, 현업 협의 가능. 끝나면 /intake)
/intake    회수·박제 → planner 정제 → RFI 질문 루프 → 베이스라인 → 마일스톤 분할 → 백로그 생성
/run       이슈 단위 자율 사이클: 구현 → 리뷰 → 테스트 → 검증 게이트 → Done
/status    현황 보고 (보드에 박제)
/retro     회고 + learned 스킬 적재
```

사람은 6개의 게이트에서만 개입합니다: 프로파일 · 헌장 · 베이스라인 ·
마일스톤 분할 · 아키텍처 · 릴리스.

## 📁 생성되는 구조

```text
project-root/
├── CLAUDE.md                      # PM 정체성 + 절대 규칙
├── .claude/
│   ├── agents/                    # 서브 에이전트 6종
│   ├── skills/                    # pm-orchestration · platform-ops · learned/
│   ├── commands/                  # 슬래시 커맨드 7종
│   ├── hooks/                     # 보드 조작 가드레일
│   └── settings.json              # 권한 (push/merge 차단)
├── .github/  또는  .gitlab/       # 이슈 4종 + PR/MR 템플릿
├── CONTRIBUTING.md · CODEOWNERS
└── docs/
    ├── 00-charter/                # 헌장 + 프로젝트 프로파일
    ├── 10-requirements/intake/    # 요구사항 인테이크 (불변 원천)
    ├── 20-design/ · 30-test/ · 90-retro/
    └── GUIDE.md                   # 전체 설계서 (이 체계의 운영 매뉴얼)
```

## 🗺️ 로드맵

- [x] v1.x — 서브에이전트 기반 단일 세션 오케스트레이션
- [ ] 정형화된 반복 구간의 [Dynamic Workflows](https://code.claude.com/docs/en/workflows) 전환 (릴리스 QA 사이클 등)
- [ ] 병렬 병목 구간의 [Agent Teams](https://code.claude.com/docs/en/agent-teams) 적용 검토
- [ ] 조직 공유 스택 스킬 팩 라이브러리 — 킥오프의 스킬 생성 단계가 참조할 재사용 풀
- [ ] 부트스트랩의 스크립트화 (판단 불필요 구간)

## 🤝 기여

이슈와 PR을 환영합니다. 이 리포지토리 자체도 Crewboard 규약을 따릅니다 —
[CONTRIBUTING.md](CONTRIBUTING.md)를 먼저 읽어주세요.
특히 파일럿 운영에서 발견한 에이전트 안티패턴 제보가 가장 가치 있는 기여입니다.

## 📄 License

[MIT](LICENSE)

---

<p align="center">
  <sub>Built with Claude Code · 설계 문서 전문은 <a href="CREWBOARD.md">CREWBOARD.md</a></sub>
</p>
