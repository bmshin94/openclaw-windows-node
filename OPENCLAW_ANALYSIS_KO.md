# OpenClaw Windows Node 전수조사 및 활용 분석 (한국어)

> 이 문서는 `bmshin94/openclaw-windows-node` 리포지토리를 전수조사한 결과와,
> 활용 방법 / 기술 정체성 / 수익화 전략을 정리한 개인 분석 노트입니다.
> 코드 동작을 변경하지 않는 문서 전용 파일입니다.

- **분석 대상 리포지토리**: https://github.com/bmshin94/openclaw-windows-node
- **원본(upstream) 리포지토리**: https://github.com/openclaw/openclaw-windows-node
- **상위 생태계(게이트웨이 본체)**: https://github.com/openclaw/openclaw
- **공식 사이트**: https://openclaw.ai/
- **공식 문서**: https://docs.openclaw.ai/platforms/windows
- **작성일**: 2026-09-28
- **분석 기준 브랜치**: `claude/zealous-pascal-g9j67y`
- **분석 기준 커밋**: `0f8eb68` (Merge pull request #1 from bmshin94/feat/claude-guide)

---

## 목차

1. [프로젝트 정체 요약](#1-프로젝트-정체-요약)
2. [출신 성분과 생태계 구조](#2-출신-성분과-생태계-구조)
3. [내부 아키텍처 전수조사](#3-내부-아키텍처-전수조사)
4. [Capability 12종과 명령 59개](#4-capability-12종과-명령-59개)
5. [보안 설계: 4중 관문](#5-보안-설계-4중-관문)
6. [로컬 MCP 서버](#6-로컬-mcp-서버)
7. [설치 및 사용법](#7-설치-및-사용법)
8. [플러그인인가 스킬인가 MCP인가](#8-플러그인인가-스킬인가-mcp인가)
9. [API 토큰 정리](#9-api-토큰-정리)
10. [왜 GitHub에서 유명한가](#10-왜-github에서-유명한가)
11. [로컬 에이전트 구축에 주는 도움](#11-로컬-에이전트-구축에-주는-도움)
12. [React / PHP로 만들 수 있는 범위](#12-react--php로-만들-수-있는-범위)
13. [수익화 아이디어 10선](#13-수익화-아이디어-10선)
14. [실행 로드맵과 리스크](#14-실행-로드맵과-리스크)
15. [참고 문서 인덱스](#15-참고-문서-인덱스)

---

## 1. 프로젝트 정체 요약

한 줄 정의:

> **OpenClaw**라는 오픈소스 개인 AI 에이전트가 **Windows PC를 직접 조작할 수 있게 해주는 네이티브 클라이언트**.

- 정식 제품명: **OpenClaw Companion** (애칭 **Molty**)
- 기술 스택: **.NET 10 + WinUI 3**
- 배포 형태: Windows 설치 파일 (x64 / ARM64), MSIX 스토어 알파 채널도 존재
- 요구 사항: Windows 10 20H2 이상 또는 Windows 11, WebView2 런타임

핵심 포인트: **이것 단독으로는 AI가 아니다.** 판단을 하는 게이트웨이(뇌)가 별도로 있어야 하며,
이 리포는 Windows 쪽 "손, 눈, 귀, 입"에 해당하는 노드 구현체다.

### 규모 지표 (실측)

| 지표 | 수치 |
| --- | --- |
| C# 소스 라인 수 | 375,857줄 |
| C# 파일 수 | 1,095개 |
| 테스트 파일 수 | 418개 |
| 테스트 케이스 수 (`[Fact]` / `[Theory]`) | 6,706개 |
| 문서 수 (Markdown) | 40개 이상 |
| CI 워크플로 | 8개 |
| 검증 스크립트 | 38개 (`scripts/`) |
| 에이전트 스킬 | 8종 (`.agents/skills/`) |

---

## 2. 출신 성분과 생태계 구조

| 항목 | 내용 |
| --- | --- |
| 라이선스 | **MIT** (`Copyright (c) 2025 Scott Hanselman`) |
| 최다 기여자 | Scott Hanselman (Microsoft, .NET 커뮤니티 유명 개발자) |
| 기타 기여자 | Karen, Dallin Romney, Steve Allen, Pedro Larroy, Natalie Aguinaldo, dependabot 등 |
| 이 리포의 성격 | upstream의 개인 포크 |
| 포크에서 추가된 것 | `cbe70ab docs: appended CLAUDE.md persona guide` (개발 파트너 페르소나 정의) |

### 생태계 계층

```text
[1] OpenClaw Core (openclaw/openclaw)
    = 게이트웨이. 에이전트의 판단 주체이자 교환국
    - Node.js 기반
    - 채널 연동: WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage 등
    - GitHub 스타 수십만 규모 (2026년 초 기준 20만 이상)
              |
              |  WebSocket (기본 ws://localhost:18789)
              v
[2] 이 리포 = OpenClaw Windows Node
    = Windows의 실행 주체
    - 트레이 앱 + 노드 클라이언트 + 로컬 MCP 서버 + winnode CLI
```

두 역할 구분이 중요하다. 자세한 정의는 [docs/OPERATOR_NODE_CONCEPTS.md](docs/OPERATOR_NODE_CONCEPTS.md) 참고.

| 역할 | 설명 |
| --- | --- |
| **Operator** | 사용자 제어 역할. Quick Send, 채팅, 진단, 채널 제어, 페어링 승인 |
| **Node** | 피제어 머신 역할. Windows 능력을 광고하고 승인된 호출을 대기 |

---

## 3. 내부 아키텍처 전수조사

`src/` 하위 프로젝트 9개:

| 프로젝트 | 역할 |
| --- | --- |
| `OpenClaw.Tray.WinUI` | 트레이 앱 본체, Companion Settings, 채팅, Command Center |
| `OpenClaw.Shared` | 게이트웨이 클라이언트, Capability 12종, MCP 브리지, MXC 샌드박스 |
| `OpenClaw.Connection` | 게이트웨이 레지스트리, 자격증명 해석, 페어링, 연결 상태머신 |
| `OpenClaw.SetupEngine` | WSL 게이트웨이 설치 엔진, 로컬 AI(llama-server) 설치 |
| `OpenClaw.SetupEngine.UI` | 첫 실행 온보딩 마법사 화면 |
| `OpenClaw.Chat` | 네이티브 채팅 모델, 타임라인 리듀서 |
| `OpenClaw.WinNode.Cli` | `winnode.exe` CLI |
| `OpenClaw.Cli` | 게이트웨이 WebSocket 검증 CLI |
| `OpenClawTray.FunctionalUI` | 선언형 WinUI 헬퍼 |

`tests/` 하위 테스트 프로젝트 11개 + 지원 프로젝트 2개:

`OpenClaw.Shared.Tests`, `OpenClaw.Tray.Tests`, `OpenClaw.Connection.Tests`,
`OpenClaw.SetupEngine.Tests`, `OpenClaw.WinNode.Cli.Tests`, `OpenClaw.Tray.UITests`,
`OpenClaw.Tray.IntegrationTests`, `OpenClawTray.FunctionalUI.Tests`, `OpenClaw.E2ETests`,
`PackagingTests`, `OpenClaw.TestSupport`, `OpenClaw.Shared.TestHost`

### 눈에 띄는 엔지니어링 규율

- **아키텍처 원장**: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)가 책임 소유권을 `authoritative` / `closed`로 관리한다.
  `closed`로 표시된 책임을 god object에 다시 추가하는 PR은 리뷰에서 거부된다.
- **god file 감축 대상 지정**: `src/OpenClaw.Tray.WinUI/App.xaml.cs`,
  `src/OpenClaw.Tray.WinUI/Pages/ConnectionPage.xaml.cs`를 명시적 감축 대상으로 선언하고 가드레일을 문서화.
- **문서와 코드 드리프트 방지**: `SkillMdDriftTests`가 스킬 문서의 명령 목록과 실제 코드
  (`McpToolBridge.CommandDescriptions`)를 비교해 불일치 시 CI를 실패시킨다.
- **증거 기반 PR**: [docs/PROOF_POOLS.md](docs/PROOF_POOLS.md)에 정의된 증거 풀 ID를 PR에 선택하고,
  현재 HEAD 기준 실제 동작 증거를 첨부해야 한다.
- **경로 기반 CI 분류**: `scripts/Get-CiChangeClassification.ps1`이 변경 경로를 분류해 필요한
  테스트 스위트만 선택 실행한다.
- **에이전트 계약**: [AGENTS.md](AGENTS.md)가 에이전트의 필수 검증 명령과 서브시스템별 추가 검증 경로를 강제한다.

---

## 4. Capability 12종과 명령 59개

`src/OpenClaw.Shared/Capabilities/` 실측 결과 12개 Capability 클래스가 존재하고,
`McpToolBridge.CommandDescriptions`에 59개 명령이 등록되어 있다.

| Capability | 기능 | 대표 명령 |
| --- | --- | --- |
| System | 셸 명령 및 스크립트 실행 (승인 + 샌드박스 필수) | `system.run`, `system.run.prepare`, `system.which`, `system.notify`, `system.execApprovals.get`, `system.execApprovals.set` |
| Screen | 스크린샷, 화면 녹화 | `screen.snapshot`, `screen.record` |
| Camera | 웹캠 사진 및 짧은 클립 | `camera.snap`, `camera.clip`, `camera.list` |
| Canvas | 호스팅 창에 시각 콘텐츠 표시 및 상호작용 | `canvas.present`, `canvas.hide`, `canvas.navigate`, `canvas.snapshot`, `canvas.eval`, `canvas.caps`, `canvas.a2ui.push`, `canvas.a2ui.pushJSONL`, `canvas.a2ui.dump`, `canvas.a2ui.reset` |
| Browser | 호환 Chromium 브라우저 원격 조작 | `browser.proxy` |
| Location | PC 대략 위치 조회 | `location.get` |
| TTS | 스피커로 음성 출력 | `tts.speak`, `tts.status` |
| STT | 마이크 오디오를 로컬에서 전사 | `stt.transcribe`, `stt.listen`, `stt.status` |
| Device | 기기 정보 및 상태 | `device.info`, `device.status` |
| Ollama | Windows에 설치된 Ollama를 게이트웨이에 공유 | `ollama.models`, `ollama.chat` |
| App (로컬 전용) | 트레이 앱 자체 자동화 | `app.status`, `app.menu`, `app.navigate`, `app.search`, `app.agents`, `app.nodes`, `app.sessions`, `app.dashboard.url`, `app.config.get`, `app.settings.get`, `app.settings.set`, `app.chat.send`, `app.chat.snapshot`, `app.chat.reset`, `app.chat.queue.list`, `app.chat.queue.cancel` |
| AppConnection (로컬 전용) | 연결 및 페어링 워크플로 | `app.connection.status`, `app.connection.gateways`, `app.connection.reconnect`, `app.connection.reconnectNode`, `app.connection.pendingApprovals`, `app.connection.approveDevicePairing`, `app.connection.rejectDevicePairing`, `app.connection.approveNodePairing`, `app.connection.rejectNodePairing`, `app.connection.applySetupCode`, `app.connection.connectSharedToken` |

`app.*` 및 `app.connection.*`은 로컬 MCP에만 노출되고 원격 게이트웨이 노드 전송에는 등록되지 않는다.

### A2UI

Windows 노드는 A2UI v0.8을 **네이티브 WinUI 3 / XAML로 렌더링**한다. WebView가 아니다.
지원 컴포넌트 18종: Row, Column, List, Card, Tabs, Modal, Divider, Text, Image, Icon, Video,
AudioPlayer, Button, CheckBox, TextField, DateTimeInput, MultipleChoice, Slider.

카탈로그 엄격 모드이므로 목록 밖 컴포넌트는 Unknown 플레이스홀더로 렌더링된다.
JSONL 엔벨로프 4종: `surfaceUpdate`, `dataModelUpdate`, `beginRendering`, `deleteSurface`.
1 MiB 초과 라인은 서버 측에서 버려지고, 잘못된 라인은 로그 후 건너뛴다 (스트림을 중단하지 않음).

자세한 내용은 [src/skills/windows-a2ui/SKILL.md](src/skills/windows-a2ui/SKILL.md) 참고.

---

## 5. 보안 설계: 4중 관문

이 프로젝트의 가장 큰 기술적 가치는 "AI에게 명령 실행 권한을 주면서도 안전을 유지하는 방법"이다.

```text
AI 요청
  |
  +-- [1] 게이트웨이 allowCommands 정책      (서버 측 허용 목록)
  +-- [2] Windows Permissions 토글           (내가 켠 능력만 광고)
  +-- [3] exec-approvals.json                (명령별 로컬 승인 기록, V2 exec 승인)
  +-- [4] MXC / AppContainer 샌드박스 격리   (프로세스 격리)
  |
  +--> 실행
```

### 샌드박스 정책 3단계

| 정책 | 인터넷 | 클립보드 | 사용자 폴더 |
| --- | --- | --- | --- |
| **Locked Down** | 차단 | 차단 | 차단 |
| **Recommended** | 허용 | 읽기 허용 | 일반 폴더 읽기 전용 |
| **Unprotected** | 허용 | 광범위 허용 | 광범위 허용 (위험 감수) |

커스텀 설정으로 폴더 접근, 네트워크, 클립보드, 타임아웃, 출력 한도를 개별 지정할 수 있다.
MXC를 사용할 수 없고 strict fallback blocking이 꺼져 있으면 비격리 호스트 실행으로 폴백할 수 있으므로 설정에 주의해야 한다.

관련 소스: `src/OpenClaw.Shared/Mxc/` (`MxcExecutor.cs`, `MxcPolicyBuilder.cs`,
`MxcIsolationTierPolicy.cs`, `SandboxPolicy.cs`, `DirectAppContainerExecutor.cs`)

### 자격증명 우선순위와 불변 조건

```text
device token  >  shared gateway token  >  bootstrap token
```

- 페어링된 기기를 하위 토큰으로 **강등하지 않는다**. (문서화된 불변 조건)
- 게이트웨이 자격증명은 `SettingsData.Token` / `SettingsData.BootstrapToken`에 더 이상 저장하지 않는다.
  레거시 JSON 필드는 1회 마이그레이션 용도로만 읽는다.
- 활성 게이트웨이 레코드: `%APPDATA%\OpenClawTray\gateways.json`
- 기기 신원 파일: `%APPDATA%\OpenClawTray\gateways\<gateway-id>\device-key-ed25519.json` (Ed25519)

자세한 내용은 [docs/CONNECTION_ARCHITECTURE.md](docs/CONNECTION_ARCHITECTURE.md) 참고.

### MXC 관련 변경 시 추가 검증

MXC 샌드박싱, `system.run`, exec 승인, Windows 노드 명령 실행을 변경하면
`.\scripts\validate-mxc-e2e.ps1`을 실행해야 한다. `-AllowSkip` 실행 결과는 머지 검증으로 인정되지 않는다.

---

## 6. 로컬 MCP 서버

- 엔드포인트: `http://127.0.0.1:8765/` (루프백 전용)
- 프로토콜: JSON-RPC 2.0, MCP 프로토콜 버전 `2024-11-05`
- 구현: `src/OpenClaw.Shared/Mcp/McpHttpServer.cs`, `src/OpenClaw.Shared/Mcp/McpToolBridge.cs`
- 지원 메서드: `initialize`, `tools/list`, `tools/call`, `ping`,
  `notifications/initialized`, `notifications/cancelled`

### 핵심 설계: 단일 레지스트리, 다중 전송

Capability 목록이 `WindowsNodeClient`가 아니라 `NodeService`에 존재한다.
이 한 가지 변경 덕분에 **게이트웨이 클라이언트가 선택 사항**이 되고, MCP 전용 모드가 가능해진다.

`McpToolBridge`는 스냅샷이 아니라 `Func<IReadOnlyList<INodeCapability>>`를 받는다.
따라서 `tools/list`가 매번 라이브 목록을 다시 읽고, **서버 시작 후 등록한 Capability도 즉시 노출된다.**
새 Capability 추가 시 MCP 쪽 코드 변경이 필요 없다.

### MCP 전용 모드

`EnableMcpServer=true`, `EnableNodeMode=false` 조합이면 게이트웨이 자격증명 없이
로컬 `NodeService`를 시작한다. 게이트웨이, 계정, 토큰, 페어링, 터널 모두 없이 Capability를 검증할 수 있다.

### 인증

| 항목 | 값 |
| --- | --- |
| 방식 | Bearer 토큰 (모든 요청 필수) |
| 파일 경로 | `%APPDATA%\OpenClawTray\mcp-token.txt` |
| 생성 시점 | `NodeService.StartMcpServer()` 최초 호출 시 (Local MCP Server를 처음 켤 때) |
| 사양 | 32바이트 CSPRNG, base64url, 패딩 제거 -> 43 ASCII 문자 (약 256비트) |
| 방어 계층 | 루프백 바인드 + `IPAddress.IsLoopback` 재확인 + Origin/Host 검사 + 토큰 게이트 |

토글을 한 번도 켜지 않았다면 토큰 파일이 존재하지 않는다. 이 지점에서 많은 사용자가 혼란을 겪는다.

자세한 내용은 [docs/MCP_MODE.md](docs/MCP_MODE.md) 참고.

---

## 7. 설치 및 사용법

### 7.1 일반 사용자 경로 (빌드 불필요)

1. 설치 파일 다운로드: `OpenClawCompanion-Setup-x64.exe` 또는 `OpenClawCompanion-Setup-arm64.exe`.
   `OpenClawCompanion-SHA256SUMS.txt`로 해시 검증 권장.
2. 더블클릭 실행. 관리자 권한 불필요. SmartScreen 경고 시 "추가 정보 -> 실행".
3. 옵션 선택: 바탕화면 아이콘, Windows 시작 시 자동 실행(권장).
4. 첫 실행 온보딩 마법사 7단계:
   1. 보안 안내 (신뢰할 수 있는 PC 확인)
   2. 게이트웨이 선택: `Install a local gateway (WSL)` 또는 `Connect to an existing gateway`
   3. Capability 프로필 선택 및 Windows 권한 상태 확인
   4. 로컬 설치 진행 (전용 `OpenClawGateway` WSL 인스턴스 생성, 기존 Ubuntu는 건드리지 않음)
   5. 게이트웨이 설치 완료
   6. OpenClaw onboard (모델 제공자 및 API 키 설정)
   7. 완료 요약

트레이 아이콘 색: 초록 연결됨, 주황 연결 중, 빨강 오류, 회색 끊김.

### 7.2 노드 모드 활성화

```text
트레이 아이콘 우클릭 -> Companion Settings...
  Connection      : 게이트웨이 연결 및 페어링 승인
  Sandbox         : 격리 정책 선택 (Recommended 권장)
  Permissions     : Node mode 켜기 + 사용할 Capability 선택
  Command Center  : 노드 연결 확인, allowlist 및 재승인 경고 해결
```

프라이버시 민감 Capability(카메라, 화면 녹화, 마이크 전사, 음성 출력, 명령 실행)는
실제로 사용할 때만 켜는 것이 기본 권장 사항이다.

### 7.3 MCP 모드 및 winnode CLI

```text
1) Settings -> Advanced -> Local MCP Server 켜기
2) 토큰 확인: %APPDATA%\OpenClawTray\mcp-token.txt
3) 도구 목록:   winnode --list-tools
4) 명령 호출:
   winnode --command system.notify --params '{"title":"OpenClaw","body":"hello"}'
   winnode --command screen.snapshot
   winnode --command system.run --params '{"command":["cmd.exe","/d","/s","/c","echo hello"],"rawCommand":"echo hello"}'
```

주요 플래그:

| 플래그 | 설명 |
| --- | --- |
| `--command <name>` | 호출할 노드 명령 (필수) |
| `--params '<json>'` | JSON **객체** 문자열. `--params @<path>`로 파일 로드 가능 |
| `--list-tools` | 라이브 `tools/list` 조회 |
| `--invoke-timeout <ms>` | 기본 15000, 최대 600000 |
| `--mcp-url` / `--mcp-port` | 엔드포인트 재지정 (기본 8765) |
| `--mcp-token <token>` | 토큰 직접 지정 (비권장) |
| `--identity release\|dev` | 트레이 프로필 선택 |
| `--verbose` | 상세 로그 |
| `--node`, `--idempotency-key` | 게이트웨이 CLI와의 호환용. 로컬에서는 무시됨 |

종료 코드: `0` 성공, `1` 툴/JSON-RPC/전송 오류 또는 HTTP 비2xx, `2` 인자 오류.

중요 안전 동작 2가지:

- `--mcp-token`으로 넘긴 값은 같은 사용자의 다른 프로세스가 OS 프로세스 목록에서 볼 수 있다.
  CLI가 stderr로 경고한다. `OPENCLAW_MCP_TOKEN` 환경변수 또는 온디스크 토큰 파일을 사용하는 것이 안전하다.
- `--mcp-url`이 루프백이 아닌 호스트를 가리키면 CLI는 **자동 로드한 로컬 MCP 토큰 전송을 거부**한다.

`system.run`은 V2 노드 경계에서 문자열 형태 `command`를 `command-array-required`로 거부한다.
셸 사용은 argv에 명시해야 한다. 자세한 내용은 [.agents/skills/winnode/SKILL.md](.agents/skills/winnode/SKILL.md) 참고.

### 7.4 개발자 경로 (소스 빌드, Windows 필요)

```powershell
.\scripts\setup-dev.ps1
.\scripts\setup-dev.ps1 -CheckOnly

.\build.ps1
.\build.ps1 -Project WinUI
dotnet build .\src\OpenClaw.Tray.WinUI\OpenClaw.Tray.WinUI.csproj -r win-x64

.\run-app-local.ps1 -AllowNonMain -Isolated

$env:OPENCLAW_REPO_ROOT = (Get-Location).Path
dotnet test .\tests\OpenClaw.Shared.Tests\OpenClaw.Shared.Tests.csproj
dotnet test .\tests\OpenClaw.Tray.Tests\OpenClaw.Tray.Tests.csproj
```

첫 실행 주의사항: 새 워크트리에서 `dotnet test --no-restore`는 조용히 아무 테스트도 실행하지 않고
성공으로 종료할 수 있다. 첫 검증에서는 `--no-restore`를 생략하거나 테스트 프로젝트를 먼저 빌드해야 한다.

### 7.5 딥링크와 로컬 파일 경로

딥링크: `openclaw://settings`, `openclaw://setup`, `openclaw://chat`, `openclaw://commandcenter`,
`openclaw://send?message=Hello`, `openclaw://logs`, `openclaw://support-context`,
`openclaw://capability-diagnostics`

| 데이터 | 경로 |
| --- | --- |
| 앱 설정 | `%APPDATA%\OpenClawTray\settings.json` |
| 게이트웨이 레지스트리 | `%APPDATA%\OpenClawTray\gateways.json` |
| 기기 신원 키 | `%APPDATA%\OpenClawTray\gateways\<gateway-id>\device-key-ed25519.json` |
| exec 승인 기록 | `%APPDATA%\OpenClawTray\exec-approvals.json` |
| MCP 토큰 | `%APPDATA%\OpenClawTray\mcp-token.txt` |
| 로그 | `%LOCALAPPDATA%\OpenClawTray\openclaw-tray.log` |

기본 로컬 게이트웨이 URL: `ws://localhost:18789`

설치와 제거 전체 절차는 [docs/SETUP.md](docs/SETUP.md) 참고.

---

## 8. 플러그인인가 스킬인가 MCP인가

결론: **세 가지 모두 아니면서, 세 가지를 모두 품고 있는 네이티브 호스트 앱**이다.

| 후보 | 판정 | 이유 |
| --- | --- | --- |
| 플러그인 | 아니다 | 게이트웨이에 꽂는 JS 플러그인 모듈이 아니다 |
| 스킬 | 아니다 | `SKILL.md` 한 장짜리 지식 묶음이 아니다 |
| MCP | 부분적으로 그렇다 | MCP 서버를 내장하지만 그것이 본질은 아니다 |
| **네이티브 호스트 앱** | **그렇다** | 독립 실행되는 Windows 데스크톱 앱 |

동시에 갖는 4가지 정체성:

| 정체성 | 근거 |
| --- | --- |
| Windows 네이티브 앱 (본질) | `installer.iss`, `OpenClaw.Tray.WinUI`, `Package.appxmanifest` |
| OpenClaw Node | `WindowsNodeClient`, `NodeService` |
| MCP 서버 | `McpHttpServer.cs`, `McpToolBridge.cs` |
| 스킬 제공자 | `.agents/skills/winnode/SKILL.md`, `src/skills/windows-a2ui/SKILL.md` |

한 문장 요약: **"OpenClaw 노드 역할 + MCP 서버 + winnode CLI + 에이전트 스킬 문서를
한 덩어리로 묶은 Windows용 네이티브 에이전트 호스트 앱"**.

실용적 관점에서 가장 중요한 결론: Claude Code 같은 MCP 클라이언트에 등록하면
**59개 Windows 명령을 도구로 사용할 수 있다.**

---

## 9. API 토큰 정리

등장하는 토큰은 4종류이고, 사용자가 직접 준비할 것은 사실상 1개뿐이다.

| 종류 | 용도 | 직접 준비 필요 |
| --- | --- | --- |
| AI 모델 API 키 (Anthropic / OpenAI 등) | 게이트웨이가 LLM 호출 | **필요** (로컬 모델만 쓰면 불필요) |
| 게이트웨이 자격증명 (device / shared / bootstrap) | 앱과 게이트웨이 간 인증 | 불필요 (앱이 자동 발급 및 관리) |
| MCP Bearer 토큰 | 로컬 MCP 접근 인증 | 불필요 (앱이 43자 자동 생성) |
| GitHub 토큰 | 업데이트 확인 | 불필요 (공개 릴리스) |

### 완전 무료 구성이 가능하다

`OpenClaw.SetupEngine`에 다음 구성 요소가 이미 포함되어 있다.

- `LlamaRuntimeInstaller` (llama-server 런타임 설치)
- `HuggingFaceModelInstaller` (모델 다운로드)
- `LocalAiGpuVerification` (GPU 검증)
- `LocalAiInstallReconciler`, `LocalAiArtifactInstaller`, `LocalAiGatewayConfiguration`

즉 앱이 로컬 LLM 설치와 GPU 검증까지 자동으로 처리하므로,
클라우드 API 키 없이 0원으로 구성할 수 있다.
별도로 Windows에 설치된 Ollama를 `ollama.models` / `ollama.chat`으로 공유하는 경로도 있다.

주의: App-managed Local AI와 Shared Windows Ollama는 서로 독립적인 경로다.
하나를 켜도 다른 하나를 활성화하거나 재구성하거나 중지하지 않는다.

---

## 10. 왜 GitHub에서 유명한가

다섯 가지 요인이 겹쳐서 터졌다.

### 10.1 모체 OpenClaw의 폭발적 성장

- 2026년 1월 말 공개 후 1주일 내 10만 스타 돌파
- 2026년 2월 중순 약 21.6만 스타
- 2026년 3월 초 약 24.7만 스타, 포크 약 4.77만
- 제작자: Peter Steinberger (PSPDFKit 창업자), 애칭 Molty

성공 요인: 로컬에서 동작하는 개인 AI 에이전트(프라이버시와 소유권),
이미 쓰는 메신저 채널로 바로 접근, 그리고 **로컬 게이트웨이 + 에이전틱 루프 + 스킬 + 영속 메모리**
조합이 개인 AI 에이전트의 사실상 표준 설계도가 되었다는 점.

### 10.2 Scott Hanselman이 만들었다

`LICENSE`의 저작권자이자 최다 기여자. Microsoft의 유명 개발자로 .NET 커뮤니티에서 영향력이 크다.
저자 자체가 화제성을 만들었다.

### 10.3 Windows 사용자의 명확한 결핍을 해결한다

[docs/WINDOWS_NODE_ARCHITECTURE.md](docs/WINDOWS_NODE_ARCHITECTURE.md) 원문 요지:

> macOS는 네이티브 메뉴바 앱으로 카메라, 캔버스, 화면 캡처, 알림, 위치, system exec를 모두 지원하지만,
> Windows 사용자는 WSL2에 의존해 네이티브 UI 통합도, 카메라도, 캔버스도 없고 NAT 네트워킹 문제까지 겪는다.

Windows 사용자 수가 압도적으로 많은데 경험이 반쪽이었다. 이 리포가 그 격차를 메우는 공식 답이다.

### 10.4 품질이 실제로 높다

375,857줄 코드에 테스트 6,706개, 문서 40개 이상, CI 워크플로 8개(CodeQL 포함),
아키텍처 책임 원장, god file 감축 가드레일, 문서-코드 드리프트 테스트.
화제성 높은 오픈소스가 보통 품질을 희생하는 것과 반대다.

### 10.5 AI 에이전트가 기여하는 리포의 선구적 사례

`AGENTS.md`의 필수 검증 계약, `.agents/skills/` 8종, `docs/PROOF_POOLS.md`의 증거 강제,
`SkillMdDriftTests`의 문서-코드 동기화, `scripts/validate-*.ps1` 자동 검증.
인간과 AI 혼성 팀이 대규모 코드베이스를 운영하는 실전 교본으로 참조 가치가 높다.

---

## 11. 로컬 에이전트 구축에 주는 도움

### 11.1 바로 사용하는 경로

```text
A) 완전 로컬 구성
   설치 -> WSL 로컬 게이트웨이 자동 설치 -> 로컬 llama-server 또는 Ollama 연결
   -> Permissions에서 Capability 선택 -> 클라우드를 타지 않는 로컬 에이전트 완성

B) MCP 경로
   Local MCP Server 켜기 -> Claude Code 등에 MCP 서버 등록
   -> 59개 Windows 명령을 에이전트 도구로 사용
```

### 11.2 설계를 차용하는 경로 (가치가 가장 큼)

로컬 에이전트를 만들 때 반드시 마주치는 난제 6개에 대한 검증된 해답이 이미 구현되어 있다.

| 난제 | 이 리포의 해답 | 참고 위치 |
| --- | --- | --- |
| AI에게 명령 실행을 허용해도 되는가 | 4중 관문 (정책, 토글, 승인, 샌드박스) | `src/OpenClaw.Shared/Mxc/`, `exec-approvals.json` |
| 프로세스 격리를 어떻게 하는가 | MXC / AppContainer + 3단 정책 | `MxcExecutor.cs`, `SandboxPolicy.cs` |
| 능력을 여러 전송로에 노출하는 법 | 단일 Capability 레지스트리 + 다중 전송 | `McpToolBridge.cs`, `NodeService` |
| 기기 신원과 페어링 | Ed25519 키 + 토큰 우선순위 + 재승인 | `DeviceIdentityStore.cs`, `CredentialResolver.cs` |
| MCP 서버를 제대로 만드는 법 | JSON-RPC 2.0 + 취소 톰스톤 + 토큰 게이트 | `McpToolBridge.cs`, `McpHttpServer.cs` |
| AI가 UI를 그리게 하는 법 | A2UI v0.8 -> 네이티브 렌더러 | `src/skills/windows-a2ui/SKILL.md` |

가장 차용 가치가 높은 패턴 3개:

1. **단일 레지스트리 다중 전송**: Capability를 한 번 등록하면 모든 전송로에 자동 노출.
   비결은 스냅샷 대신 `Func<IReadOnlyList<INodeCapability>>`를 넘겨 매 호출 시 라이브로 읽는 것.
2. **자격증명 우선순위와 강등 금지 규칙**: 로컬 에이전트에서 반드시 터지는 버그 지점을 규칙으로 봉쇄.
3. **에이전트 친화 리포 운영**: `AGENTS.md` 검증 계약 + 문서-코드 드리프트 테스트 + 증거 강제.
   AI가 만든 PR의 신뢰도를 구조적으로 끌어올린다.

### 11.3 한계

| 한계 | 내용 |
| --- | --- |
| Windows 전용 | WinUI 3 + .NET 10. macOS / Linux에는 적용되지 않는다 |
| 판단 주체가 없음 | 게이트웨이(OpenClaw Core)가 필요하다. 단독 에이전트가 아니다 |
| 규모가 크다 | 375k 줄. 전체 이해보다 패턴 추출이 현실적이다 |
| 빌드 환경 제약 | Windows + .NET 10 + WinUI 워크로드 필요. Linux 컨테이너에서는 빌드 불가 |
| MXC 미지원 환경 | MXC 부재 시 비격리 호스트 실행으로 폴백할 수 있어 설정 주의 필요 |

---

## 12. React / PHP로 만들 수 있는 범위

| 만들려는 것 | React | PHP | 판정 |
| --- | --- | --- | --- |
| 웹 대시보드 (노드 상태 및 설정 관리) | 가능 (최적) | 가능 | 가능 |
| MCP 클라이언트 (도구 호출) | 가능 | 가능 | 가능 |
| MCP 서버 (도구 제공) | 가능 (Node) | 가능 | 가능 |
| 게이트웨이 플러그인 및 스킬 | 가능 (TS) | 불가 | React 쪽 가능 |
| A2UI 웹 렌더러 | 가능 (최적) | 불가 | 가능 |
| Electron 데스크톱 앱 | 가능 | 불가 | 가능하지만 무겁다 |
| Windows 네이티브 트레이 앱 | 불가 | 불가 | 불가 |
| 스크린샷 / 카메라 / TTS 직접 접근 | 불가 | 불가 | 불가 |
| MXC / AppContainer 샌드박스 | 불가 | 불가 | 불가 |
| WSL 배포판 생성 및 관리 | 불가 | 불가 | 불가 |
| `openclaw://` 딥링크 등록 | 불가 | 불가 | 불가 |

네이티브 영역은 Win32 / WinRT API가 필요하므로 C# / C++ / Rust만 가능하다.

### 12.1 권장 아키텍처

```text
+--------------------------------------------------+
|  React 웹 대시보드 (신규 개발)                     |
|  - 도구 브라우저 / 실행 / 결과 뷰어                |
|  - A2UI 웹 렌더러                                 |
+---------------------+----------------------------+
                      |  HTTPS
+---------------------v----------------------------+
|  PHP 또는 Node 프록시 (신규 개발)                  |
|  - 토큰 보관 (프론트엔드에 노출 금지)              |
|  - Origin / CORS 처리, 감사 로그, 권한 제어        |
+---------------------+----------------------------+
                      |  JSON-RPC (loopback)
+---------------------v----------------------------+
|  OpenClaw Windows Node (기존 C# 앱)               |
|  - MCP 서버 127.0.0.1:8765                        |
|  - 59개 네이티브 명령                              |
+--------------------------------------------------+
```

### 12.2 반드시 지켜야 할 제약

- `McpHttpServer`가 Origin / Host 검사를 수행하므로 브라우저에서 직접 호출하면 막힐 수 있다.
  서버 사이드 프록시를 두는 것이 정답이며, 토큰이 서버에만 남아 더 안전하다.
- MCP Bearer 토큰을 프론트엔드 번들에 포함하면 안 된다. 프록시 서버 환경변수로만 관리한다.
- PHP가 `127.0.0.1:8765`에 접근하려면 **같은 Windows PC에서 실행되어야 한다.**
  원격 PHP 서버는 루프백에 도달할 수 없고, 이는 의도된 보안 설계다.

---

## 13. 수익화 아이디어 10선

### 13.0 전제 조건

| 조건 | 상태 |
| --- | --- |
| 라이선스 | MIT. 상업적 이용, 수정, 재배포 자유 |
| 시장 규모 | 스타 약 24.7만, 포크 약 4.77만 |
| 빈 틈 | 웹 UI, 팀 관리, 컴플라이언스 영역이 거의 비어 있음 |
| 기술 장벽 | 네이티브는 어렵지만 위에 얹는 레이어는 React / PHP로 가능 |
| 경쟁 | 생태계가 어려서 선점 여지가 크다 |

법적 유의 사항: MIT이므로 상업화는 자유지만 `LICENSE` 파일과 저작권 표시를 유지해야 한다.
"OpenClaw" 상표와 로고를 공식 제품처럼 사용하지 말고, "for OpenClaw" 또는
"OpenClaw-compatible" 형태의 표현을 사용한다.

### 티어 1: 즉시 시작 가능

#### 아이디어 1. ClawDash (MCP 웹 대시보드 SaaS)

- 문제: 59개 명령이 있는데 GUI 브라우저가 없고 `winnode` CLI뿐이다.
- 해결: 브라우저에서 클릭으로 모든 노드 명령을 실행하고 결과를 시각화한다.
- 스택: React + TypeScript + Tailwind, 프록시는 Node 또는 Laravel
- 기능: 도구 브라우저, 파라미터 폼 자동 생성, 결과 뷰어(이미지 및 JSON), 실행 히스토리,
  즐겨찾기, 명령 시퀀스 매크로, 스케줄 실행
- 가격: Free (1 노드) / Pro 월 9달러 / Team 월 29달러
- 개발 기간: MVP 3~4주, 난이도 낮음
- 근거: CLI 도구에 GUI를 붙이는 것은 검증된 공식이다 (Docker Desktop, GitKraken)
- 차별화: 명령 시퀀스 매크로 저장과 스케줄링

#### 아이디어 2. A2UI Web Renderer (오픈코어 + 상용 라이선스)

- 문제: A2UI는 Windows(WinUI) 렌더러와 Lit 레퍼런스만 있고 React 렌더러가 없다.
- 해결: A2UI v0.8을 React 컴포넌트로 렌더링하는 npm 라이브러리
- 구성: 18개 컴포넌트 매핑, A2UIValue 태그드 유니온 파서, JSONL 4종 엔벨로프, dataBinding
- 수익: 기본 18종 무료(MIT) / Pro 일회성 199달러 (차트, 데이터그리드, 캘린더, 테마 시스템, 우선 지원)
- 부가: 테마팩 49달러, 기업 지원 계약 연 2,000달러
- 개발 기간: 4~6주, 난이도 중
- 근거: npm 다운로드가 무료 마케팅이 되고, 표준 렌더러가 되면 생태계 표준 지위를 얻는다

#### 아이디어 3. 셋업 대행 서비스

- 문제: WSL, 게이트웨이, 로컬 LLM, GPU, 권한 설정은 일반 사용자가 수행하기 어렵다.
- 해결: 원격 접속 셋업 대행 + 교육 + 유지보수

| 패키지 | 가격 | 내용 |
| --- | --- | --- |
| Basic | 150,000원 | 설치, 게이트웨이, 기본 권한 설정 (약 1.5시간) |
| Local AI | 350,000원 | 로컬 LLM 설치, GPU 최적화, 모델 선정 (약 3시간) |
| Business | 1,200,000원 | 팀 5대, 보안 정책 수립, 문서화, 교육 |
| 월 유지보수 | 80,000원 / 월 | 업데이트, 장애 대응, 원격 지원 |

- 초기 투자 0원, 즉시 시작 가능, 확장성은 낮음 (시간을 파는 구조)
- 반복 작업을 PowerShell로 자동화하면 아이디어 4의 재료가 된다

#### 아이디어 4. ClawKit (셋업 자동화 스크립트 팩)

- 구성: 무인 설치 스크립트, 보안 프로필 프리셋 3종, 대량 배포(Intune / GPO), 헬스체크, 롤백
- 가격: Personal 39달러 / Business 299달러 (사이트 라이선스) / Enterprise 1,500달러
- 개발 기간: 2~3주, 난이도 낮음
- 판매 채널: Gumroad, Lemon Squeezy (자체 결제 인프라 불필요)

### 티어 2: 중기 (3~6개월, 기업 시장)

#### 아이디어 5. ClawGuard (기업용 정책 및 컴플라이언스 관리)

- 문제: 기업이 AI 에이전트를 직원 PC에 배포하면 "누가 언제 어떤 명령을 실행했는가"를 증명해야 한다.
  그 도구가 현재 없다.
- 해결: 중앙 정책 배포, 전사 감사 로그 수집, 이상 탐지, 컴플라이언스 리포트
- 스택: React 대시보드 + Laravel 또는 Node API + PostgreSQL (수집 에이전트는 C# 필요)
- 기능: Capability 허용 정책 중앙 배포, 감사 로그 집계, 이상 탐지 알림,
  ISO 27001 / SOC 2 / GDPR / ISMS-P 리포트, 샌드박스 정책 강제, 역할 기반 권한
- 가격: 시트당 월 15달러 (최소 20시트) / Enterprise 연 25,000달러 이상
- 목표 고객: AI를 도입하는 50~500인 기업, 금융, 의료, 공공
- 개발 기간: 4~6개월, 난이도 높음, 수익 잠재력 최상

되는 이유:

1. 기업은 감사 가능성에 실제로 예산을 쓴다. 규제상 선택이 아니라 필수다.
2. 현재 이 영역이 비어 있어 선점하면 표준 지위를 얻는다.
3. 이 리포가 이미 exec 승인 기록, 진단 데이터, 텔레메트리 규약을 갖고 있어
   수집과 집계, 리포팅 레이어만 추가하면 된다.
4. B2B SaaS는 LTV가 높고 이탈률이 낮다.

#### 아이디어 6. 스킬 마켓플레이스 (업종 특화 스킬팩)

| 스킬팩 | 대상 | 가격 |
| --- | --- | --- |
| 회계 자동화 | 세무사, 소상공인 | 79달러 |
| 법무 문서 | 로펌, 법무팀 | 149달러 |
| 의료 기록 | 병원 (HIPAA 고려) | 199달러 |
| 영상 편집 워크플로 | 크리에이터 | 59달러 |
| 개발 자동화 | 개발팀 | 89달러 |
| SNS 운영 | 마케터 | 69달러 |

- 수익 구조: 직접 판매 또는 마켓 수수료 20~30% (플랫폼 포지션)
- 개발 기간: 스킬팩 1개당 2~3주, 마켓 플랫폼 약 2개월
- 유의: `mergisi/awesome-openclaw-agents`에 이미 162개 무료 템플릿이 존재한다.
  따라서 템플릿 자체가 아니라 검증, 지원, 업데이트 보장, 큐레이션을 팔아야 한다.

#### 아이디어 7. ClawRemote (모바일 원격 제어 앱)

- 스택: React Native / Expo + 릴레이 서버 (Node 또는 PHP)
- 기능: 스크린샷 실시간 뷰어, 빠른 명령 버튼, 음성 명령, 파일 전송, 푸시 알림
- 가격: Free / Pro 월 4.99달러
- 개발 기간: 2~3개월, 난이도 중
- 기술 난관: 노드 MCP는 루프백 전용이므로 게이트웨이를 경유하거나 안전한 릴레이를 직접 만들어야 한다.
  이 지점이 제품의 핵심 난이도이자 진입 장벽이 된다.

### 티어 3: 장기 (6개월 이상)

#### 아이디어 8. 멀티 OS 노드 플랫폼

macOS 네이티브 앱과 이 Windows 리포는 있지만 Linux 데스크톱 노드는 없다.
Linux 노드를 오픈소스로 공개하고 통합 관리 플랫폼을 유료화하는 오픈코어 모델.
월 19달러. Rust 또는 C++ 필요, 난이도 높음.

#### 아이디어 9. 에이전트 관측성(Observability) SaaS

[docs/TELEMETRY.md](docs/TELEMETRY.md)에 OpenTelemetry 가드레일이 이미 정의되어 있다.
어떤 명령이 느리고 어디서 실패하며 토큰 비용이 얼마인지 보여주는 에이전트 전용 APM.
월 29달러(스타터)에서 월 499달러(엔터프라이즈). 텔레메트리 규약이 이미 있어 연동이 쉽다.

#### 아이디어 10. 교육 콘텐츠 및 커뮤니티

| 상품 | 가격 |
| --- | --- |
| 로컬 AI 에이전트 구축 강의 | 149,000원 |
| AI 에이전트 보안 설계 전자책 | 29,000원 |
| 유료 커뮤니티 (템플릿, Q&A, 라이브) | 19,000원 / 월 |
| 기업 출강 | 2,000,000원 / 회 |

초기 투자 0원, 즉시 시작 가능. 다른 아이디어의 마케팅 채널로도 기능한다.

### 13.1 종합 비교

| 번호 | 아이디어 | 초기 투자 | 개발 기간 | 난이도 | 수익 잠재력 | 추천도 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | ClawDash | 낮음 | 3~4주 | 낮음 | 중 | 최상 |
| 2 | A2UI React 렌더러 | 낮음 | 4~6주 | 중 | 중상 | 상 |
| 3 | 셋업 대행 | 0원 | 즉시 | 낮음 | 중 | 상 |
| 4 | ClawKit | 낮음 | 2~3주 | 낮음 | 중 | 중 |
| 5 | ClawGuard | 중간 | 4~6개월 | 높음 | 최상 | 최상 |
| 6 | 스킬 마켓플레이스 | 낮음 | 2개월 이상 | 중 | 상 | 상 |
| 7 | ClawRemote | 중간 | 2~3개월 | 중 | 중 | 중 |
| 8 | 멀티 OS 플랫폼 | 높음 | 6개월 이상 | 높음 | 상 | 상 |
| 9 | 관측성 SaaS | 중간 | 4개월 이상 | 높음 | 상 | 상 |
| 10 | 교육 콘텐츠 | 0원 | 1~2개월 | 낮음 | 중 | 중 |

---

## 14. 실행 로드맵과 리스크

### 14.1 단계별 로드맵 (React / PHP 역량 기준)

```text
Phase 0 (0~1개월) 현금 확보와 문제 발굴
  - 셋업 대행 서비스 시작 (#3)
  - 반복 작업 스크립트화 (#4 재료)
  - 블로그와 영상으로 과정 기록 (#10 재료 및 마케팅)
  - 핵심: 실제 사용자의 페인포인트를 직접 듣는다

Phase 1 (1~3개월) 첫 제품 출시
  - ClawDash MVP 출시 (#1)
  - ClawKit 판매 시작 (#4)
  - A2UI React 렌더러 오픈소스 공개 (#2)
  - 핵심: 무료 오픈소스로 신뢰를 쌓고 유료 티어로 전환한다

Phase 2 (3~6개월) B2B 전환
  - Phase 1 고객 중 기업 니즈 발굴
  - ClawGuard 파일럿 (#5), 2~3개 기업과 공동 개발
  - 스킬 마켓 베타 (#6)
  - 핵심: 개인 월 9달러에서 기업 시트당 월 15달러로 단가를 점프시킨다

Phase 3 (6개월 이상) 스케일
  - ClawGuard 정식 출시, 컴플라이언스 인증
  - 마켓플레이스 수수료 모델 정착
  - 관측성 SaaS 추가 (#9) 및 번들 판매
  - 핵심: 플랫폼 포지션 확보
```

### 14.2 하나만 고른다면

**ClawDash (#1)로 시작해 ClawGuard (#5)로 확장한다.**

1. React 역량을 그대로 활용할 수 있다. 네이티브 지식이 필요 없다.
2. 3~4주면 출시해 빠른 피드백 루프를 만들 수 있다.
3. 대시보드 사용자 중 감사 로그를 요구하는 기업이 곧 ClawGuard의 첫 고객이 된다.
4. 두 영역 모두 현재 비어 있다.
5. 개인 월 9달러에서 기업 시트당 월 15달러로 단가가 점프한다.

### 14.3 리스크와 대응

| 리스크 | 대응 |
| --- | --- |
| OpenClaw 생태계 급변 | 핵심 로직을 MCP 표준에 맞춰 설계해 다른 에이전트에도 재사용 가능하게 한다 |
| 본가의 공식 대시보드 출시 | 감사, 정책, 컴플라이언스 등 기업 기능으로 차별화한다 |
| MIT라서 복제가 쉽다 | 데이터, 통합, 브랜드, 고객 관계로 방어한다. 코드는 방어 수단이 아니다 |
| 보안 사고 시 책임 문제 | 책임 한계 명시, 보안 감사, 사이버 보험 |
| 상표권 이슈 | 제품명에 "OpenClaw"를 직접 쓰지 않고 "for OpenClaw" 형태로 표기한다 |

---

## 15. 참고 문서 인덱스

### 이 리포 내부 문서

| 주제 | 문서 |
| --- | --- |
| 에이전트 검증 계약 | [AGENTS.md](AGENTS.md) |
| 프로젝트 개요 및 Capability 표 | [README.md](README.md) |
| 개발 가이드 | [DEVELOPMENT.md](DEVELOPMENT.md) |
| 아키텍처 소유권 원장 | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) |
| 연결 및 페어링 | [docs/CONNECTION_ARCHITECTURE.md](docs/CONNECTION_ARCHITECTURE.md) |
| Operator 및 Node 개념 | [docs/OPERATOR_NODE_CONCEPTS.md](docs/OPERATOR_NODE_CONCEPTS.md) |
| 게이트웨이, 노드, exec 흐름 FAQ | [docs/OPENCLAW_GATEWAY_NODE_EXEC_FAQ.md](docs/OPENCLAW_GATEWAY_NODE_EXEC_FAQ.md) |
| 로컬 MCP 모드 | [docs/MCP_MODE.md](docs/MCP_MODE.md) |
| Windows 노드 동작 및 테스트 | [docs/WINDOWS_NODE_TESTING.md](docs/WINDOWS_NODE_TESTING.md) |
| Windows 플랫폼 전략 (역사적 맥락) | [docs/WINDOWS_NODE_ARCHITECTURE.md](docs/WINDOWS_NODE_ARCHITECTURE.md) |
| 온보딩 마법사 | [docs/ONBOARDING_WIZARD.md](docs/ONBOARDING_WIZARD.md) |
| 설치 및 제거 | [docs/SETUP.md](docs/SETUP.md) |
| 셋업 엔진 재설계 | [docs/SETUP_ENGINE_REDESIGN.md](docs/SETUP_ENGINE_REDESIGN.md) |
| WSL 게이트웨이 운영 | [docs/WSL_GATEWAY_ADMIN.md](docs/WSL_GATEWAY_ADMIN.md) |
| WSL argv 확장 함정 | [docs/WSL_EXE_ARGV_PITFALL.md](docs/WSL_EXE_ARGV_PITFALL.md) |
| 증거 풀 정의 | [docs/PROOF_POOLS.md](docs/PROOF_POOLS.md) |
| 텔레메트리 가드레일 | [docs/TELEMETRY.md](docs/TELEMETRY.md) |
| 테스트 인벤토리 | [docs/TEST_COVERAGE.md](docs/TEST_COVERAGE.md) |
| A2UI 네이티브 렌더러 | [docs/A2UI_NATIVE_WINUI.md](docs/A2UI_NATIVE_WINUI.md) |
| winnode 에이전트 스킬 | [.agents/skills/winnode/SKILL.md](.agents/skills/winnode/SKILL.md) |
| Windows A2UI 스킬 | [src/skills/windows-a2ui/SKILL.md](src/skills/windows-a2ui/SKILL.md) |
| 보안 정책 | [SECURITY.md](SECURITY.md) |

### 외부 링크

| 주제 | 링크 |
| --- | --- |
| 이 리포지토리 (분석 대상) | https://github.com/bmshin94/openclaw-windows-node |
| 원본 리포지토리 | https://github.com/openclaw/openclaw-windows-node |
| OpenClaw 본체 (게이트웨이) | https://github.com/openclaw/openclaw |
| 공식 사이트 | https://openclaw.ai/ |
| Windows 공식 문서 | https://docs.openclaw.ai/platforms/windows |
| Discord 커뮤니티 | https://discord.gg/clawd |
| OpenClaw 위키백과 | https://en.wikipedia.org/wiki/OpenClaw |
| 에이전트 템플릿 모음 | https://github.com/mergisi/awesome-openclaw-agents |
| 아키텍처 분석 기사 | https://medium.com/@Micheal-Lanham/210-000-github-stars-in-10-days-what-openclaws-architecture-teaches-us-about-building-personal-ai-dae040fab58f |
| 개인 AI 에이전트 구축 및 보안 가이드 | https://www.freecodecamp.org/news/how-to-build-and-secure-a-personal-ai-agent-with-openclaw/ |
| OpenClaw 소개 (DigitalOcean) | https://www.digitalocean.com/resources/articles/what-is-openclaw |

---

## 부록: 검증 상태

이 문서는 **코드 변경이 없는 문서 전용 추가**이므로 빌드 및 테스트 산출물에 영향을 주지 않는다.

`AGENTS.md`가 요구하는 필수 검증(`./build.ps1`,
`dotnet test ./tests/OpenClaw.Shared.Tests/...`, `dotnet test ./tests/OpenClaw.Tray.Tests/...`)은
**Windows + .NET 10 + WinUI 워크로드 환경을 요구하므로 이 Linux 분석 환경에서는 실행할 수 없다.**

- 분석 환경: Linux 6.18.44 컨테이너
- 실행 가능했던 검증: 리포지토리 정적 전수조사 (파일 트리, 소스 grep, 문서 판독, git 이력)
- 실행 불가: `build.ps1` (PowerShell + Windows 필요), WinUI 대상 `dotnet test`, MXC E2E 프루프
- 코드 변경 여부: **없음** (Markdown 1개 파일 추가)

Windows 환경에서 이 브랜치를 검증하려면 다음을 실행한다.

```powershell
$env:OPENCLAW_REPO_ROOT = (Get-Location).Path
.\build.ps1
dotnet test .\tests\OpenClaw.Shared.Tests\OpenClaw.Shared.Tests.csproj
dotnet test .\tests\OpenClaw.Tray.Tests\OpenClaw.Tray.Tests.csproj
```
