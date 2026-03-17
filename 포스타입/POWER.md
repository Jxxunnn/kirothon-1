# Investigate Power

## Description

Sentry 에러 자동 분석 Power. 슬랙 스레드에서 Sentry 이슈 링크를 추출하고, 에러 상세 정보를 조회하며, 워크스페이스 코드를 분석하여 RCA 보고서를 생성한다.

프론트엔드(Next.js) Sentry 프로젝트 기반의 에러 분석에 특화되어 있다.

## Power Name

investigate

## Command

`/investigate <slack_thread_url>`

슬랙 스레드 URL을 입력으로 받아 해당 스레드에 포함된 Sentry 이슈를 자동 분석한다.

## Keywords

sentry, investigate, error, rca, slack, root-cause-analysis, stacktrace, next.js, frontend

## Required MCP Servers

이 Power를 사용하려면 다음 2개의 MCP 서버가 반드시 설정되어 있어야 한다.

| MCP Server | 용도 |
|---|---|
| **Slack MCP** | 슬랙 워크스페이스의 스레드 메시지를 읽고, 분석 결과를 스레드에 게시하기 위해 사용 |
| **Sentry MCP** | Sentry 프로젝트의 에러 상세 정보(스택트레이스, 이벤트, 태그 등)를 조회하기 위해 사용 |

## Onboarding

`/investigate` 명령이 실행되면, 분석을 시작하기 전에 아래의 사전 검증 단계를 **반드시** 순서대로 수행한다. 어느 단계에서든 실패하면 즉시 분석을 중단하고 사용자에게 안내 메시지를 제공한다.

---

### 1. Slack Thread URL 형식 검증

사용자가 입력한 URL이 다음 형식과 일치하는지 검증한다:

```
https://<workspace>.slack.com/archives/<channel_id>/p<timestamp>
```

- `<workspace>`: 슬랙 워크스페이스 이름 (영문, 숫자, 하이픈)
- `<channel_id>`: 채널 ID (예: `C01ABCDEF`)
- `<timestamp>`: 메시지 타임스탬프 (숫자, 예: `1234567890123456`)

**URL이 유효하지 않은 경우**, 다음 에러 메시지를 사용자에게 표시하고 분석을 중단한다:

> ❌ 입력한 URL이 올바른 Slack Thread URL 형식이 아닙니다.
>
> 올바른 형식: `https://<workspace>.slack.com/archives/<channel_id>/p<timestamp>`
>
> 예시: `https://my-team.slack.com/archives/C01ABCDEF/p1234567890123456`
>
> 슬랙에서 스레드의 "링크 복사" 기능을 사용하여 URL을 가져오세요.

---

### 2. MCP 서버 연결 상태 확인

URL 검증을 통과한 후, 필수 MCP 서버 2개의 연결 상태를 순서대로 확인한다.

#### 2-1. Slack MCP 연결 확인

Slack MCP 서버가 연결되어 있고 사용 가능한 상태인지 확인한다. Slack MCP의 도구(예: 메시지 읽기)를 호출하여 연결 상태를 검증한다.

**Slack MCP에 연결할 수 없는 경우**, 다음 설정 가이드를 사용자에게 제공하고 분석을 중단한다:

> ❌ Slack MCP 서버에 연결할 수 없습니다. 아래 가이드를 따라 설정해 주세요.
>
> **Slack MCP 설정 가이드**
>
> 1. Kiro IDE의 MCP 설정 파일을 엽니다: `.kiro/settings/mcp.json`
> 2. 아래 설정을 추가합니다:
>
> ```json
> {
>   "mcpServers": {
>     "slack": {
>       "command": "npx",
>       "args": ["-y", "@anthropic/slack-mcp"],
>       "env": {
>         "SLACK_BOT_TOKEN": "<your-slack-bot-token>",
>         "SLACK_TEAM_ID": "<your-slack-team-id>"
>       }
>     }
>   }
> }
> ```
>
> 3. `SLACK_BOT_TOKEN`과 `SLACK_TEAM_ID`를 실제 값으로 교체합니다.
> 4. 설정 저장 후 Kiro IDE를 재시작하거나 MCP 서버를 다시 연결합니다.
>
> 설정 완료 후 `/investigate` 명령을 다시 실행해 주세요.

