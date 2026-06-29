# PLAN: Claude Code 기반 Pixel Agents를 Codex 기반 Agent로 전환하기

## 목표

현재 Pixel Agents는 Claude Code를 실행하고, Claude의 Hook 이벤트와 JSONL transcript를 읽어서 에이전트가 일하는 상태를 픽셀 캐릭터로 보여준다.

목표는 이 흐름을 **Codex를 VS Code 터미널에서 실행하고, Codex의 작업 상태를 Pixel Agents 캐릭터로 시각화하는 구조**로 바꾸는 것이다.

권장 방향은 `claudeProvider`를 바로 덮어쓰는 것이 아니라, 기존 `HookProvider` 추상화를 살려서 `codexProvider`를 새로 추가하는 것이다. 이후 설정으로 `claude` 또는 `codex` provider를 선택하게 만들면 회귀 위험이 작고 테스트하기 쉽다.

## 현재 감지 구조 요약

현재 Claude 감지는 두 갈래다.

1. Hook 기반 즉시 감지
   - Claude Code hook이 실행된다.
   - `server/src/providers/hook/claude/hooks/claude-hook.ts`가 stdin으로 hook payload를 받는다.
   - `~/.pixel-agents/server.json`에서 Pixel Agents 서버 포트와 token을 읽는다.
   - `POST /api/hooks/claude`로 이벤트를 보낸다.
   - `server/src/hookEventHandler.ts`가 이벤트를 표준 `AgentEvent`로 처리하고 UI에 broadcast한다.

2. JSONL transcript 기반 보조 감지
   - `adapters/vscode/agentManager.ts`가 `claude --session-id <uuid>`를 실행한다.
   - 예상 transcript 파일은 `~/.claude/projects/<project>/<session-id>.jsonl`이다.
   - `server/src/fileWatcher.ts`가 이 파일을 polling한다.
   - `server/src/transcriptParser.ts`가 `tool_use`, `tool_result`, `progress`, `turn_duration`을 해석한다.

Codex 전환 시 대응되는 감지 소스는 다음이다.

- Codex hooks: `~/.codex/hooks.json` 또는 `~/.codex/config.toml`
- Codex non-interactive JSONL: `codex exec --json`
- 장기적으로는 Codex app-server: rich client/IDE 수준 통합에 적합

공식 Codex manual 기준으로 확인한 내용:

- `codex`는 interactive TUI를 실행한다.
- `codex exec`는 non-interactive 실행이다.
- `codex exec --json`은 stdout으로 JSONL 이벤트를 낸다.
- JSONL 이벤트에는 `thread.started`, `turn.started`, `turn.completed`, `turn.failed`, `item.*`, `error` 등이 있다.
- Codex hooks는 `SessionStart`, `PreToolUse`, `PermissionRequest`, `PostToolUse`, `SubagentStart`, `SubagentStop`, `Stop`, `PreCompact`, `PostCompact` 등을 지원한다.
- Codex hook은 `hooks.json` 또는 `config.toml`의 `[hooks]`에서 설정한다.

주의: 이 머신에서는 `codex --help` 실행이 `Access denied`로 실패했다. 실제 구현 전에 `codex doctor`, PATH, Windows 실행 권한, VS Code에서의 Codex CLI 실행 가능 여부를 먼저 확인해야 한다.

## 권장 아키텍처

새 구조는 provider 추가 방식으로 간다.

```text
core/
  src/provider.ts
    공통 HookProvider, AgentEvent 정의

server/src/providers/
  index.ts
    provider registry

  hook/claude/
    기존 Claude provider 유지

  hook/codex/
    codex.ts
      Codex HookProvider 구현

    codexHookInstaller.ts
      ~/.codex/hooks.json 또는 ~/.codex/config.toml에 hook 등록

    constants.ts
      Codex hook event 목록, terminal prefix, hook script 이름

    hooks/codex-hook.ts
      Codex hook stdin payload를 Pixel Agents 서버로 POST

    codexJsonParser.ts
      codex exec --json 이벤트를 Pixel Agents 상태 이벤트로 변환
```

## 1단계: Provider 선택 구조 만들기

수정 대상:

