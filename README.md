# taco-claude-plugin

Inflab 사내 LLM 사용량 트래커의 **Claude Code 플러그인**. Claude Code 세션
이벤트(hook)마다 `taco` CLI 를 호출해, 해당 세션 transcript 의 토큰 사용량을
수집한다.

이 레포는 **hook 등록 + 얇은 래퍼 스크립트**만 담는다. 실제 파싱/필터/전송
로직은 전부 `taco` CLI (`inflearn/taco`, brew 설치) 안에 있다.

## 사전 조건

`taco` CLI 가 PATH 에 있어야 한다.

```sh
brew tap inflearn/internal git@github.com:inflearn/homebrew-internal
brew install taco
taco init          # git email/name, 수집 org/디렉터리, auto_sync 설정
```

## 설치

```sh
claude plugin marketplace add git@github.com:inflearn/taco-claude-plugin
claude plugin install taco
```

설치 후 Claude Code 를 다시 시작하면 hook 이 등록된다. 이후 세션마다
`Stop` / `SubagentStop` / `PreCompact` / `SessionEnd` 이벤트에서 transcript 가
증분 수집된다.

## 동작

```
Claude Code hook (Stop/SubagentStop/PreCompact/SessionEnd)
  → ${CLAUDE_PLUGIN_ROOT}/scripts/run   (async, stdin = hook 이벤트 JSON)
    → taco hook                     (transcript 증분 파싱 → 로컬 버퍼)
      → auto_sync 켜진 경우에만 → devops-api
```

- **자동 sync 는 기본 OFF.** 기본은 로컬 버퍼에만 쌓이고, 전송은 `taco sync`
  또는 `taco init` 에서 auto_sync 활성화 시.
- 수집 대상은 `taco init` 의 `allowed_orgs` / `allowed_dirs` 필터를 따른다.
- 래퍼는 CLI 를 못 찾아도 조용히 종료 — 에디터를 절대 막지 않는다.

## 구성

| 파일 | 역할 |
|---|---|
| `.claude-plugin/plugin.json` | 플러그인 매니페스트 |
| `.claude-plugin/marketplace.json` | marketplace 레지스트리 매니페스트 |
| `hooks/hooks.json` | hook 이벤트 → `scripts/run` 등록 (async) |
| `scripts/run` | stdin 을 `taco hook` 으로 파이프하는 POSIX sh 래퍼 |

## 버전

플러그인 버전은 CLI 의 `taco hook` stdin 계약(transcript_path/cwd/
session_id)에 맞춰 올린다. 현재 CLI v0.2.0 과 호환.