#### 2-2. Sentry MCP 연결 확인

Sentry MCP 서버가 연결되어 있고 사용 가능한 상태인지 확인한다. Sentry MCP의 도구(예: 이슈 조회)를 호출하여 연결 상태를 검증한다.

**Sentry MCP에 연결할 수 없는 경우**, 다음 설정 가이드를 사용자에게 제공하고 분석을 중단한다:

> ❌ Sentry MCP 서버에 연결할 수 없습니다. 아래 가이드를 따라 설정해 주세요.
>
> **Sentry MCP 설정 가이드**
>
> 1. Kiro IDE의 MCP 설정 파일을 엽니다: `.kiro/settings/mcp.json`
> 2. 아래 설정을 추가합니다:
>
> ```json
> {
>   "mcpServers": {
>     "sentry": {
>       "command": "npx",
>       "args": ["-y", "@sentry/mcp-server"],
>       "env": {
>         "SENTRY_AUTH_TOKEN": "<your-sentry-auth-token>",
>         "SENTRY_ORGANIZATION": "<your-sentry-org-slug>"
>       }
>     }
>   }
> }
> ```
>
> 3. `SENTRY_AUTH_TOKEN`과 `SENTRY_ORGANIZATION`을 실제 값으로 교체합니다.
> 4. 설정 저장 후 Kiro IDE를 재시작하거나 MCP 서버를 다시 연결합니다.
>
> 설정 완료 후 `/investigate` 명령을 다시 실행해 주세요.

---

### 3. Onboarding 완료

위 모든 검증을 통과하면, URL에서 파싱한 정보(workspace, channel_id, thread_ts)를 사용하여 Phase 1(Sentry Context 수집)으로 진행한다.

## Steering Files

Onboarding이 완료되면, 아래 3개의 Steering 파일을 **반드시 순서대로** 실행한다. 각 단계는 이전 단계의 출력에 의존하므로 순서를 건너뛰거나 병렬로 실행할 수 없다.

| 순서 | Steering 파일 | 단계 | 설명 |
|------|--------------|------|------|
| 1 | `phase1-sentry-context.md` | Sentry 컨텍스트 수집 | Slack MCP로 스레드 메시지를 읽고, Sentry 이슈 링크를 추출하며, Sentry MCP로 에러 상세 정보를 조회한다 |
| 2 | `phase2-code-analysis.md` | 코드 분석 | Phase 1에서 추출한 스택트레이스의 파일 경로와 라인 번호를 기반으로 워크스페이스 코드를 읽고, 호출 체인을 추적하여 근본 원인을 추정한다 |
| 3 | `phase3-report.md` | 보고서 생성 | Phase 1, 2의 결과를 종합하여 RCA 보고서를 작성하고, 담당 파트를 판단한 뒤, 사용자 확인 후 슬랙 스레드에 게시한다 |

### 실행 규칙

1. **순차 실행**: `phase1-sentry-context.md` → `phase2-code-analysis.md` → `phase3-report.md` 순서로 실행한다. 이전 단계가 완료되어야 다음 단계로 진행할 수 있다.

2. **단계별 결과 요약 표시**: 각 Steering 파일의 실행이 완료되면, 해당 단계의 결과 요약을 사용자에게 반드시 표시한다.
   - **Phase 1 완료 시**: 추출된 Sentry 이슈 수, 각 이슈의 에러 타입, 발생 횟수, 분석 대상 소스 파일 경로 목록을 요약하여 표시한다.
   - **Phase 2 완료 시**: 분석한 파일 수, 추적한 호출 체인, 추정된 근본 원인을 요약하여 표시한다.
   - **Phase 3 완료 시**: RCA 보고서 생성 완료 여부, 슬랙 게시 결과, 멘션 대상 파트를 요약하여 표시한다.

3. **에러 발생 시 중단**: 어느 단계에서든 치명적 에러(예: Sentry MCP 조회 실패)가 발생하면, 해당 단계에서 분석을 중단하고 사유를 사용자에게 표시한다.