- `server/src/providers/index.ts`
- `adapters/vscode/PixelAgentsViewProvider.ts`
- `server/src/cli.ts`
- `adapters/vscode/constants.ts`
- `package.json`

현재는 곳곳에서 `claudeProvider`를 직접 import한다.

예:

```ts
import { claudeProvider, copyHookScript } from '../../server/src/providers/index.js';
```

이를 provider registry로 바꾼다.

```ts
const provider = getProvider(configuredProviderId);
const runtime = new AgentRuntime(this.store, provider);
```

추가할 설정:

```text
pixel-agents.provider = "claude" | "codex"
```

초기 migration 중에는 기본값을 `claude`로 유지하고, Codex provider가 안정화되면 기본값을 `codex`로 바꾸는 편이 안전하다.

## 2단계: VS Code 터미널에서 Codex 실행

수정 대상:

- `adapters/vscode/agentManager.ts`
- `server/src/providers/hook/codex/codex.ts`
- `server/src/providers/hook/codex/constants.ts`

현재 Claude 실행 지점:

```ts
const launch = claudeProvider.buildLaunchCommand?.(sessionId, cwd, { bypassPermissions });
terminal.sendText([launch.command, ...launch.args].join(' '));
```

Codex provider에는 `buildLaunchCommand()`를 구현한다.

후보 1: interactive Codex

```text
codex --cd <cwd>
```

장점:

- 사용자가 원하는 "VS Code 터미널에서 Codex 실행"에 가장 가깝다.
- 기존 `+ Agent` 버튼 UX와 잘 맞는다.
- Codex hooks로 작업 상태를 감지할 수 있다.

단점:

- stdout JSONL이 보장되지 않으므로 tool 상세 추적은 hook payload에 의존한다.

후보 2: non-interactive task mode

```text
codex exec --json "<prompt>"
```

장점:

- `--json` 이벤트 스트림이 있어 상태 추적이 쉽다.

단점:

- interactive agent라기보다 일회성 task 실행에 가깝다.
- launch 시 prompt가 필요하다.

권장 MVP:

- `+ Agent`는 interactive `codex`를 실행한다.
- 추후 별도 "Run Task" 기능에서 `codex exec --json`을 사용한다.

## 3단계: Codex Hook 설치기 추가

추가 대상:

- `server/src/providers/hook/codex/codexHookInstaller.ts`
- `server/src/providers/hook/codex/hooks/codex-hook.ts`
- `server/src/providers/hook/codex/constants.ts`

Claude 쪽 기존 구현:

- `server/src/providers/hook/claude/claudeHookInstaller.ts`
- `~/.claude/settings.json`에 hook command 추가
- `dist/hooks/claude-hook.js`를 `~/.pixel-agents/hooks/`로 복사

Codex 쪽 목표:

- `~/.codex/hooks.json` 또는 `~/.codex/config.toml`에 hook command 추가
- `dist/hooks/codex-hook.js`를 `~/.pixel-agents/hooks/`로 복사
- 다음 이벤트를 등록
  - `SessionStart`
  - `PreToolUse`
  - `PermissionRequest`
  - `PostToolUse`
  - `SubagentStart`
  - `SubagentStop`
  - `Stop`

예상 hook config 형태:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "node \"<home>/.pixel-agents/hooks/codex-hook.js\"",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

주의:

Codex는 non-managed hook에 대해 trust review를 요구한다. 개발 중에는 Codex CLI에서 `/hooks`로 hook을 확인/신뢰해야 할 수 있다. 자동화 환경에서만 `--dangerously-bypass-hook-trust` 사용을 검토한다.

## 4단계: Codex Hook payload 정규화

추가/수정 대상:

- `server/src/providers/hook/codex/codex.ts`
- 필요 시 `core/src/provider.ts`
- 필요 시 `server/src/hookEventHandler.ts`

`codexProvider.normalizeHookEvent(raw)`를 구현한다.

목표 mapping:

```text
SessionStart       -> { kind: "sessionStart" }
PreToolUse         -> { kind: "toolStart" }
PermissionRequest  -> { kind: "permissionRequest" }
PostToolUse        -> { kind: "toolEnd" }
Stop               -> { kind: "turnEnd" }
SubagentStart      -> { kind: "subagentStart" }
SubagentStop       -> { kind: "subagentEnd" }
```

