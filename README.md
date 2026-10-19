---
title: orca-orchestration-rules
created: 2026-10-16
updated: 2026-10-19
tags: [domain/인공지능, area/ai에이전트, agent/claude-1b219bef-b3db-4ef6-8088-81f7f0d19120]
source: ai
---

# orca-orchestration-rules

> Orca 의 감독형 오케스트레이션(Run/Task 워커)을 언제 쓰고, 워커를 어떻게 띄우고 이름 붙이고 기다리고 정리하는지 정한 운영 규칙 스킬이다.

## 하는 일

- 감독은 사용자가 요청할 때만 들어가는 모드다. 감독 없는 Run 은 Run 이 없는 것보다 나쁘다.
- 명령 표면은 옮겨 적지 않고 Orca 의 버전 맞춤 가이드(`orca skills get orchestration`)에서 읽는다.

## 설치

에이전트 도구의 스킬 경로에 clone 하거나 링크한다.

```bash
git clone https://github.com/s12171934/orca-orchestration-rules.git ~/.claude/skills/orca-orchestration-rules
# 다른 도구는 그 도구의 스킬 경로에 링크한다. 예: ln -s ~/.claude/skills/orca-orchestration-rules ~/.codex/skills/orca-orchestration-rules
```

- Orca 와 그 `orchestration` 스킬이 설치되어 있어야 한다. 감독 없는 단순 격리 작업은 `orca-work-placement` 스킬이 맡는다.

## 구성과 공개 면

- [[20_Agents/skills/orca-orchestration-rules/SKILL|SKILL.md]] — 스킬 본문. 형식은 에이전트 도구가 정한다.
- [[20_Agents/skills/orca-orchestration-rules/references/코디네이터 비용 실측|코디네이터 비용 실측]] — §3 비용 규칙의 실측 근거와 수치.
- 공개 면은 스킬 이름(`orca-orchestration-rules`)뿐이다. 다른 스킬은 이 이름으로만 이 스킬을 가리킨다.
