---
name: ich-pr-message
description: 최근 커밋 내용을 기반으로 PR 메시지를 영문/한글로 생성. 사용자가 모델을 선택할 수 있다.
---

# Step 0: 모델 선택

스킬 실행 전에 AskUserQuestion 도구로 사용할 모델을 입력받는다:

```
PR 메시지 생성에 사용할 모델을 선택하세요:

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

Bash 도구로 다음 2개 명령을 **병렬로** 실행하여 git 컨텍스트를 수집한다:

```bash
git log --oneline -20
```

```bash
git branch --show-current
```

# Step 2: 사용자 확인

최근 커밋 목록을 기반으로 PR 메시지를 생성합니다.

## 절차

1. **커밋 범위 확인**: Step 1의 git log를 사용자에게 보여주고, PR에 포함할 커밋 범위를 확인합니다. (예: "최근 5개 커밋을 포함할까요?" 또는 특정 커밋 해시 범위)
2. **이슈 번호 확인**: Resolve할 GitHub 이슈 번호가 있는지 사용자에게 물어봅니다. (예: "Resolve할 이슈 번호가 있나요?")
3. **그 외 요청사항 확인**: PR 메시지 작성 시 추가로 포함하거나 제외할 내용이 있는지 사용자에게 물어봅니다. (예: "그 외 PR 메시지에 반영할 요청사항이 있나요?")
4. **커밋 분석**: 사용자가 확인한 범위의 커밋들을 반드시 아래 두 가지 모두 실행하여 분석합니다:
   - `git log --stat <range>` — 커밋 메시지와 변경 파일 목록 확인
   - `git diff <base>...<head>` — **실제 코드 변경 내용을 반드시 읽고 분석**해야 합니다. 커밋 메시지만으로 PR 메시지를 작성하지 마세요.

# Step 3: PR 메시지 생성

사용자 확인이 완료되면, 수집된 컨텍스트와 커밋 분석 결과를 기반으로 PR 메시지를 생성한다.

### 모델별 실행 방식

#### Claude 모델 (opus / sonnet / haiku)

Agent 도구를 사용한다:

```
Agent({
  description: "PR 메시지 생성",
  subagent_type: "general-purpose",
  model: "{MODEL}",
  prompt: "{커밋 분석 결과 + 사용자 요청사항 + 아래 PR 메시지 생성 지침}"
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
      {"role": "system", "content": "당신은 PR 메시지를 영문/한글로 생성하는 전문가입니다."},
      {"role": "user", "content": "{커밋 분석 결과 + 사용자 요청사항 + 아래 PR 메시지 생성 지침}"}
    ],
    "temperature": 0.3,
    "max_tokens": 4096
  }'
```

- Bash 도구의 `timeout` 파라미터도 **300000ms**(5분)로 설정한다.
- 응답의 `.choices[0].message.content`를 결과로 사용한다.
- 응답 실패 시 사용자에게 알리고 중단한다.

### PR 메시지 생성 지침

아래 템플릿에 맞춰 **영문**으로 작성한 후, 한글로 번역하여 함께 제공합니다. 사용자의 요청사항을 반영합니다.

## 중요

- **반드시 커밋 범위를 사용자에게 먼저 확인**한 후 진행하세요.
- **커밋 범위 확인 후, 이슈 번호를 물어보세요.** 이슈 번호가 있으면 Description 본문에 자연스럽게 `Resolve #번호`를 포함합니다. (예: "... Resolve #6")
- PR 메시지만 제안하고, 실제 PR 생성은 실행하지 마세요.
- 사용자가 복사할 수 있도록 코드 블록으로 감싸서 제공하세요.

## PR 템플릿

아래는 출력 시 사용할 템플릿 구조입니다.
- HTML 주석(`<!-- -->`)은 작성 가이드이므로 **출력에 포함하지 마세요**.
- 각 섹션 헤더와 체크박스 구조만 유지하고, 내용을 채워서 출력합니다.

```markdown
## Description

{변경 내용 영문 서술. Resolve 이슈가 있으면 자연스럽게 포함}

## Pull request checklist

- [ ] My code follows the style guidelines of this project
- [ ] I have performed a self-review of my code
- [ ] Build (`npm run build`) was run locally
- [ ] Mention an issue number e.g. `#5379`
- [ ] Docs have been reviewed and added / updated if needed
- [ ] Tests for the changes have been added (for bug fixes / features)
- [ ] All tests passing

## Pull request type

- [ ] Bugfix
- [ ] Feature
- [ ] Code style update (formatting, renaming)
- [ ] Refactoring (no functional changes, no api changes)
- [ ] Documentation content changes
- [ ] Other (please describe):

## What is the current behavior?

{PR 이전의 상태/문제점 서술}

## What is the new behavior?

{변경/추가된 동작을 * 목록으로 서술}

## Other information

{추가 정보. 없으면 비워둠}

## To reviewers

{리뷰어에게 전달할 사항. 없으면 비워둠}
```

## 출력 형식

반드시 아래 형식으로 출력하세요:

### 1단계: 커밋 범위 확인

사용자에게 커밋 목록을 보여주고 범위를 확인합니다.

### 2단계: 영문 PR 메시지

위 PR 템플릿을 채워서 마크다운 코드 블록으로 제공합니다.
- Description: 변경 내용을 명확하게 영문으로 작성
- Pull request checklist: 해당하는 항목에 `[x]` 체크
- Pull request type: 해당하는 타입에 `[x]` 체크
- What is the current behavior: PR 이전 상태/문제점 영문 서술
- What is the new behavior: 변경/추가된 동작을 `*` 목록으로 영문 서술
- Other information: 필요시 추가 정보
- To reviewers: 리뷰어에게 전달할 사항

### 3단계: 한글 번역

2단계에서 작성한 영문 내용을 한글로 번역하여 동일한 템플릿 형식으로 마크다운 코드 블록에 제공합니다.
(체크박스, HTML 주석 등 템플릿 구조는 유지하고, 서술 내용만 한글로 번역)
