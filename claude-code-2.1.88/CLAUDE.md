# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 저장소 성격

이 디렉터리는 일반적인 의미의 "프로젝트"가 아니라, npm에 배포된 `@anthropic-ai/claude-code@2.1.88` 패키지에서 **추출한 읽기 전용 소스 트리**입니다. `package/cli.js`(13MB의 자가 완결형 Node.js 번들) 및 `cli.js.map`(57MB 소스맵)에서 1,906개의 TypeScript/TSX 파일을 복원해 `source/src/`에 풀어놓은 형태입니다.

**중요한 결과들:**
- **빌드/재빌드 불가능.** 코드가 `import { feature } from 'bun:bundle'`(Bun 번들러 컴파일 타임 API)에 의존하고, 원본 `package.json`의 빌드 의존성·`tsconfig`·번들러 설정이 공개되지 않았으며, 2,850개의 번들된 `node_modules` 의존성이 소스맵 항목으로만 존재합니다. `npm install` / `bun build` / `tsc` 등은 실행되지 않습니다.
- **테스트 스위트 없음.** `package.json`의 유일한 `scripts` 항목은 `prepare`이며, 무단 배포를 막기 위한 가드일 뿐입니다(`AUTHORIZED` 환경 변수 미설정 시 에러로 종료). `npm test`, lint, formatter 모두 존재하지 않습니다.
- **추출된 소스는 "읽기·연구용"입니다.** 코드 변경을 가하더라도 실행되는 것은 `package/cli.js` 번들이지 `source/src/`의 파일이 아닙니다. 실행을 바꾸려면 번들 자체를 패치하거나 외부 도구(예: 부트스트랩 후 monkey-patch)를 사용해야 합니다.

## 실행 방법

번들은 Node.js >= 18에서 직접 실행됩니다:

```sh
node package/cli.js --version          # 2.1.88 (Claude Code) 출력
node package/cli.js --help             # 모든 옵션 출력
node package/cli.js -p "hello world"   # 비대화형 1회성 실행
node package/cli.js                    # 대화형 REPL
```

전역 설치 또는 심볼릭 링크:

```sh
npm install -g @anthropic-ai/claude-code@2.1.88
ln -s "$(pwd)/package/cli.js" /usr/local/bin/claude
```

내부 진단/디버깅용 진입점(코드를 읽다가 마주칠 수 있는 특수 경로):
- `node package/cli.js --dump-system-prompt [--model <id>]` — 렌더된 시스템 프롬프트를 stdout으로 출력 후 종료. `feature('DUMP_SYSTEM_PROMPT')` 게이트로 외부 빌드에서는 제거됨(`source/src/entrypoints/cli.tsx`).
- `node package/cli.js --claude-in-chrome-mcp` / `--chrome-native-host` — Chrome 확장 통합용 MCP 서버 / native messaging 호스트.
- `node package/cli.js --daemon-worker=<kind>` — `DAEMON` 피처가 켜진 빌드에서 슈퍼바이저가 스폰하는 워커 프로세스 모드.

## 시스템 레이아웃 (한눈에)

```
┌────────────────────────────────────────────────────────────────────┐
│  사용자 환경                                                       │
│   $ claude / $ node package/cli.js                                 │
└──────────────┬─────────────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────────────────────────┐
│  package/cli.js  (13MB 자가 완결형 Node.js 번들)                   │
│   • bun build 결과물, 외부 의존성 0개 (sharp만 optional)           │
│   • source/src/ 는 cli.js.map에서 추출한 "읽기 전용" 사본          │
└──────────────┬─────────────────────────────────────────────────────┘
               │ entrypoint
               ▼
┌────────────────────────────────────────────────────────────────────┐
│  entrypoints/cli.tsx  ─── 동적 import만 사용 (fast-path 보호)      │
│   ├─ --version              → 직접 출력 후 종료 (zero imports)     │
│   ├─ --dump-system-prompt   → 시스템 프롬프트 덤프 (ant only)      │
│   ├─ --claude-in-chrome-mcp → Chrome MCP 서버 모드                 │
│   ├─ --chrome-native-host   → Chrome native messaging              │
│   ├─ --daemon-worker=<kind> → 슈퍼바이저 워커 (DAEMON 게이트)      │
│   └─ (그 외)                → main.tsx 로 위임                     │
└──────────────┬─────────────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────────────────────────┐
│  main.tsx  ─── import 순서가 곧 부수효과 순서                      │
│   1. profileCheckpoint('main_tsx_entry')                           │
│   2. startMdmRawRead()       ← plutil/reg query 병렬 스폰          │
│   3. startKeychainPrefetch() ← macOS Keychain 2건 병렬             │
│   4. 나머지 ~135ms 의 imports (위 IO와 병렬로 흐름)                │
└──────────────┬─────────────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────────────────────────┐
│  entrypoints/init.ts :: init()  (memoize, 단일 실행)               │
│   • applySafeConfigEnvironmentVariables                            │
│   • loadPolicyLimits / loadRemoteManagedSettings                   │
│   • populateOAuthAccountInfoIfNeeded                               │
│   • configureGlobalMTLS / configureGlobalAgents (proxy)            │
│   • setupGracefulShutdown / registerCleanup                        │
│   • (텔레메트리는 setMeter() 호출 시 lazy import)                  │
└──────────────┬─────────────────────────────────────────────────────┘
               │
        ┌──────┴──────┐
        ▼             ▼
   대화형 REPL       헤드리스 / SDK
   replLauncher      cli/print.ts
        │             │
        ▼             ▼
   screens/REPL    QueryEngine (직접)
        │             │
        └──────┬──────┘
               ▼
         query() 제너레이터 (query.ts)
               │
               ▼
    Anthropic Messages API ← services/api/claude.ts
```

## 아키텍처 개요 (큰 그림)

### 진입과 부트스트랩

`package/cli.js`의 진입점은 `source/src/entrypoints/cli.tsx`입니다. 이 파일은 **고의적으로 동적 import만 사용**하는데, `--version` 같은 빠른 경로에서 무거운 모듈 평가를 피하기 위함입니다. `main.tsx`는 import 순서가 부수효과 순서이며(`profileCheckpoint('main_tsx_entry')`, 그 다음 macOS Keychain prefetch, MDM 설정 prefetch 등), 다른 임포트들과 병렬로 실행되어 시작 지연을 줄이도록 의도적으로 배치되었습니다. 임포트를 재배치하면 시작 성능이 저하될 수 있다는 점에 주의해야 합니다.

`source/src/entrypoints/init.ts`의 `init()`은 `memoize`된 단일 부트스트랩 함수로, OAuth 계정 채우기, 정책 한도 로딩, 원격 관리 설정 로딩, 텔레메트리 초기화, 종료 핸들러 등록 등을 일괄 처리합니다.

### 쿼리 루프 (QueryEngine)

`source/src/QueryEngine.ts`의 `QueryEngine` 클래스는 **한 대화당 하나** 생성되며, `submitMessage()`마다 새로운 턴이 같은 대화 안에서 실행됩니다. 메시지, 파일 상태 캐시, 토큰 사용량, 스킬 발견 추적 등이 턴 간에 유지됩니다. 실제 어시스턴트 호출 루프는 `source/src/query.ts`의 `query()` 제너레이터가 담당하고, QueryEngine은 그 위에서 라이프사이클 / 자원 / 정리 책임을 집니다.

대화형 REPL 경로(`source/src/replLauncher.tsx` → `source/src/screens/REPL.tsx`)와 헤드리스 / SDK 경로(`source/src/cli/print.ts`, `source/src/entrypoints/sdk/`)가 동일한 `query()` 위에 올라가 있습니다. REPL은 UI 스크롤백을 위해 전체 히스토리를 보존하고, 헤드리스는 메모리를 한정하기 위해 `HISTORY_SNIP` 게이트의 스닙 동작을 사용합니다.

#### 한 턴의 라이프사이클

