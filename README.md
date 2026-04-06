# claude-plugin-jp

Claude Code용 커스텀 플러그인 모음입니다. 개인 워크플로우에 특화된 스킬들을 제공합니다.

## 플러그인 구성

### ich-plugin

ICH(사내) 환경에 특화된 Git 워크플로우 스킬을 제공합니다.

| 스킬 | 설명 | 상세 |
|------|------|------|
| `ich-commit-message` | git diff/status를 분석하여 Conventional Commits 스펙에 맞는 커밋 메시지를 영문/한글로 생성 | [SKILL.md](plugins/ich-plugin/skills/ich-commit-message/SKILL.md) |
| `ich-pr-message` | 커밋 이력을 기반으로 PR 템플릿에 맞는 PR 메시지를 영문/한글로 생성 | [SKILL.md](plugins/ich-plugin/skills/ich-pr-message/SKILL.md) |

### personal-plugin

개인적으로 만든 고급 개발 워크플로우 스킬을 모아놓은 플러그인입니다.

| 스킬 | 설명 | 상세 |
|------|------|------|
| `elite-dev-mode` | 코드 변경마다 병렬 CTO 레벨 리뷰를 수행하는 엘리트 개발 모드. 변경 영향도(High/Medium/Low)에 따라 즉시/일괄/최종 리뷰를 자동 트리거 | [SKILL.md](plugins/personal-plugin/skills/elite-dev-mode/SKILL.md) |
| `plan-storm` | 3명의 에이전트가 병렬로 독립적인 구현 계획을 수립하고, 교차 리뷰를 거쳐 최종 통합 계획서를 생성 | [SKILL.md](plugins/personal-plugin/skills/plan-storm/SKILL.md) |

## 프로젝트 구조

```
claude-plugin-jp/
├── .claude-plugin/
│   └── marketplace.json          # 마켓플레이스 메타데이터
├── plugins/
│   ├── ich-plugin/
│   │   ├── .cluade-plugin/
│   │   │   └── plugin.json
│   │   └── skills/
│   │       ├── ich-commit-message/
│   │       │   └── SKILL.md
│   │       └── ich-pr-message/
│   │           └── SKILL.md
│   └── personal-plugin/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           ├── elite-dev-mode/
│           │   └── SKILL.md
│           └── plan-storm/
│               └── SKILL.md
├── LICENSE                       # Apache License 2.0
└── README.md
```

## 라이선스

[Apache License 2.0](LICENSE)