# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 저장소 성격

이 디렉터리는 코드 프로젝트가 아니라 **논문 집필 작업 공간**입니다. 목표는 `claude-code-2.1.88/`에서 추출한 Claude Code 내부 구조를 분석한 결과를, `example_paper/`에 들어있는 국내학술대회 논문 양식에 맞춰 학술 논문 형태로 작성하는 것입니다.

```
/home/asus/paper/
├── CLAUDE.md                  ← 이 파일 (작업 공간 가이드)
├── claude-code-2.1.88/        ← 분석 대상: 추출된 Claude Code 소스
│   ├── CLAUDE.md              ← 코드 구조에 대한 상세 분석 (한국어, 90KB+).
│   │                            논문 본문의 1차 소스. 임의로 수정 금지.
│   ├── README.md              ← 소스 추출 방법 및 npm 패키지 메타정보
│   ├── package/cli.js(.map)   ← 13MB 번들 + 57MB 소스맵 (실행되는 실체)
│   └── source/src/            ← 소스맵에서 복원한 1,906개 .ts/.tsx (읽기용)
├── example_paper/             ← 논문 양식 및 예시 (PDF만 들어있음)
│   ├── 논문양식(국내학술대회용).pdf   ← 따라야 할 템플릿
│   └── (예시 논문 4편)                ← 분야별 톤·구조 참고용
└── map_parsing.ipynb          ← cli.js.map → source/ 추출 스크립트 (실행 완료)
```

## 작업 흐름

논문 작성은 **두 단계의 독립적 자료**를 합치는 일입니다:

1. **내용(WHAT)** — `claude-code-2.1.88/CLAUDE.md`. 진입점, QueryEngine, Tool/Agent/Skill 시스템, MCP, 컴팩션, CCR/KAIROS 등 분산 모드, 시퀀스 다이어그램 8개가 이미 정리되어 있습니다. 새 분석을 만들기 전에 항상 먼저 이 파일을 읽어 중복을 피하세요. 더 깊이 들어가야 할 때만 `source/src/` 트리로 내려갑니다.
2. **형식(HOW)** — `example_paper/`의 PDF. 파일을 직접 읽으려면 `Read` 도구의 `pages` 파라미터를 사용하세요(PDF 10페이지 이상이면 필수). 양식 PDF에는 섹션 구성·서식 규칙이, 예시 4편에는 학술적 어조·인용·도식 사용 패턴이 들어 있습니다.

원천 코드와 양식 PDF 모두 한국어 기반입니다. 응답·작성 결과물 또한 한국어로 산출합니다 (사용자 지침 및 양식 일치 목적).

## 자주 쓰는 명령

이 작업 공간 자체에 빌드·테스트·린트 시스템은 없습니다. 의미 있는 명령은 보조 도구들입니다:

```sh
# 분석 대상 번들을 직접 실행해 동작을 확인 (Node.js >= 18)
node claude-code-2.1.88/package/cli.js --version
node claude-code-2.1.88/package/cli.js --help
node claude-code-2.1.88/package/cli.js --dump-system-prompt   # 시스템 프롬프트 덤프

# 소스맵 재추출이 필요한 경우 (이미 실행 완료, 보통 불필요)
jupyter nbconvert --to notebook --execute map_parsing.ipynb
```

PDF 양식을 읽을 때 페이지 범위 지정 예: `Read(file_path="...양식.pdf", pages="1-5")`.

## 분석된 코드의 핵심 사실 (논문에서 자주 인용할 지점)

`claude-code-2.1.88/CLAUDE.md`의 요지를 빠르게 떠올리기 위한 색인입니다. 세부는 그 파일을 직접 참조하세요.

- **번들 구조.** `package/cli.js`는 Bun으로 번들된 자가 완결형 Node 실행 파일이며, `source/src/`는 소스맵에서 복원한 읽기 전용 사본 — 재빌드 불가능 (`bun:bundle`의 `feature(...)` 컴파일 타임 API, 공개되지 않은 빌드 설정, 2,850개의 번들된 의존성 때문).
- **DCE 패턴.** `feature('KAIROS')`, `feature('DAEMON')` 등 약 89개 피처 플래그와 `USER_TYPE === 'ant'` 분기 약 294군데가 빌드 변형(외부 vs Anthropic 내부)을 가른다. 이 분기들은 `import` 대신 `require()`로 작성되어 데드 코드 제거에 의존한다.
- **턴 루프.** `QueryEngine` (대화당 1) → `query()` 제너레이터 (턴 루프) → Anthropic Messages API. REPL/headless/SDK 세 경로가 같은 `query()` 위에 올라간다.
- **도구 카탈로그.** Bash, FileEdit/Read/Write, Glob, Grep, Agent, Skill, Task*, Plan*, Worktree*, WebFetch/Search, MCP, LSP 등 30개 이상이 게이트별로 등록된다.
- **분산 모드.** CCR(원격), Bridge, Direct Connect, Coordinator, KAIROS, Swarm/Teammate, Daemon — 각각 게이트·통신 채널·핵심 모듈이 다르다.
- **컴팩션 전략.** autoCompact / microCompact (api·cached) / reactiveCompact / snipCompact / contextCollapse / timeBasedMCConfig.
- **8개 시퀀스 다이어그램.** 시작 시퀀스, 사용자 턴, 권한 결정, AgentTool, MCP+OAuth, 백그라운드 Bash, 자동 컴팩션, 슬래시 커맨드 디스패치.

## 논문 작성 시 유의사항

- 하위 `claude-code-2.1.88/CLAUDE.md`는 사실상 1차 분석 결과물이다. 표·다이어그램을 논문에 옮길 때는 이 파일을 출처로 삼되, 외부 빌드에서 실제로 도달 가능한 코드 경로만 인용하라(예: KAIROS·DAEMON 같은 내부 전용 게이트는 외부 빌드에 미포함이라는 점을 명시).
- 라인 인용이 필요할 때는 `source/src/<path>:<line>` 형식을 쓴다. 실행되는 실체는 `package/cli.js`라는 점, 그리고 `source/`는 소스맵 복원본이라는 점을 본문에 한 번은 분명히 적어둘 것 — 독자가 GitHub의 비공개 소스를 본 줄 오해할 수 있다.
- `example_paper/`의 예시 4편은 모두 응용 AI 분야(부동산·음악·패션)이므로, 어조·분량·도식 밀도는 참고하되 주제·방법론을 그대로 가져오지 말 것.
- 양식 PDF에 들어있는 구체 규칙(여백·폰트·참고문헌 스타일 등)은 작성 시점에 PDF를 직접 열어 확인. 이 파일에 단정해서 적어두지 않는다 — 양식이 업데이트되면 빠르게 어긋난다.