```
사용자 입력 (텍스트 / 슬래시 커맨드 / 붙여넣은 이미지)
        │
        ▼
utils/processUserInput/processUserInput.ts
   ├─ "/명령어"        → processSlashCommand.tsx   (로컬 JSX 렌더 가능)
   ├─ "!shell"         → processBashCommand.tsx
   └─ 일반 텍스트       → processTextPrompt.ts
        │
        ▼  (executeUserPromptSubmitHooks 통과)
        ▼  (이미지 리사이즈, 첨부 메시지화)
        ▼
QueryEngine.submitMessage(prompt)
   ├─ setCwd(cwd)
   ├─ fetchSystemPromptParts({ tools, mcpClients, … })
   ├─ loadMemoryPrompt()                  ← memdir 통합 시점
   ├─ getInitialThinkingConfig()
   └─ ── for await (msg of query(...)) ──┐
                                         │
                              query.ts :: query()
                                 ├─ buildSystemInitMessage
                                 ├─ services/api/claude.ts  (Messages API)
                                 │      ↑↓
                                 │   AssistantMessage / partial events
                                 │      ↓
                                 ├─ 도구 호출 발견 시
                                 │      ↓
                                 │   findToolByName(name)
                                 │      ↓
                                 │   canUseTool(...) ── 권한 체크
                                 │      ↓
                                 │   tool.call(input, ctx) ── 도구 실행
                                 │      ↓ (yield ToolResult)
                                 ├─ shouldAutoCompact?  → compact/compact.ts
                                 ├─ contextCollapse?    → CONTEXT_COLLAPSE 게이트
                                 └─ 종료 조건까지 반복
        │
        ▼
SDKMessage stream → REPL 렌더 / 헤드리스 stdout (NDJSON)
        │
        ▼
recordTranscript / flushSessionStorage  (sessionStorage.ts)
```

### Tool 및 Tool 등록

`source/src/Tool.ts`는 모든 도구가 따르는 `Tool` 인터페이스, 권한 결과 타입, 진행 상황 타입을 정의합니다. `source/src/tools.ts`는 등록된 도구 목록을 만드는 중앙 집결지로, **빌드 시 데드 코드 제거(DCE)에 강하게 의존**합니다. 패턴은 다음과 같습니다:

```ts
const RemoteTriggerTool = feature('AGENT_TRIGGERS_REMOTE')
  ? require('./tools/RemoteTriggerTool/RemoteTriggerTool.js').RemoteTriggerTool
  : null
```

`bun:bundle`의 `feature(...)` 호출은 컴파일 시 상수로 평가되어, false 분기 전체가 번들에서 제거됩니다. 코드를 읽을 때 "왜 이게 `require()`(`import` 아님)이고 `eslint-disable` 주석이 있지?"라고 묻는다면, 답은 항상 **이 DCE 패턴을 유지하기 위해서**입니다. `feature('KAIROS')`, `feature('DAEMON')`, `feature('VOICE_MODE')`, `feature('BRIDGE_MODE')`, `feature('COORDINATOR_MODE')` 등 약 89개의 피처 플래그가 빌드 변형(외부 vs Anthropic 내부 "ant", 데몬, 어시스턴트 모드, voice 모드 등)을 갈라냅니다. 비슷하게 `process.env.USER_TYPE === 'ant'`로 게이트된 ant 전용 코드도 약 294군데 존재합니다.

도구 구현은 `source/src/tools/<ToolName>/`에 한 폴더씩 들어있으며, 각각 동일한 패턴(`<Tool>.tsx`, `UI.tsx`, `prompt.ts`, 권한·검증 헬퍼)을 따릅니다. 도구 카탈로그는 `BashTool`, `FileEditTool`, `FileReadTool`, `FileWriteTool`, `GlobTool`, `GrepTool`, `AgentTool`, `SkillTool`, `TaskCreateTool`/`TaskGet`/`TaskList`/`TaskUpdate`/`TaskOutput`/`TaskStop`, `EnterPlanModeTool`/`ExitPlanModeTool`, `EnterWorktreeTool`/`ExitWorktreeTool`, `WebFetchTool`/`WebSearchTool`, `NotebookEditTool`, `ToolSearchTool`, `ScheduleCronTool`(CronCreate/Delete/List), `RemoteTriggerTool`, `MCPTool`/`ListMcpResourcesTool`/`ReadMcpResourceTool`, `McpAuthTool`, `AskUserQuestionTool`, `LSPTool`, `TodoWriteTool`, `SendMessageTool`, `TeamCreateTool`/`TeamDeleteTool` 등으로 구성됩니다.

#### 도구 활성화 매트릭스 (게이트별)

```
┌──────────────────────┬─────────────────────────────────────────────┐
│ 게이트               │ 등록되는 도구들                             │
├──────────────────────┼─────────────────────────────────────────────┤
│ 항상                 │ Bash, FileEdit, FileRead, FileWrite, Glob,  │
│                      │ Grep, Agent, Skill, NotebookEdit, WebFetch, │
│                      │ WebSearch, TodoWrite, TaskStop, TaskOutput, │
│                      │ Brief, ExitPlanModeV2, Grep, Tungsten,      │
│                      │ AskUserQuestion, LSP, ListMcpResources,     │
│                      │ ReadMcpResource, ToolSearch, EnterPlanMode, │
│                      │ EnterWorktree, ExitWorktree, Config,        │
│                      │ TaskCreate, TaskGet, TaskUpdate, TaskList   │
│                      │ (TodoV2 / ToolSearch 는 옵티미스틱 토글)    │
├──────────────────────┼─────────────────────────────────────────────┤
│ USER_TYPE === 'ant'  │ REPL, SuggestBackgroundPR                   │
├──────────────────────┼─────────────────────────────────────────────┤
│ PROACTIVE 또는       │ Sleep                                       │
│ KAIROS               │                                             │
├──────────────────────┼─────────────────────────────────────────────┤
│ AGENT_TRIGGERS       │ CronCreate, CronDelete, CronList            │
├──────────────────────┼─────────────────────────────────────────────┤
│ AGENT_TRIGGERS_REMOTE│ RemoteTrigger                               │
├──────────────────────┼─────────────────────────────────────────────┤
│ MONITOR_TOOL         │ Monitor                                     │
├──────────────────────┼─────────────────────────────────────────────┤
│ KAIROS               │ SendUserFile                                │
├──────────────────────┼─────────────────────────────────────────────┤
│ KAIROS /             │ PushNotification                            │
│ KAIROS_PUSH_NOTIF…   │                                             │
├──────────────────────┼─────────────────────────────────────────────┤
│ KAIROS_GITHUB_WEB…   │ SubscribePR                                 │
├──────────────────────┼─────────────────────────────────────────────┤
│ CLAUDE_CODE_VERIFY…  │ VerifyPlanExecution (env var, feature 아님) │
├──────────────────────┼─────────────────────────────────────────────┤
│ lazy require         │ TeamCreate, TeamDelete, SendMessage         │
│ (순환 의존 회피)     │                                             │
└──────────────────────┴─────────────────────────────────────────────┘
```

### Agent Tool

`source/src/tools/AgentTool/`가 서브에이전트 시스템 전체를 담고 있습니다. `built-in/`에 기본 내장 에이전트(`exploreAgent`, `planAgent`, `generalPurposeAgent`, `verificationAgent`, `claudeCodeGuideAgent`, `statuslineSetup`)가 정의되어 있고, `loadAgentsDir.ts`가 사용자/프로젝트의 `.claude/agents/*` 마크다운 정의를 파싱합니다. `runAgent.ts`는 에이전트 인스턴스를 실행하고, `forkSubagent.ts`는 `FORK_SUBAGENT` 게이트에서 메인 컨텍스트를 분기시켜 서브에이전트를 띄우는 경로입니다. `agentMemorySnapshot.ts` / `agentMemory.ts`는 에이전트 간 메모리 전달 / 스냅샷을 처리합니다.

