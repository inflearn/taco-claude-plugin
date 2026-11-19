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
    → 맥락이 크면 {"systemMessage": "taco: …"} 한 줄을 화면에 표시

Claude Code hook (SessionStart: startup|clear)
  → taco hook sessionstart --local-db <플러그인>/hooks/nudge-compat/local.db
                (sync, timeout 5s — 로컬 메모 디렉터리만 읽음)
    → 같은 프로젝트 직전 세션의 이어받기 메모가 있으면 그 경로를 Claude 맥락에 넣음

Claude Code hook (PostCompact)
  → taco hook   (async — 압축 요약을 이어받기 메모에 보관)

Claude Code hook (PreCompact)
  → taco hook precompact --local-db <플러그인>/hooks/nudge-compat/local.db
                (sync, timeout 5s — 네트워크·DB 없음)
    → stdout 의 지시문이 압축 요약 지시에 덧붙는다
```

- **맥락 안내**(taco CLI 0.12.11+). 옛 CLI 에선 아무것도 하지 않고 조용히 끝난다 —
  `hooks/nudge-compat` 는 일부러 **파일**이라, 옛 CLI 의 `taco hook` 은 그 아래
  `--local-db` 를 못 열고 출력 없이 exit 0 한다(`taco nudge` 를 직접 부르면 옛 CLI 는
  exit 1 → 매 턴 hook 오류). async hook 의 `systemMessage` 는
  사람에게 안 보이고 Claude 에게만 가므로, 안내는 별도 **동기** hook 으로 낸다.
  - Stop: 본 세션 마지막 턴 맥락이 15만 토큰을 넘으면(같은 세션은 10만 단위마다 한 번)
    "taco: 지금 맥락 32만 토큰 — `/compact`나 새 세션으로 이어가면 같은 한도로 더 쓸 수 있어요"
  - UserPromptSubmit: 캐시 유지 시간(5분/1시간)보다 오래 쉬었고 맥락이 10만을 넘으면
    "taco: 오래 쉬었다 이어가면 맥락 12만 토큰을 다시 읽어요 — 요약해 새 세션으로 시작하면 같은 한도로 더 써요"
  - 끄기: `taco config set nudge false` 또는 `TACO_NUDGE=0`.

- **이어받기 메모**(taco CLI 0.13.0+, 설정 없음). 맥락이 큰 세션의 메모(최근 요청·바꾼 파일·
  남은 할 일·압축 요약)를 taco 가 이 컴퓨터에만 남기고(서버 전송 없음, 7일 보관), 같은
  프로젝트에서 새 세션이나 `/clear` 로 시작하면 그 메모의 경로를 Claude 에게 넣는다.
  사람에게는 첫 프롬프트에 "taco: 직전 세션(32만 토큰)의 이어받기 메모를 넣어 뒀어요 — "이어서"라고만
  하면 돼요" 한 줄이 뜬다. `/taco:handoff` 를 쓰면 Claude 가 직접 정리한 요약을 저장해 두고
  `/clear` 하면 그 요약으로 바로 이어진다. 끄기: `TACO_HANDOFF=0`.
  - SessionStart 는 **동기** hook 이다. async SessionStart 의 출력은 첫 요청보다 늦게 끝나면
    다음 턴에야 Claude 에게 간다(Claude Code 2.1.311 실측). 동기여도 세션 시작 중 입력은 막지
    않고 Claude 의 첫 응답만 hook 이 끝날 때까지 기다린다. 이 hook 은 로컬 메모 디렉터리만 읽어
    보통 수십 ms 안에 끝나지만(실측), 디스크가 느리면 설정된 제한(5초)까지 기다릴 수 있다.
    옛 CLI 에선 위 호환 경로로 출력 없이 끝난다.

- **압축 요약 보강**(taco CLI 0.13.0+). 압축할 때 "편집·생성한 파일 경로, 확정된 결정과
  이유, 남은 할 일, 현재 브랜치와 실패 중인 테스트"를 요약에 남기라는 지시(800자 이하)를
  넣는다. 사용자가 `/compact <지시>` 로 직접 지시했으면 아무것도 넣지 않는다. 옛 CLI 는
  맥락 안내와 같은 호환 경로로 출력 없이 exit 0. 끄기: `TACO_PRECOMPACT=0`.

- **자동 전송은 기본 ON**(`auto_sync`). 로그인 전에는 로컬 버퍼에만 쌓이고,
  `taco login` 후부터 서버로 전송된다.
- 수집 대상은 `allowed_orgs` / `allowed_dirs` 필터를 따른다.
- 래퍼는 CLI 를 못 찾아도 조용히 종료 — 에디터를 절대 막지 않는다.

## 구성

| 파일 | 역할 |
|---|---|
| `.claude-plugin/plugin.json` | 플러그인 매니페스트 |
| `.claude-plugin/marketplace.json` | marketplace 레지스트리 매니페스트 |
| `hooks/hooks.json` | hook 이벤트 → `taco hook` 등록 (async) + Stop/UserPromptSubmit → `taco hook nudge` (sync) + SessionStart → `taco hook sessionstart` (sync) + PreCompact → `taco hook precompact` (sync) |
| `hooks/nudge-compat` | 옛 CLI 호환 표지 파일(위 "맥락 안내" 참고) — 지우지 말 것 |
| `skills/handoff/SKILL.md` | `/taco:handoff` — 지금 세션 요약을 MCP `taco_handoff_save` 로 저장 |
| `.mcp.json` | taco MCP 서버 등록 — `taco`(로컬 stdio) + `taco-remote`(원격 http) |

## 문서 · 대시보드

- 문서: https://taco.inflab.com/docs
- 대시보드: https://taco.inflab.com

## 라이선스

[MIT](./LICENSE)