실제 구현 전 반드시 real payload를 캡처해야 하는 항목:

- session/thread id 필드명
- hook event name 필드명
- tool name 필드명
- tool input 구조
- cwd 제공 여부
- transcript path 제공 여부

원칙:

Codex raw payload를 읽는 코드는 `codex.ts` 내부로 제한한다. `hookEventHandler.ts`는 지금처럼 provider-neutral `AgentEvent`만 다루게 유지한다.

## 5단계: Codex JSONL/Event stream parser 추가

추가/수정 대상:

- `server/src/providers/hook/codex/codexJsonParser.ts`
- `server/src/transcriptParser.ts`
- `server/src/fileWatcher.ts`
- 필요 시 `core/src/provider.ts`

Claude parser는 Claude JSONL에 강하게 묶여 있다.

현재 가정:

```text
assistant content[].tool_use
user content[].tool_result
system subtype=turn_duration
progress parentToolUseID
```

Codex `codex exec --json`은 다른 이벤트를 낸다.

```text
thread.started
turn.started
item.started
item.completed
turn.completed
turn.failed
error
```

Codex parser mapping 초안:

```text
thread.started
  -> session/thread id 기록

turn.started
  -> agentStatus active

item.started + command_execution
  -> agentToolStart, "Running: <command>"

item.completed + command_execution
  -> agentToolDone

item.started + file change/edit/write 계열
  -> agentToolStart, "Editing files" 또는 "Writing files"

item.completed + agent_message
  -> text activity 기록

turn.completed
  -> agentToolsClear + agentStatus waiting

turn.failed 또는 error
  -> agentStatus waiting, debug/status 표시
```

MVP에서는 interactive `codex` + hooks만 먼저 지원하고, `codex exec --json` parser는 2차로 구현해도 된다.

## 6단계: Hooks-only agent를 정식 지원

수정 대상:

- `server/src/agentRuntime.ts`
- `server/src/fileWatcher.ts`
- `server/src/types.ts`
- `adapters/vscode/agentManager.ts`

현재 내부 launch는 Claude JSONL 파일이 생긴다는 전제가 강하다.

```ts
const expectedFile = path.join(projectDir, `${sessionId}.jsonl`);
```

Codex interactive 모드는 안정적인 transcript path가 없을 수 있다. 따라서 hooks-only agent를 정식 경로로 만들어야 한다.

필요 변경:

- `jsonlFile: ""` 허용
- `hooksOnly: true` 허용
- provider가 `getSessionDirs`를 제공하지 않으면 JSONL polling을 건너뜀
- `SessionStart`가 도착하면 실제 Codex thread/session id와 agent를 연결
- launch 직후에는 "pending launched agent" 상태를 둘 수 있음

이미 `fileWatcher.ts`의 `adoptExternalSessionFromHook()`에는 transcript 없는 hooks-only agent 처리 힌트가 있다. 이 패턴을 내부 launch에도 확장한다.

## 7단계: Codex tool status label 정의

수정 대상:

- `server/src/providers/hook/codex/codex.ts`
- `webview-ui/src/office/toolUtils.ts`

`formatToolStatus()`를 Codex tool 이름에 맞게 구현한다.

초기 mapping:

```text
Bash / command_execution
  -> Running: <command>

apply_patch / Edit / Write
  -> Editing files

read_file / Read
  -> Reading files

web_search
  -> Searching the web

mcp__*
  -> Using MCP tool
```

Codex provider에 설정할 항목:

```ts
permissionExemptTools;
subagentToolNames;
readingTools;
terminalNamePrefix;
```

`terminalNamePrefix`는 우선 `Codex`로 둔다.

## 8단계: build script가 여러 hook script를 빌드하게 변경

수정 대상:

- `esbuild.js`
- `.vscodeignore`
- `package.json`

현재 `esbuild.js`는 Claude hook만 빌드한다.

```text
server/src/providers/hook/claude/hooks/claude-hook.ts
```

이를 provider hook script 전체를 빌드하도록 바꾼다.

```text
server/src/providers/hook/*/hooks/*-hook.ts
```

출력:

```text
dist/hooks/claude-hook.js
dist/hooks/codex-hook.js
```