#### 에이전트 계층 구조

```
                    메인 세션 (LocalMainSessionTask)
                            QueryEngine
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
        AgentTool 호출    TaskCreate 호출    Team* 호출
              │                 │                 │
   ┌──────────┴──────────┐      │                 │
   │                     │      │                 │
   ▼                     ▼      ▼                 ▼
runAgent.ts         forkSubagent.ts          spawnInProcess.ts
(in-process)        (FORK_SUBAGENT)          (utils/swarm/)
   │                     │                          │
   ▼                     ▼                          ▼
LocalAgentTask     서브 QueryEngine          InProcessTeammateTask
   │               (메인의 메시지/상태       (별도 색·이름 가진
   │                스냅샷 분기)              "동료" 세션)
   │                     │                          │
   └───────────┬─────────┘                          │
               ▼                                    │
   built-in 에이전트 또는 사용자 정의               │
   ┌─────────────────────────────┐                  │
   │ Explore       (read-only)   │                  │
   │ Plan          (architect)   │                  │
   │ general-purpose             │                  │
   │ verification  (verifier)    │                  │
   │ claude-code-guide           │                  │
   │ statusline-setup            │                  │
   │ + ~/.claude/agents/*.md     │                  │
   └─────────────────────────────┘                  │
                                                    ▼
                                          leaderPermissionBridge
                                          permissionSync (mailbox)
                                          teammateLayoutManager

원격 변형:
   AgentTool 호출 ──▶ RemoteAgentTask
                       (CCR 클러스터에서 호스팅, WebSocket으로 통신)
```

### Skill 시스템

`source/src/skills/`. `bundled/index.ts`의 `initBundledSkills()`가 시작 시 모든 번들 스킬을 등록합니다. 일부 스킬은 피처 플래그로 조건부 등록됩니다(예: `KAIROS_DREAM` → `dream`, `AGENT_TRIGGERS` → `loop`). 사용자 정의 스킬은 `loadSkillsDir.ts`를 통해 `~/.claude/skills/` 등에서 로드됩니다. 등록된 번들 스킬에는 `update-config`, `keybindings`, `verify`, `debug`, `lorem-ipsum`, `skillify`, `remember`, `simplify`, `batch`, `stuck`이 항상 포함되고, 게이트에 따라 `dream`, `hunter`, `loop`, `schedule`, `claude-api`, `claude-in-chrome`, `run-skill-generator`가 추가됩니다.

### 슬래시 커맨드

`source/src/commands.ts`가 모든 슬래시 커맨드를 임포트해 한 곳에 모읍니다. 커맨드 구현은 `source/src/commands/<cmd>/` 또는 `<cmd>.ts(x)` 단일 파일 형태로 흩어져 있으며, 그 수는 80개 이상입니다(`/init`, `/review`, `/security-review`, `/compact`, `/commit`, `/resume`, `/teleport`, `/ultraplan`, `/bughunter`, `/mcp`, `/agents`, `/skills`, `/tasks`, `/plan`, ...). 사용자 입력 → 커맨드 디스패치 → 로컬 JSX 렌더 또는 LLM 메시지 변환은 `source/src/utils/processUserInput/`에서 처리됩니다.

### 상태 관리

`source/src/state/AppState.tsx` + `AppStateStore.ts`가 React 컨텍스트와 외부 store(createStore)를 함께 제공하는 **이중 진입점**입니다. React 컴포넌트가 아닌 코드(.ts)는 `AppStateStore.js`를 직접 임포트해 React 의존을 피하고, .tsx만 `AppState.tsx`를 통합니다. `source/src/bootstrap/state.ts`는 전역 단일 상태(현재 디렉터리, 세션 ID, 누적 비용/사용량, 텔레메트리 카운터 등)를 담는 **별도** 모듈이며 파일 상단에 "DO NOT ADD MORE STATE HERE"라는 경고가 있습니다 — 함부로 늘리지 마세요.

#### 상태 레이어 분리

```
┌────────────────────────────────────────────────────────────────┐
│  bootstrap/state.ts          (모듈 전역 단일 상태)             │
│   • originalCwd, projectRoot, sessionId                        │
│   • totalCostUSD, totalAPIDuration, totalToolDuration          │
│   • turnHookCount/Duration, turnToolCount/Duration             │
│   • currentTurnTokenBudget, totalInputTokens                   │
│   • registeredHookMatchers (SDK 콜백 + 플러그인 훅)            │
│   • isRemoteMode, mainLoopModelOverride, teleportedSessionInfo │
│                                                                │
│  ⚠  "DO NOT ADD MORE STATE HERE — BE JUDICIOUS"                │
│     이 파일이 부풀어오르면 새 모듈로 빼라는 강한 신호.         │
└──────────────┬─────────────────────────────────────────────────┘
               │
               │ (사이드 채널: setSessionId 등의 setter)
               ▼
┌────────────────────────────────────────────────────────────────┐
│  state/AppStateStore.ts      (React 의존 없는 store)           │
│   • createStore(initialState, onChangeAppState)                │
│   • getDefaultAppState()                                       │
│   • SpeculationState, CompletionBoundary                       │
│   • toolPermissionContext, message log, fileStateCache         │
│                                                                │
│  → .ts 코드(QueryEngine, query 등)는 이 파일을 직접 임포트     │
└──────────────┬─────────────────────────────────────────────────┘
               │ re-export + Provider 래핑
               ▼
┌────────────────────────────────────────────────────────────────┐
│  state/AppState.tsx          (React Context Provider)          │
│   • <AppStateProvider> → <HasAppStateContext> 중첩 방지        │
│   • <VoiceProvider>     (VOICE_MODE 게이트, 외부는 passthrough)│
│   • <MailboxProvider>                                          │
│   • useSettingsChange 훅 통합                                  │
│                                                                │
│  → .tsx 컴포넌트만 임포트. 부트스트랩 fast-path는 임포트 금지. │
└────────────────────────────────────────────────────────────────┘
```

### Task / 백그라운드 실행

`source/src/Task.ts`는 `local_bash`, `local_agent`, `remote_agent`, `in_process_teammate`, `local_workflow`, `monitor_mcp`, `dream`의 7가지 태스크 타입을 정의합니다. 구현체는 `source/src/tasks/<Type>Task/`에 있습니다. `LocalShellTask`는 백그라운드 bash를 띄우고, `LocalAgentTask`는 로컬에서 서브에이전트를 실행하며, `RemoteAgentTask`는 원격(클라우드) 에이전트 호출을, `InProcessTeammateTask`는 같은 프로세스에서 동료(swarm 멤버) 에이전트를 운영합니다. `dialogLaunchers.tsx`와 `LocalMainSessionTask.ts`는 메인 세션 자체를 태스크로 모델링해 백그라운드/원격 워크플로와 동일한 인터페이스로 다룹니다.

#### 태스크 타입과 ID 프리픽스

```
TaskType              prefix   생성 경로                실행 위치
─────────────────────────────────────────────────────────────────
local_bash            b        BashTool (백그라운드)    자식 프로세스
local_agent           a        AgentTool / TaskCreate   같은 프로세스
remote_agent          r        AgentTool (원격)         CCR 클러스터
in_process_teammate   t        swarm / TeamCreate       같은 프로세스
local_workflow        w        ralph / loop / team      같은 프로세스
monitor_mcp           m        Monitor 도구             MCP 서버
dream                 d        KAIROS_DREAM 스킬        같은 프로세스
                      x        (fallback, 알 수 없음)

ID 형식: prefix + 8자(36^8 ≈ 2.8조 조합, 알파벳: 0-9a-z)
파일 출력: utils/task/diskOutput.ts → getTaskOutputPath(id)
TaskStatus: pending → running → (completed | failed | killed)
```

### MCP (Model Context Protocol)

