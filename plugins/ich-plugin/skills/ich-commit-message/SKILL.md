---
name: ich-commit-message
description: 최근 작업 내용을 기반으로 Conventional Commits 스펙에 맞는 커밋 메시지를 생성
disable-model-invocation: true
---

# Dynamic Context

```shell
!git diff --cached
```

```shell
!git diff
```

```shell
!git status --short
```

# Instructions

위의 git diff와 status 정보를 분석하여 **Conventional Commits** 스펙에 맞는 커밋 메시지를 생성하세요.

## Conventional Commits 형식

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

## Type 종류

| Type | 용도 |
|------|------|
| `feat` | 새로운 기능 추가 |
| `fix` | 버그 수정 |
| `docs` | 문서 변경 |
| `style` | 코드 포맷팅 (기능 변경 없음) |
| `refactor` | 리팩토링 (기능/버그 변경 없음) |
| `perf` | 성능 개선 |
| `test` | 테스트 추가/수정 |
| `build` | 빌드 시스템, 외부 의존성 변경 |
| `ci` | CI 설정 변경 |
| `chore` | 기타 변경 (src/test 미포함) |
| `revert` | 이전 커밋 되돌림 |

## 규칙

1. **scope**: 변경된 모듈/레이어를 반영 (예: `domain`, `persistence`, `application`, `api`, `shared`, `infra`)
2. **description**: 영문 소문자로 시작, 마침표 없음, 명령형 (imperative mood)
3. **body**: 변경 이유와 핵심 내용을 작성 (필요한 경우에만)
4. **BREAKING CHANGE**: 호환성이 깨지는 변경이 있으면 footer에 `BREAKING CHANGE: <설명>` 명시
5. 여러 종류의 변경이 섞여 있으면 가장 의미 있는 type을 선택하되, body에서 추가 변경사항을 설명

## 출력 형식

반드시 아래 형식으로 **영문 커밋 메시지**와 **한글 번역**을 함께 출력하세요:

```
## English

<type>(<scope>): <description>

<body>

<footer>

---

## 한글

<type>(<scope>): <한글 설명>

<한글 body>

<한글 footer>
```

## 주의사항

- 변경사항이 없으면 (diff가 비어 있으면) 커밋할 내용이 없다고 알려주세요.
- staged 변경사항과 unstaged 변경사항이 모두 있으면, staged 기준으로 메시지를 작성하되 unstaged 변경도 참고로 언급하세요.
- 커밋 메시지만 제안하고, 실제 git commit은 실행하지 마세요.