VS Code extension 패키징에 새 hook 파일이 포함되는지도 확인한다.

## 9단계: UI 문구와 설정을 provider-neutral하게 변경

수정 대상:

- `webview-ui/src/App.tsx`
- `webview-ui/src/components/BottomToolbar.tsx`
- `webview-ui/src/components/SettingsModal.tsx`
- `webview-ui/src/components/ChangelogModal.tsx`
- `README.md`
- `package.json`

Claude 전용 문구를 일반화한다.

예:

```text
Open Claude
  -> Open Agent

Claude terminal
  -> Agent terminal

Claude Code Hooks
  -> Provider hooks / Instant Detection
```

설정 화면에는 Provider 선택을 추가한다.

```text
Provider: Claude Code / Codex
```

## 10단계: 테스트 추가

추가/수정 대상:

- `server/__tests__/codex.test.ts`
- `server/__tests__/codex-hook.test.ts`
- `server/__tests__/hookEventHandler.test.ts`
- `e2e/fixtures/mock-codex`
- `e2e/fixtures/mock-codex-runner.cjs`
- `e2e/tests/codex/...`

Unit test:

- Codex `PreToolUse` -> `toolStart`
- Codex `PermissionRequest` -> permission bubble
- Codex `Stop` -> waiting/done
- Codex `codex exec --json` `item.started command_execution` -> `agentToolStart`
- Codex `turn.completed` -> `agentStatus waiting`

E2E test:

- mock Codex executable을 만든다.
- interactive launch를 흉내낸다.
- Codex hook payload를 Pixel Agents server로 보낸다.
- 필요 시 `codex exec --json` JSONL stream도 흉내낸다.

기존 Claude e2e 구조를 복제해서 `claude` 대신 `codex` fixture를 붙이는 방향이 좋다.

## 주요 리스크

1. Codex hook payload의 실제 필드명이 아직 미확정이다.
   - 먼저 debug hook으로 real payload를 캡처해야 한다.

2. Codex hooks는 trust review가 필요할 수 있다.
   - `/hooks` 안내와 개발용 bypass 옵션을 문서화해야 한다.

3. Interactive `codex`는 stdout JSONL을 내지 않을 수 있다.
   - interactive는 hooks 기반으로 감지하고, JSONL이 필요한 경우 `codex exec --json` task mode를 별도로 둔다.

4. 현재 runtime은 JSONL 파일 존재를 강하게 가정한다.
   - hooks-only internal agent를 정식 지원해야 한다.

5. 현재 머신에서 `codex.exe`가 Access denied로 실행되지 않는다.
   - 구현 전 로컬 Codex 설치/권한 문제를 해결해야 한다.

## 추천 구현 순서

1. provider registry와 `pixel-agents.provider` 설정 추가
2. `codexProvider` skeleton 추가
3. VS Code `+ Agent`에서 `codex` 실행
4. Codex hook script와 installer 추가
5. `esbuild.js`를 multi-provider hook build로 변경
6. 실제 Codex hook payload 캡처
7. `normalizeHookEvent()` 구현
8. hooks-only launched agent 지원
9. provider-neutral UI 문구 적용
10. mock Codex 기반 unit/e2e 테스트 추가
11. `codex exec --json` task mode 추가
12. Codex app-server 통합 검토

## MVP 기준

MVP는 다음만 되면 된다.

- VS Code의 `+ Agent`가 Codex 터미널을 실행한다.
- Pixel Agents가 Codex session 캐릭터를 만든다.
- `PreToolUse`에서 캐릭터가 active 상태가 된다.
- tool 이름/status가 말풍선 또는 overlay에 표시된다.
- `PermissionRequest`에서 permission bubble이 뜬다.
- `Stop`에서 waiting/done 상태가 된다.
- 터미널을 닫으면 캐릭터가 사라진다.

MVP 이후로 미룰 것:

- `codex exec --json` task mode
- Codex app-server 직접 연동
- Codex cloud task 시각화
- rich diff/file-change 표시
- provider 선택 UI 고도화

## 변경 기록

[변경] provider registry를 추가하고 `claude`/`codex` provider 선택 경로를 만들었다 / 이유: Claude 전용 import를 덮어쓰지 않고 Codex를 병렬 provider로 붙여 회귀 위험을 줄이기 위해서.