`source/src/services/mcp/`. `MCPConnectionManager.tsx`가 연결 라이프사이클을, `client.ts`가 발견된 도구·커맨드·리소스 집계를, `auth.ts`/`oauthPort.ts`/`xaa.ts`/`xaaIdpLogin.ts`가 OAuth 흐름을 다룹니다. `InProcessTransport.ts`와 `SdkControlTransport.ts`는 별도 프로세스 없이 SDK 컨트롤 채널로 MCP를 운영하는 경로입니다. `officialRegistry.ts`가 Anthropic이 큐레이팅한 MCP 서버 카탈로그를 prefetch합니다.

#### MCP 연결 토폴로지

```
┌────────────────────────────────────────────────────────────────┐
│ services/mcp/MCPConnectionManager.tsx                          │
│   서버별 상태머신: idle → connecting → ready / error / retry   │
└──────────────┬─────────────────────────────────────────────────┘
               │
       서버 설정 출처:                       Transport 선택:
       ├─ 사용자 설정 (config.ts)            ├─ stdio (자식 프로세스)
       ├─ 프로젝트 설정 (.mcp.json)          ├─ HTTP / SSE
       ├─ 플러그인 (plugins/)                ├─ WebSocket
       ├─ official registry (prefetch)       ├─ InProcessTransport
       └─ SDK 동적 등록                      │    (같은 프로세스)
                                             └─ SdkControlTransport
                                                  (SDK 컨트롤 채널)
               │
               ▼
┌────────────────────────────────────────────────────────────────┐
│ client.ts :: getMcpToolsCommandsAndResources                   │
│   서버에서 노출된 tools / prompts / resources 를 집계          │
│   ├─ tools  → tools.ts 의 카탈로그에 머지 (MCPTool 래퍼)       │
│   ├─ prompts → commands.ts (슬래시 커맨드로 노출)              │
│   └─ resources → ListMcp/ReadMcp 도구 + skills/mcpSkillBuilders│
└──────────────┬─────────────────────────────────────────────────┘
               │
               ▼
        OAuth / Elicitation 필요 시:
          ├─ services/mcp/auth.ts          (RFC 8252 PKCE)
          ├─ services/mcp/oauthPort.ts     (loopback 콜백 서버)
          ├─ services/mcp/elicitationHandler.ts  (URL 승인 다이얼로그)
          ├─ services/mcp/xaa.ts / xaaIdpLogin.ts  (Anthropic IdP)
          └─ services/mcpServerApproval.tsx  (사용자 확인 UI)
```

### 메모리 / 컴팩션

`source/src/memdir/`가 **auto-memory**(`.claude-mb/.../memory/MEMORY.md` 인덱스 + 개별 `.md` 파일들) 시스템을 구현합니다. `findRelevantMemories.ts`는 현재 컨텍스트에 관련 있는 메모리를 골라 시스템 프롬프트에 끼워넣고, `memoryAge.ts`는 만료/노화 관리를 합니다.

`source/src/services/compact/`는 컨텍스트 윈도우 관리를 담당합니다. `autoCompact.ts`는 토큰 임계치 기반 자동 컴팩션, `microCompact.ts`/`apiMicrocompact.ts`/`cachedMicroCompact`는 더 짧은 단위의 부분 압축, `reactiveCompact.ts`(피처 게이트)는 사용자 반응에 따른 트리거, `snipCompact.ts`/`snipProjection.ts`(`HISTORY_SNIP` 게이트)는 SDK용 히스토리 잘라내기를 합니다. `timeBasedMCConfig.ts`는 시간 기반 마이크로컴팩션 정책을 결정합니다.

#### 컴팩션 전략 매트릭스

```
                  토큰 임계 도달 / 사용자 /compact / 도구 후
                              │
                              ▼
                  autoCompact :: calculateTokenWarningState
                              │
              ┌───────────────┼──────────────────────┐
              ▼               ▼                      ▼
        autoCompact      microCompact            snipCompact
        (전체 요약)     (최근 N턴만 요약)      (HISTORY_SNIP)
              │               │                      │
              │               ├─ apiMicrocompact     │
              │               │   (LLM 호출)         │
              │               └─ cachedMicroCompact  │
              │                   (캐시 재사용)      │
              ▼                                      ▼
        compact.ts ::                          REPL: 전체 보존,
        buildPostCompactMessages               UI에서만 projection
              │                                헤드리스: 즉시 잘라
              ▼                                메모리 한정
        postCompactCleanup
              │
              ▼
        sessionMemoryCompact
        (SessionMemory 갱신)

추가:
   reactiveCompact  (REACTIVE_COMPACT)  사용자 거절/재시도 기반
   contextCollapse  (CONTEXT_COLLAPSE)  도구 결과 인플레이스 축약
   timeBasedMCConfig  시간대별 micro-compact 정책 (밤/장시간 세션)
```

### 원격 / 분산 모드

- **CCR (Claude Code Remote)**: `CLAUDE_CODE_REMOTE=true` 환경에서 동작하는 원격 호스팅 컨테이너 모드. `cli.tsx` 진입 시점에 `--max-old-space-size=8192`를 강제합니다. `source/src/remote/RemoteSessionManager.ts`, `SessionsWebSocket.ts`, `remotePermissionBridge.ts`, `sdkMessageAdapter.ts`가 원격 IO·권한 브리지를 담당합니다.
- **Bridge mode**: `BRIDGE_MODE` 게이트. 로컬 ↔ 원격 세션을 잇는 모드.
- **Coordinator mode**: `COORDINATOR_MODE` 게이트. `source/src/coordinator/coordinatorMode.ts`. 여러 서브에이전트를 묶어 운영하는 상위 오케스트레이션.
- **KAIROS (assistant mode)**: `feature('KAIROS')`. `source/src/assistant/`. 백그라운드에서 능동적으로 알림/제안을 하는 모드. `SleepTool`, `SendUserFileTool`, `PushNotificationTool`, `SubscribePRTool`(`KAIROS_GITHUB_WEBHOOKS`) 등이 이 모드에서만 활성화됩니다.
- **Swarm / Teammate**: `source/src/utils/swarm/`. 같은 프로세스에서 여러 에이전트 멤버를 운영하고 권한·메시지를 동기화합니다.

#### 분산 모드 매트릭스

```
┌────────────┬───────────────┬───────────────┬───────────────────┐
│ 모드       │ 게이트        │ 통신 채널     │ 핵심 모듈         │
├────────────┼───────────────┼───────────────┼───────────────────┤
│ CCR        │ CLAUDE_CODE_  │ WebSocket     │ remote/Remote     │
│ (Remote)   │ REMOTE=true   │ + 세션 ingress│ SessionManager    │
│            │ (env, feature │               │ remote/Sessions   │
│            │  아님)        │               │ WebSocket         │
│            │               │               │ remote/sdkMessage │
│            │               │               │ Adapter           │
├────────────┼───────────────┼───────────────┼───────────────────┤
│ Bridge     │ BRIDGE_MODE   │ 로컬 IPC      │ commands/bridge/  │
│            │               │ + REPL bridge │ commands/         │
│            │               │               │   bridge-kick     │
│            │               │               │ hooks/            │
│            │               │               │   useReplBridge   │
├────────────┼───────────────┼───────────────┼───────────────────┤
│ Direct     │ DIRECT_CONNECT│ 직접 TCP/WS   │ server/           │
│ Connect    │               │ (peer-to-peer)│   directConnect   │
│            │               │               │   Manager         │
│            │               │               │ hooks/            │
│            │               │               │   useDirectConnect│
├────────────┼───────────────┼───────────────┼───────────────────┤
│ Coordinator│ COORDINATOR_  │ in-process    │ coordinator/      │
│            │ MODE          │ (단일 메인이  │   coordinatorMode │
│            │               │  서브 다수    │ + 시스템 프롬프트 │
│            │               │  지휘)        │   getCoordinator  │
│            │               │               │   UserContext     │
├────────────┼───────────────┼───────────────┼───────────────────┤
│ KAIROS     │ KAIROS        │ 백그라운드    │ assistant/        │
│ (Assistant)│ + gate.js     │ 폴링/push     │ buddy/            │
│            │               │ + KAIROS_     │   (Companion      │
│            │               │   GITHUB_     │    Sprite)        │
│            │               │   WEBHOOKS    │ awaySummary       │
├────────────┼───────────────┼───────────────┼───────────────────┤
│ Swarm /    │ (전용 게이트  │ Mailbox +     │ utils/swarm/      │
│ Teammate   │  없음, 도구로 │ permissionSync│   inProcessRunner │
│            │  활성)        │ + 메시지 주입 │   spawnInProcess  │
│            │               │               │   leaderPermission│
│            │               │               │   Bridge          │
│            │               │               │   teammateLayout  │
│            │               │               │   Manager         │
├────────────┼───────────────┼───────────────┼───────────────────┤
│ Daemon     │ DAEMON        │ 슈퍼바이저 →  │ --daemon-worker   │
│            │               │ 워커 자식들   │ (cli.tsx fast-    │
│            │               │ (--daemon-    │  path)            │
│            │               │  worker)      │                   │
└────────────┴───────────────┴───────────────┴───────────────────┘
```

