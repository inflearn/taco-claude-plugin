# taco-claude-plugin

taco(AI 코딩 사용량·비용 트래커)의 **Claude Code 플러그인**. Claude Code 세션
이벤트(hook)마다 `taco` CLI 를 호출해 세션 transcript 의 토큰 사용량을 수집하고,
taco MCP 서버(사용량 조회 도구)를 함께 등록한다 — 로컬(`taco` · stdio)과
원격(`taco-remote` · taco-api 커넥터, 모든 컴퓨터 합산) 둘 다.

이 레포는 **hook 등록 + MCP 등록 + 얇은 래퍼 스크립트**만 담는다. 실제 파싱/필터/
전송 로직은 전부 `taco` CLI 안에 있다.

## 사전 조건: taco CLI

`taco` CLI 가 PATH 에 있어야 한다.

```sh
# macOS / Linux
curl -fsSL https://inf.run/taco | sh

# Windows (PowerShell)
irm https://inf.run/taco.ps1 | iex
```

설치 후 로그인:

```sh
taco login
```

## 설치

```sh
claude plugin marketplace add inflearn/taco-claude-plugin
claude plugin install taco@inflearn
```

설치 후 Claude Code 를 다시 시작하면 hook 과 MCP 서버가 등록된다. 이후 세션마다
`Stop` / `SubagentStop` / `PreCompact` / `SessionEnd` 이벤트에서 transcript 가
증분 수집되고(맥락이 크면 화면에 한 줄 안내도 뜬다 — 아래), `/mcp` 에서 taco 도구(사용량·세션 조회)를 쓸 수 있다 — `taco`(로컬)
는 바로, `taco-remote`(원격)는 인프런 로그인 1회(OAuth) 후.

## 동작

```
Claude Code hook (Stop/SubagentStop/PreCompact/SessionEnd)
  → taco hook   (async, stdin = hook 이벤트 JSON)
    → transcript 증분 파싱 → 로컬 버퍼
      → auto_sync 켜진 경우 → taco 서버

Claude Code hook (Stop/UserPromptSubmit)
  → taco hook nudge --local-db <플러그인>/hooks/nudge-compat/local.db
                (sync, timeout 5s — transcript 끝부분만 읽음, ~15ms)
    → 맥락이 크면 {"systemMessage": "🌮 …"} 한 줄을 화면에 표시
```

- **맥락 안내**(taco CLI 0.12.11+). 옛 CLI 에선 아무것도 하지 않고 조용히 끝난다 —
  `hooks/nudge-compat` 는 일부러 **파일**이라, 옛 CLI 의 `taco hook` 은 그 아래
  `--local-db` 를 못 열고 출력 없이 exit 0 한다(`taco nudge` 를 직접 부르면 옛 CLI 는
  exit 1 → 매 턴 hook 오류). async hook 의 `systemMessage` 는
  사람에게 안 보이고 Claude 에게만 가므로, 안내는 별도 **동기** hook 으로 낸다.
  - Stop: 본 세션 마지막 턴 맥락이 15만 토큰을 넘으면(같은 세션은 10만 단위마다 한 번)
    "🌮 지금 맥락 32만 토큰 — `/compact`나 새 세션으로 이어가면 같은 한도로 더 쓸 수 있어요"
  - UserPromptSubmit: 캐시 유지 시간(5분/1시간)보다 오래 쉬었고 맥락이 10만을 넘으면
    "🌮 오래 쉬었다 이어가면 맥락 12만 토큰을 다시 읽어요 — 요약해 새 세션으로 시작하면 같은 한도로 더 써요"
  - 끄기: `taco config set nudge false` 또는 `TACO_NUDGE=0`.

- **자동 전송은 기본 ON**(`auto_sync`). 로그인 전에는 로컬 버퍼에만 쌓이고,
  `taco login` 후부터 서버로 전송된다.
- 수집 대상은 `allowed_orgs` / `allowed_dirs` 필터를 따른다.
- 래퍼는 CLI 를 못 찾아도 조용히 종료 — 에디터를 절대 막지 않는다.

## 구성

| 파일 | 역할 |
|---|---|
| `.claude-plugin/plugin.json` | 플러그인 매니페스트 |
| `.claude-plugin/marketplace.json` | marketplace 레지스트리 매니페스트 |
| `hooks/hooks.json` | hook 이벤트 → `taco hook` 등록 (async) + Stop/UserPromptSubmit → `taco hook nudge` (sync) |
| `hooks/nudge-compat` | 옛 CLI 호환 표지 파일(위 "맥락 안내" 참고) — 지우지 말 것 |
| `.mcp.json` | taco MCP 서버 등록 — `taco`(로컬 stdio) + `taco-remote`(원격 http) |

## 문서 · 대시보드

- 문서: https://taco.inflab.com/docs
- 대시보드: https://taco.inflab.com

## 라이선스

[MIT](./LICENSE)