[변경] `codexProvider`, Codex hook installer, Codex hook bridge script를 추가했다 / 이유: Codex hook payload를 Pixel Agents 서버의 `/api/hooks/codex`로 전달하고 표준 `AgentEvent`로 정규화하기 위해서.

[변경] VS Code `+ Agent` 실행 경로가 active provider의 `buildLaunchCommand()`를 사용하도록 바꿨다 / 이유: 설정에서 Codex를 선택하면 VS Code 터미널에서 `codex --cd <cwd>`를 실행해야 하기 때문에.

[변경] hooks-only agent 저장/복원 필드(`hooksOnly`, `providerId`)를 persist 경로에 추가했다 / 이유: Codex interactive 세션은 안정적인 Claude식 JSONL 파일을 보장하지 않으므로 hook 이벤트만으로도 agent를 유지해야 하기 때문에.

[변경] Hook HTTP route가 provider별 raw payload를 조건 없이 handler로 넘기도록 완화했다 / 이유: Codex payload는 `session_id`/`hook_event_name` 외의 field name을 쓸 수 있고, 이 차이는 provider normalize 단계에서 처리해야 하기 때문에.

[변경] `HookEventHandler`가 Codex `SessionStart`의 실제 session/thread id를 launch 직후 hooks-only agent에 연결하도록 보강했다 / 이유: VS Code에서 먼저 만든 임시 agent와 Codex가 알려주는 실제 session id를 매칭해야 이후 tool/permission/stop 이벤트가 같은 캐릭터로 라우팅되기 때문에.

[변경] standalone CLI도 active provider를 읽고 hook 설치/외부 스캔을 provider 기준으로 수행하도록 바꿨다 / 이유: VS Code 확장뿐 아니라 `npx pixel-agents` 실행에서도 Claude 고정 동작을 제거하기 위해서.

[변경] `esbuild.js`가 `server/src/providers/hook/*/hooks/*-hook.ts` 전체를 `dist/hooks`로 빌드하도록 바꿨다 / 이유: Claude hook script뿐 아니라 Codex hook script도 확장/CLI 패키지에 포함되어야 하기 때문에.

[변경] webview 설정에 Provider 선택 UI와 provider 상태 수신/저장 메시지를 추가했다 / 이유: 사용자가 화면에서 Claude Code 또는 Codex를 선택하고 다음 agent launch 흐름에 반영할 수 있어야 하기 때문에.

[변경] UI/manifest의 Claude 전용 문구 일부를 agent/provider 중립 문구로 바꿨다 / 이유: 동일한 Pixel Agents 화면이 Claude와 Codex provider를 모두 설명할 수 있어야 하기 때문에.

[변경] 화면 버튼, 설정 모달, 레이아웃 편집 툴바, 에이전트 오버레이 상태 문구를 한국어로 바꿈 / 이유: 사용자가 보는 Layout/Settings/마우스오버 상태가 영어로 남아 있으면 Codex용 Agent 데모 흐름이 어색해지기 때문

[변경] Claude/Codex provider의 tool status formatter와 webview tool-name 추론 매핑에 한국어 상태 문구를 추가함 / 이유: `실행 중`, `승인 필요`, `웹 검색 중` 같은 한국어 상태가 표시되어도 캐릭터 애니메이션과 하위 작업 라벨 추출이 깨지지 않게 하기 위해

[변경] 에이전트 사회적 행동 스케줄러를 추가해 여러 에이전트가 주기적으로 회의 테이블/소파 주변으로 이동하고 대화 말풍선을 주고받게 함 / 이유: 화면이 단순 상태 표시가 아니라 작업 중 검색, 승인, 논의 상황을 살아 있는 팀처럼 보여야 하기 때문

[변경] 캐릭터 FSM에 `socialMode`와 대화 말풍선 상태를 추가하고, 회의 중에는 활성 에이전트도 책상으로 강제 복귀하지 않게 조정함 / 이유: 작업 중인 에이전트가 의논하러 이동하는 연출을 만들려면 기존 “활성=항상 자기 자리” 규칙을 잠시 예외 처리해야 하기 때문