### UI 레이어

자체 포크된 Ink (`source/src/ink/`)를 사용하며 — 일반 npm `ink`가 아닙니다. 렌더링 파이프라인(`renderer.ts`, `reconciler.ts`, `output.ts`), 키 입력 파싱(`parse-keypress.ts`), termio 추상화(`termio/`), 폭/래핑(`stringWidth.ts`, `wrap-text.ts`)이 모두 인라인되어 있습니다. React 컴포넌트는 `source/src/components/`, 풀 스크린 흐름은 `source/src/screens/`(`REPL.tsx`, `Doctor.tsx`, `ResumeConversation.tsx`).

### 키바인딩

`source/src/keybindings/`에 자체 키바인딩 스키마·파서·리졸버가 있고, 사용자 키바인딩은 `~/.claude/keybindings.json`에서 로드됩니다(`loadUserBindings.ts`). 새 키바인딩을 추가할 때 일반 자모 문자(예: `g`, `j/k`, `/`, `n`, `[`, `v`)를 텍스트 입력 컨텍스트에서 사용하려면 `useInput`을 직접 쓰는 대신 `useKeybinding` 경로를 따라야 한다는 lint 규칙(`custom-rules/prefer-use-keybindings`)이 있습니다.

### 설정 / 마이그레이션

`source/src/utils/config.ts`가 글로벌 설정(`~/.claude/config.json` 등)을 담당하고, `source/src/migrations/`에는 버전 간 설정 마이그레이션이 모여 있습니다(예: Sonnet 1m → 4.5, 4.5 → 4.6, Opus → Opus1m, replBridgeEnabled → remoteControlAtStartup). 새 마이그레이션을 더한다면 이 디렉터리의 기존 패턴을 따라야 합니다.

### 텔레메트리 / 분석

`source/src/services/analytics/`. `sink.ts`에 게이트 초기화, `growthbook.ts`에 GrowthBook 기반 피처 게이트, `index.ts`에 `logEvent()`가 있습니다. **`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`** 라는 명시적으로 긴 이름의 타입은 이름 자체가 가드입니다 — `logEvent`에 코드/파일 경로를 메타데이터로 넘기지 않았음을 호출자가 명시적으로 선언하도록 강제합니다.

## 시퀀스 다이어그램

### 1. 시작 시퀀스 (`claude` → 첫 프롬프트까지)

```
User   cli.tsx        main.tsx       init.ts      RemoteSettings  Keychain    REPL
 │        │              │              │              │             │          │
 │ exec   │              │              │              │             │          │
 ├───────▶│              │              │              │             │          │
 │        │ fast-path? (--version/--daemon-worker/--dump-system-prompt)         │
 │        │ ─── no ───   │              │              │             │          │
 │        │              │              │              │             │          │
 │        │ dynamic import('../main.tsx')              │             │          │
 │        ├─────────────▶│              │              │             │          │
 │        │              │ profileCheckpoint('main_tsx_entry')       │          │
 │        │              │ startMdmRawRead() ────────────────────────┼─▶ spawn  │
 │        │              │              │              │     plutil/reg query   │
 │        │              │ startKeychainPrefetch() ──────────────────▶ async    │
 │        │              │              │              │     OAuth + API key    │
 │        │              │ (~135ms 의 잔여 imports, 위 IO와 병렬)               │
 │        │              │              │              │             │          │
 │        │              │ await init() │              │             │          │
 │        │              ├─────────────▶│ enableConfigs()            │          │
 │        │              │              │ applySafeConfigEnvVars     │          │
 │        │              │              │ ensureKeychainPrefetch ────┼─▶ await  │
 │        │              │              │ initializePolicyLimits     │          │
 │        │              │              │ initializeRemoteManagedSettings       │
 │        │              │              ├─────────────▶│             │          │
 │        │              │              │              │ (MDM/조직 정책 로드)   │
 │        │              │              │ populateOAuthAccountInfo   │          │
 │        │              │              │ initJetBrainsDetection     │          │
 │        │              │              │ setupGracefulShutdown      │          │
 │        │              │◀─────────────│              │             │          │
 │        │              │ initBuiltinPlugins() / initBundledSkills()│          │
 │        │              │ getCommands() / getTools()  │             │          │
 │        │              │ showSetupScreens() (필요 시 trust dialog, login)     │
 │        │              │ launchRepl(root, appProps, replProps, renderAndRun)  │
 │        │              ├─────────────────────────────────────────────────────▶│
 │        │              │              │              │             │   <App>  │
 │        │              │              │              │             │  <REPL/> │
 │◀───────┼──────────────┼──────────────┼──────────────┼─────────────┼──────────┤
 │        │              │              │              │             │  프롬프트 │
```

### 2. 사용자 텍스트 입력 → 모델 응답 (한 턴)

```
User  REPL  processUserInput  QueryEngine  query()  Anthropic API  Tool  Permission
 │     │           │              │           │           │           │       │
 │ "X"  │          │              │           │           │           │       │
 ├────▶│           │              │           │           │           │       │
 │     │ executeUserPromptSubmitHooks (사용자 훅이 차단 가능)            │       │
 │     │ ── blocked? 시스템 메시지로 종료 ───┐                            │       │
 │     ├──────────▶│              │           │           │           │       │
 │     │           │ 슬래시/!shell/텍스트 분기                            │       │
 │     │           │ createUserMessage / 이미지 첨부화                    │       │
 │     │           │           │  │           │           │           │       │
 │     │ submitMessage(prompt)   │           │           │           │       │
 │     ├───────────────────────▶ │           │           │           │       │
 │     │           │              │ fetchSystemPromptParts(tools, mcpClients) │
 │     │           │              │ loadMemoryPrompt() (memdir)               │
 │     │           │              │ for await (m of query(...))               │
 │     │           │              ├──────────▶│           │           │       │
 │     │           │              │           │ buildSystemInitMessage         │
 │     │           │              │           │ POST /v1/messages              │
 │     │           │              │           ├──────────▶│           │       │
 │     │           │              │           │           │ stream events      │
 │     │           │              │           │◀──────────┤           │       │
 │     │           │              │           │ AssistantMessage(yield)        │
 │     │◀──────────────────────────────────────┤           │           │       │
 │     │ (스트리밍 UI 업데이트)   │           │           │           │       │
 │     │           │              │           │           │           │       │
 │     │           │              │           │ tool_use 블록 감지              │
 │     │           │              │           │ findToolByName ─────▶          │
 │     │           │              │           │ canUseTool(tool, input, ctx)   │
 │     │           │              │           ├─────────────────────────────▶│
 │     │           │              │           │           │           │ 평가  │
 │     │           │              │           │           │           │ (sandbox /
 │     │           │              │           │           │           │  allowlist /
 │     │           │              │           │           │           │  사용자)
 │     │ permission UI            │           │           │           │       │
 │◀────┼──────────────────────────────────────┼───────────┼───────────┼───────┤
 │ 승인 │                          │           │           │           │       │
 ├────▶│                          │           │           │           │       │
 │     │           │              │           │           │           │ allow │
 │     │           │              │           │◀───────────────────────────────┤
 │     │           │              │           │ tool.call(input, ctx)          │
 │     │           │              │           ├──────────────────────▶│       │
 │     │           │              │           │           │           │ 실행  │
 │     │           │              │           │◀──────────────────────┤       │
 │     │           │              │           │ ToolResult 메시지 추가          │
 │     │           │              │           │ POST /v1/messages (다음 턴)    │
 │     │           │              │           ├──────────▶│           │       │
 │     │           │              │           │           │ (반복)    │       │
 │     │           │              │           │ stop_reason="end_turn"         │
 │     │           │              │◀──────────┤           │           │       │
 │     │           │              │ recordTranscript / flushSessionStorage     │
 │     │◀───────────────────────── │           │           │           │       │
 │     │ 최종 메시지 렌더          │           │           │           │       │
 │◀────┤           │              │           │           │           │       │
```

