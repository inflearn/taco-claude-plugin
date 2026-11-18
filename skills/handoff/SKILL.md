---
name: handoff
description: 지금 세션을 다음 세션이 바로 이어받을 수 있게 요약해 taco 에 저장한다. 저장 뒤 /clear 하면 새 세션에 자동으로 이어진다.
disable-model-invocation: true
allowed-tools: mcp__plugin_taco_taco__taco_handoff_save
---

지금 세션을 다음 세션이 이어받을 수 있게 600단어 안으로 요약해라. 아래 다섯 가지를 이 순서로, 짧은 항목으로 쓴다.

1. 목표: 사용자가 이 세션에서 이루려던 것
2. 결정: 확정된 결정과 그 이유
3. 바뀐 파일: 편집·생성한 파일 경로와 무엇을 바꿨는지 한 줄씩
4. 남은 일: 다음에 할 일, 실패 중인 테스트, 현재 브랜치
5. 주의점: 다음 세션이 모르면 실수할 것

비밀번호·토큰·키 같은 비밀값은 요약에 쓰지 않는다.

요약을 `taco_handoff_save` 도구의 `summary` 로 저장한 뒤, 사용자에게 한 줄로만 알려라: "/clear 하면 새 세션이 이 요약으로 바로 이어져요."

`taco_handoff_save` 도구가 없거나 "unknown tool" 오류가 나면 taco CLI 가 오래된 것이다. 요약은 화면에 보여 주고, 사용자에게 `taco update` 후 Claude Code 를 다시 시작하라고 한 줄로 안내해라.
