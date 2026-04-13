---
name: ich-commit-message
description: 최근 작업 내용을 기반으로 Conventional Commits 스펙에 맞는 커밋 메시지를 생성. 사용자가 모델을 선택할 수 있다.
---

# Step 0: 모델 선택

스킬 실행 전에 AskUserQuestion 도구로 사용할 모델을 입력받는다:

```
커밋 메시지 생성에 사용할 모델을 선택하세요:

[Claude 모델]
  1. opus   — 가장 정밀한 분석
  2. sonnet — 균형 잡힌 성능 (기본값)
  3. haiku  — 빠른 응답 (가벼운 작업용)

[LM Studio]
  4. lmstudio — LM Studio에 현재 로드된 모델 사용 (localhost:1234)

번호 또는 모델명을 입력하세요. (기본값: 2/sonnet)
```

- 사용자가 값을 입력하지 않거나 빈 값이면 기본값 **sonnet**을 사용한다.
- `lmstudio` 선택 시, Bash 도구로 연결을 확인한다:
  ```bash
  curl -s --max-time 10 http://localhost:1234/v1/models
  ```
  - 응답 실패 시 사용자에게 알리고 모델을 다시 선택하도록 한다.
  - 정상 응답이면 로드된 모델명을 `{LMSTUDIO_MODEL}`에 저장한다.
- 이후 이 값을 `{MODEL}`로 참조한다.

# Step 1: 컨텍스트 수집

Bash 도구로 다음 3개 명령을 **병렬로** 실행하여 git 컨텍스트를 수집한다:

```bash
git diff --cached
```

```bash
git diff
```

```bash
git status --short
```

# Step 2: 커밋 메시지 생성

수집한 git diff와 status 정보를 기반으로 커밋 메시지를 생성한다.

### 모델별 실행 방식

#### Claude 모델 (opus / sonnet / haiku)

Agent 도구를 사용한다:

```
Agent({
  description: "커밋 메시지 생성",
  subagent_type: "general-purpose",
  model: "{MODEL}",
  prompt: "{Step 1 결과} + {아래 커밋 메시지 생성 지침}"
})
```

#### LM Studio 모델 (lmstudio)

Bash 도구로 OpenAI 호환 API를 호출한다. **타임아웃은 5분(300초)**으로 설정한다:

```bash
curl -s --max-time 300 http://localhost:1234/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "{LMSTUDIO_MODEL}",
    "messages": [
      {"role": "system", "content": "당신은 Conventional Commits 스펙에 맞는 커밋 메시지를 생성하는 전문가입니다."},
      {"role": "user", "content": "{Step 1 결과} + {아래 커밋 메시지 생성 지침}"}
    ],
    "temperature": 0.3,
    "max_tokens": 2048
  }'
```

- Bash 도구의 `timeout` 파라미터도 **300000ms**(5분)로 설정한다.
- 응답의 `.choices[0].message.content`를 결과로 사용한다.
- 응답 실패 시 사용자에게 알리고 중단한다.

### 커밋 메시지 생성 지침

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