### 3. 권한 결정 흐름 (`canUseTool`)

```
query()  canUseTool  hooks.PreToolUse  permission/rules  Sandbox  PermissionDialog  User
   │         │             │                  │              │            │           │
   │ call    │             │                  │              │            │           │
   ├────────▶│             │                  │              │            │           │
   │         │ collectPreToolUseHooks                        │            │           │
   │         ├────────────▶│                  │              │            │           │
   │         │             │ 사용자 훅 / 플러그인 훅 실행                  │           │
   │         │             │ ── decision: allow/deny/ask ─┐  │            │           │
   │         │◀────────────┤                  │           │  │            │           │
   │         │ deny? → 결과 반환 (behavior='deny')        │  │            │           │
   │         │ allow? → 도구 즉시 실행                    │  │            │           │
   │         │ ask?  → 다음 단계                          │  │            │           │
   │         │             │                  │           │  │            │           │
   │         │ 룰 매칭 (allowlist / denylist / wildcards) │  │            │           │
   │         ├────────────────────────────────▶              │            │           │
   │         │◀──────────────────────────────── ── match? ─┤            │           │
   │         │             │                  │              │            │           │
   │         │ Bash 전용: shouldUseSandbox?  │              │            │           │
   │         ├────────────────────────────────────────────▶│            │           │
   │         │             │                  │              │ 안전 분석:│           │
   │         │             │                  │              │ - read-only│           │
   │         │             │                  │              │ - cd 없음 │           │
   │         │             │                  │              │ - destructive 없음     │
   │         │◀────────────────────────────────────────────┤ sandbox OK │           │
   │         │ 샌드박스 가능 → 별도 다이얼로그 없이 실행                  │           │
   │         │             │                  │              │            │           │
   │         │ 그 외 → 사용자에게 물음                                    │           │
   │         ├──────────────────────────────────────────────────────────▶│           │
   │         │             │                  │              │            │ 표시      │
   │         │             │                  │              │            ├──────────▶│
   │         │             │                  │              │            │           │
   │         │             │                  │              │            │ 응답:     │
   │         │             │                  │              │            │ allow /   │
   │         │             │                  │              │            │ deny /    │
   │         │             │                  │              │            │ session   │
   │         │             │                  │              │            │ allow     │
   │         │             │                  │              │            │◀──────────┤
   │         │◀──────────────────────────────────────────────────────────┤           │
   │         │ collectPostToolUseHooks (실행 후, 결과에 영향)              │           │
   │         │ permissionDenials 누적 (SDK 보고용)                         │           │
   │◀────────┤             │                  │              │            │           │
   │ 결과    │             │                  │              │            │           │
```

### 4. AgentTool 서브에이전트 실행

```
메인 query()  AgentTool  loadAgentsDir  runAgent  서브 QueryEngine  TaskRegistry  AppState
    │            │             │            │             │               │           │
    │ tool_use   │             │            │             │               │           │
    ├──────────▶│             │            │             │               │           │
    │            │ getActiveAgentsFromList                │               │           │
    │            ├────────────▶│            │             │               │           │
    │            │             │ 사용자 .claude/agents/*.md 파싱            │           │
    │            │             │ + built-in 머지                            │           │
    │            │◀────────────┤            │             │               │           │
    │            │ 에이전트 정의 결정 (model, tools, isolation)              │           │
    │            │             │            │             │               │           │
    │            │ generateTaskId('local_agent')  → "a..."                │           │
    │            ├──────────────────────────────────────────────────────▶│           │
    │            │             │            │             │               │ 등록      │
    │            │             │            │             │               │ pending   │
    │            │ setAppState({ tasks: append })                          │           │
    │            ├──────────────────────────────────────────────────────────────────▶│
    │            │             │            │             │               │           │
    │            │ runAgent(definition, prompt, ctx)                                    │
    │            ├────────────────────────▶│             │               │           │
    │            │             │            │ new QueryEngine(서브 config)              │
    │            │             │            ├────────────▶│               │           │
    │            │             │            │ submitMessage(에이전트 프롬프트)          │
    │            │             │            ├────────────▶│               │           │
    │            │             │            │             │ (전체 턴 루프, 위 시퀀스 2) │
    │            │             │            │             │ 상태: running           │
    │            │             │            │             │ ── 도구 호출 시 권한은   │
    │            │             │            │             │    부모 컨텍스트 상속    │
    │            │             │            │             │    (제한된 tool 화이트   │
    │            │             │            │             │    리스트 적용)         │
    │            │             │            │ stop_reason="end_turn"                  │
    │            │             │            │◀────────────┤               │           │
    │            │             │            │ agentMemorySnapshot 저장              │
    │            │             │            │ (다음 호출에서 재사용)                  │
    │            │             │            │ 상태: completed                          │
    │            │◀────────────────────────┤             │               │           │
    │            │ ToolResult (에이전트 최종 텍스트)                        │           │
    │◀───────────┤             │            │             │               │           │
    │            │             │            │             │ TaskRegistry 갱신          │
    │            │             │            │             │               │◀──────────┤
    │ 결과 메시지 │             │            │             │               │           │

원격 변형 (RemoteAgentTask):
   runAgent 대신 → remote/RemoteSessionManager.create() → WebSocket 으로 CCR 클러스터,
   결과는 sdkMessageAdapter 로 SDKMessage 로 변환되어 동일한 ToolResult 형태로 합류.
```

### 5. MCP 서버 연결 + OAuth

```
설정 로더  MCPConnectionManager  Transport  MCP 서버  auth.ts  oauthPort  브라우저  User
   │             │                    │          │          │         │          │       │
   │ config 변경 │                    │          │          │         │          │       │
   ├───────────▶│ servers diff       │          │          │         │          │       │
   │             │ for each new server: connect()           │          │          │       │
   │             │ Transport 종류:    │          │          │         │          │       │
   │             │  - stdio → 자식 프로세스 spawn            │          │         │       │
   │             │  - http/sse → fetch  ─────▶  │          │          │         │       │
   │             │  - websocket → ws 연결       │          │          │         │       │
   │             │  - InProcessTransport → 같은 프로세스 핸들러         │         │       │
   │             │  - SdkControlTransport → SDK 컨트롤 채널            │         │       │
   │             │                    │          │          │         │          │       │
   │             │ initialize 요청 송신          │          │         │          │       │
   │             ├───────────────────▶├─────────▶│          │         │          │       │
   │             │                    │          │ -32042 (Unauthorized)         │       │
   │             │                    │◀─────────┤          │         │          │       │
   │             │ OAuth 필요 감지     │          │          │         │          │       │
   │             ├──────────────────────────────────────────▶│         │          │       │
   │             │                    │          │          │ buildAuthorizeURL  │       │
   │             │                    │          │          │ (RFC 8252 PKCE)    │       │
   │             │                    │          │          ├────────▶│          │       │
   │             │                    │          │          │         │ loopback │       │
   │             │                    │          │          │         │ 서버 기동 │       │
   │             │                    │          │          │ URL 사용자에게 표시 │       │
   │             │                    │          │          │ (mcpServerApproval) │      │
   │             │                    │          │          │ ─────────────────────▶    │
   │             │                    │          │          │         │          │ 승인  │
   │             │                    │          │          │         │          │ → 브라우저
   │             │                    │          │          │         │          │ 콜백  │
   │             │                    │          │          │         │◀─────────┤       │
   │             │                    │          │          │ 토큰 교환│          │       │
   │             │                    │          │          │ POST /token        │       │
   │             │                    │          │          │ + 토큰 저장        │       │
   │             │                    │          │          │ (keychain/      │       │
   │             │                    │          │          │  secureStorage)    │       │
   │             │                    │          │          │         │          │       │
   │             │ Authorization 헤더 첨부 재시도            │         │          │       │
   │             ├───────────────────▶├─────────▶│          │         │          │       │
   │             │                    │          │ initialize result               │       │
   │             │                    │◀─────────┤          │         │          │       │
   │             │ tools/list, prompts/list, resources/list                        │       │
   │             ├───────────────────▶├─────────▶│          │         │          │       │
   │             │                    │◀─────────┤          │         │          │       │
   │             │ AppState.mcpClients 에 등록 ───────────────────────────────────────▶  │
```

### 6. 백그라운드 Bash 태스크 (장기 실행 + 모니터링)

```
query()  BashTool  bash/ast  LocalShellTask  자식 셸  TaskOutput  Monitor 도구  User
   │        │          │           │             │          │            │           │
   │ 호출   │          │           │             │          │            │           │
   ├──────▶│          │           │             │          │            │           │
   │        │ parseForSecurity (AST 분석)         │          │            │           │
   │        ├────────▶│           │             │          │            │           │
   │        │◀────────┤ 분리된 명령들, 위험도 분류 │          │            │           │
   │        │ canUseTool / shouldUseSandbox (시퀀스 3 참조)  │            │           │
   │        │          │           │             │          │            │           │
   │        │ run_in_background=true ?           │          │            │           │
   │        │ ── yes ──▶ spawnShellTask          │          │            │           │
   │        ├──────────────────────▶            │          │            │           │
   │        │          │           │ generateTaskId('b...') │            │           │
   │        │          │           │ getTaskOutputPath(id)  │            │           │
   │        │          │           │ child_process.spawn ───▶│           │           │
   │        │          │           │             │ 셸 실행  │            │           │
   │        │          │           │ stdout/err  │◀─────────┤            │           │
   │        │          │           │ ───▶ outputFile (디스크)            │           │
   │        │          │           ├─────────────────────────▶            │           │
   │        │ Task 등록 (status=running)                                  │           │
   │        │ "Started background task X" 메시지로 즉시 반환               │           │
   │◀───────┤          │           │             │          │            │           │
   │ (LLM 이 다음 작업으로 진행)    │             │          │            │           │
   │        │          │           │             │          │            │           │
   │        │          │           │ (시간 경과, 셸 계속 출력)            │           │
   │        │          │           │             │          │            │           │
   │ 후속 턴: TaskOutput 도구로 진행 상황 폴링     │          │            │           │
   ├───────────────────────────────────────────────────────▶│            │           │
   │        │          │           │             │          │ outputFile  │           │
   │        │          │           │             │          │ 에서 offset │           │
   │        │          │           │             │          │ 부터 읽기   │           │
   │◀──────────────────────────────────────────────────────┤            │           │
   │        │          │           │             │          │            │           │
   │ 또는: Monitor 도구로 stdout 라인을 알림으로 수신                       │           │
   ├──────────────────────────────────────────────────────────────────▶ │           │
   │        │          │           │             │          │            │ tail -F   │
   │        │          │           │             │          │            │ + match   │
   │        │          │           │             │          │            │ 라인마다  │
   │        │          │           │             │          │            │ 알림      │
   │◀──────────────────────────────────────────────────────────────────┤           │
   │        │          │           │ 자식 종료   │ exit code│            │           │
   │        │          │           │◀────────────┤          │            │           │
   │        │          │           │ status=completed/failed/killed       │           │
   │        │          │           │ markTaskNotified → 시스템 메시지     │           │
   │◀────────────────────────────── ┤             │          │            │           │
   │ "Task X exited with code 0"   │             │          │            │           │
   │ (다음 턴 컨텍스트에 자동 주입)  │             │          │            │           │

킬:
   TaskStop 도구 → Task.kill(taskId, setAppState) →
     - LocalShell: SIGTERM → SIGKILL (grace period)
     - LocalAgent / Teammate: abortController.abort()
     - RemoteAgent: WebSocket stop 메시지
```

### 7. 자동 컴팩션 트리거

```
query() 루프  autoCompact  tokenBudget  compact.ts  Anthropic API  postCompactCleanup
     │              │            │            │            │                │
     │ 매 모델 응답 후                          │            │                │
     ├────────────▶│            │            │            │                │
     │              │ getCurrentTurnTokenBudget()           │                │
     │              ├───────────▶│            │            │                │
     │              │◀───────────┤ usage / max_tokens 비교 │                │
     │              │ calculateTokenWarningState            │                │
     │              │ ── below threshold ─▶ no-op           │                │
     │              │ ── warning      ─▶ UI 배너 표시       │                │
     │              │ ── auto-compact ─▶ 본 시퀀스 계속     │                │
     │              │            │            │            │                │
     │              │ buildPostCompactMessages              │                │
     │              ├────────────────────────▶│            │                │
     │              │            │            │ 요약 프롬프트 구성:           │
     │              │            │            │  - 시스템: "다음 대화를 요약" │
     │              │            │            │  - user: 전체 메시지 직렬화  │
     │              │            │            │ POST /v1/messages           │
     │              │            │            ├───────────▶│                │
     │              │            │            │            │ 요약 텍스트     │
     │              │            │            │◀───────────┤                │
     │              │            │            │ SDKCompactBoundaryMessage    │
     │              │            │            │ 생성                          │
     │              │◀────────────────────────┤            │                │
     │              │ 메시지 배열을 [boundary, summary] 로 교체              │
     │              │ readFileState 부분 폐기 (스테일 캐시 정리)             │
     │              ├──────────────────────────────────────────────────────▶│
     │              │                                                       │ post-
     │              │                                                       │ compact
     │              │                                                       │ cleanup
     │              │ sessionMemoryCompact (SessionMemory 갱신)              │
     │              │ logEvent('tengu_compact_triggered', ...)               │
     │              │                                                       │
     │◀─────────────┤ 다음 모델 호출은 [boundary] 이후 메시지만 전송         │

마이크로컴팩션 (microCompact):
   동일하지만 "최근 N턴만" 요약 → 메시지 일부만 교체.
   apiMicrocompact: 일반 LLM 호출
   cachedMicroCompact: 직전 요약을 캐시에서 재사용
   reactiveCompact (게이트): 사용자 거절/재시도 시 트리거

스닙 (HISTORY_SNIP, 헤드리스 전용):
   query() 가 yield 한 시스템 메시지가 "snip boundary" 이면 snipReplay 콜백 호출.
   SDK 호출자는 ask() 진입 시 snipReplay 를 주입하여 메모리를 한정.
```

### 8. 슬래시 커맨드 디스패치 (`/init` 같은 로컬 JSX vs LLM 메시지)

```
User  PromptInput  processSlashCommand  commands.ts  Command.call()  LLM  렌더
 │         │              │                 │             │             │     │
 │ "/cmd"  │              │                 │             │             │     │
 ├───────▶│              │                 │             │             │     │
 │         │ parseSlashCommand            │             │             │     │
 │         ├─────────────▶│                 │             │             │     │
 │         │              │ findCommand(name)             │             │     │
 │         │              ├────────────────▶│             │             │     │
 │         │              │◀────────────────┤ Command 객체 (또는 null) │     │
 │         │              │                 │             │             │     │
 │         │              │ command.type 분기:            │             │     │
 │         │              │  - 'local-jsx'  → React 컴포넌트를 직접 렌더 │     │
 │         │              │                  (LLM 호출 없음, 예: /init,  │     │
 │         │              │                   /doctor, /resume, /config) │     │
 │         │              ├───────────────────────────────────────────────▶ │
 │         │              │                 │             │             │     │
 │         │              │  - 'prompt'     → 텍스트를 LLM 메시지로 변환  │     │
 │         │              │                  (예: /commit, /review)      │     │
 │         │              ├─────────────────────────────▶│             │     │
 │         │              │                              │ getPrompt(ctx) │     │
 │         │              │                              │ 인자 / 컨텍스트 합성 │
 │         │              │◀─────────────────────────────┤ "프롬프트 텍스트"   │
 │         │              │ createCommandInputMessage    │             │     │
 │         │              │ (visible: 사용자에게는 "/cmd args",          │     │
 │         │              │  LLM 에게는 합성된 프롬프트)                  │     │
 │         │              │                              │             │     │
 │         │              │  - 'local'      → 직접 함수 실행, 결과를     │     │
 │         │              │                  CommandResultDisplay 로     │     │
 │         │              │                  반환                         │     │
 │         │              │                              │             │     │
 │         │              │ (위 두 경우는 시퀀스 2 로 합류)              │     │
 │◀────────┴──────────────┴─────────────────────────────────────────────┴─────┤
```

## 디렉터리 빠른 참조

```
source/src/
├── entrypoints/        CLI 진입점, SDK 타입, init() 함수
│   ├── cli.tsx         fast-path 라우터 (--version, --daemon-worker, …)
│   ├── init.ts         memoize 된 단일 부트스트랩
│   └── sdk/            Agent SDK 타입 / 스키마
├── bootstrap/          전역 모듈 상태 (확장 금지)
├── main.tsx            REPL/headless 디스패치 + Commander 정의
├── QueryEngine.ts      대화당 한 인스턴스, 턴 라이프사이클
├── query.ts            모델 호출 + 도구 실행 루프 (제너레이터)
├── Tool.ts             Tool 인터페이스, 권한 / 진행 타입
├── Task.ts             7가지 TaskType 정의 + ID 생성
├── tools.ts            도구 등록 허브 (DCE 패턴)
├── tools/              도구 구현 (폴더당 하나)
│   ├── AgentTool/      서브에이전트 (built-in/, runAgent, fork)
│   ├── BashTool/       AST 검증, 권한, 샌드박스
│   ├── FileEditTool/   diff 생성, 히스토리 스냅샷
│   ├── McpAuthTool/    OAuth 흐름 트리거
│   └── …
├── commands.ts         슬래시 커맨드 등록 허브
├── commands/           슬래시 커맨드 구현 (80+)
├── skills/             스킬 시스템
│   ├── bundled/        번들된 기본 스킬들
│   └── loadSkillsDir.ts  사용자 스킬 로더
├── services/
│   ├── api/            Anthropic API 클라이언트, 재시도, 사용량
│   ├── mcp/            MCP 연결 매니저, OAuth
│   ├── compact/        컴팩션 전략들
│   ├── analytics/      텔레메트리 + GrowthBook
│   ├── oauth/          Anthropic 콘솔 OAuth
│   ├── policyLimits/   정책 한도 (rate limits 등)
│   ├── remoteManagedSettings/  MDM/조직 설정
│   └── …
├── state/              AppState (이중 진입점: .ts vs .tsx)
├── screens/            풀스크린 흐름 (REPL, Doctor, Resume)
├── components/         일반 React 컴포넌트
├── ink/                자체 포크된 Ink (NPM ink 아님)
├── hooks/              React 훅 (use*.ts, use*.tsx)
├── memdir/             auto-memory 시스템
├── migrations/         설정 마이그레이션
├── tasks/              TaskType별 구현 (LocalShell/Agent/Remote/…)
├── coordinator/        Coordinator 모드
├── assistant/          KAIROS 어시스턴트 모드
├── buddy/              버디 / 컴패니언 스프라이트 UI
├── remote/             CCR / 원격 세션 매니저
├── server/             DIRECT_CONNECT 서버
├── cli/                handlers + transports (WebSocket/SSE/Hybrid)
├── plugins/bundled/    번들 플러그인
├── keybindings/        자체 키바인딩 시스템
├── utils/
│   ├── swarm/          Teammate / 멀티에이전트 동기화
│   ├── processUserInput/  슬래시·bash·텍스트 디스패처
│   ├── hooks.ts        UserPromptSubmitHook 등 사용자 훅
│   ├── settings/       설정 파서 / 변화 감지
│   ├── permissions/    권한 컨텍스트 / 거부 추적
│   ├── plugins/        플러그인 로더
│   ├── bash/           Bash AST 분석 (검증·분리)
│   ├── claudeInChrome/ Chrome 통합
│   └── …
├── native-ts/          순수 TS 네이티브 폴리필 (yoga-layout, color-diff,
│                       file-index — native 빌드 환경 없을 때 대체)
├── types/              공통 타입 (message, ids, permissions, tools, hooks)
├── constants/          상수 (prompts, product, oauth, xml, querySource, …)
└── vim/, voice/, schemas/, outputStyles/, history.ts, cost-tracker.ts, …

source/vendor/          네이티브 모듈 소스 스텁 (audio-capture, image-
                        processor, modifiers-napi, url-handler)
```

## 코드를 수정할 때 알아야 할 패턴

- **DCE 패턴을 깨지 마세요.** `feature(...)`와 `process.env.USER_TYPE === 'ant'` 분기는 의도적으로 `require()`(static `import` 아님)이며 `eslint-disable` 주석이 붙어있습니다. `import`로 바꾸면 빌드 시 사라져야 할 코드가 외부 빌드에 새어 들어갑니다.
- **`bootstrap/state.ts`에 상태를 추가하지 마세요.** 파일 상단의 "DO NOT ADD MORE STATE HERE — BE JUDICIOUS WITH GLOBAL STATE" 경고를 존중하세요.
- **`main.tsx`의 import 순서는 부수효과 순서입니다.** 상단 주석에 적힌 이유(MDM raw read·Keychain prefetch가 다른 imports와 병렬로 흘러야 함)를 깨지 않도록 재배치 금지.
- **순환 임포트 깨기용 lazy `require()`.** `commands.ts`, `tools.ts`, `main.tsx`, `state/AppState.tsx`에 `const getXxx = () => require('./xxx.js')` 형태의 lazy import가 다수 있습니다. 순환 의존을 끊는 의도이며 그대로 두세요.
- **`.ts` vs `.tsx` 분리**가 React 의존 분리를 위한 도구입니다. React를 들고 들어오면 안 되는 코드 경로(부트스트랩 fast-path, 헤드리스 SDK 등)는 `.ts`만 사용하며, `state/AppState.tsx`는 `AppStateStore.ts`를 re-export해 양쪽이 같은 store를 보면서도 React 평가를 늦출 수 있게 합니다.
- **`source/` 트리의 편집은 실행 동작에 영향을 주지 않습니다.** 실행되는 코드는 `package/cli.js`이며, 소스맵에서 풀어낸 `source/src/`는 참조용입니다. 동작을 바꿔 검증하려면 번들 자체를 패치하거나 별도 빌드 환경에서 동등한 변경을 재현해야 합니다.

## 진단

빌드/실행 환경 진단은 번들된 `/doctor` 슬래시 커맨드(`source/src/commands/doctor/` 및 `source/src/screens/Doctor.tsx`)가 담당합니다 — 인증, MCP 연결, IDE 통합, 권한, 자동 업데이트 상태를 확인합니다. 사용자에게 진단을 요청할 때는 이 커맨드를 추천하세요.
