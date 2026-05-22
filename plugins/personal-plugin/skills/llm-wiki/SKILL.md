---
name: llm-wiki
description: "Karpathy의 LLM Wiki 패턴을 따라, 점진적으로 빌드되고 유지되는 영속적 마크다운 wiki를 만들고 운영하는 스킬. raw source(원본 문서)와 사용자 사이에 LLM이 관리하는 wiki 레이어를 두어, 새 source가 들어올 때마다 요약·entity/concept 페이지 갱신·상호참조·모순 표시까지 자동으로 처리한다. 사용자가 'wiki 만들어줘', '지식 베이스 구축', 'knowledge base 만들기', 'source 추가해줘', 'wiki에 정리해줘', 'llm-wiki', 'wiki에 질문', 'wiki에 물어봐', 'wiki 점검', 'wiki lint', 'lint wiki' 등을 요청할 때 트리거. 또한 research 노트 누적, 독서 동반 wiki, 개인 저널, 경쟁사 분석, 여행 계획 등 시간에 걸쳐 정보를 누적·정리하는 모든 상황에서 적극 사용한다. 단순 RAG/검색 요청이 아니라, 정리된 지식 자산을 점진적으로 빌드하려는 의도가 보이면 트리거한다."
---

# LLM Wiki

Karpathy의 LLM Wiki 패턴을 운영하는 스킬. raw source와 사용자 사이에 영속적인 마크다운 wiki 레이어를 두고, LLM이 bookkeeping(요약, 상호참조, 정리, 갱신)을 모두 담당한다.

## 핵심 컨셉

기존 RAG는 매 쿼리마다 raw 문서에서 fragment를 다시 찾고 합성한다. 지식이 누적되지 않는다.

LLM Wiki는 다르다:
- **Wiki는 한 번 컴파일되고 계속 최신 상태로 유지되는 자산**
- 새 source가 들어오면 LLM이 즉시 wiki에 반영 (요약 + 관련 페이지 갱신 + 모순 표시)
- 사용자는 **소싱·탐색·질문**, LLM은 **bookkeeping** 담당
- 사용자는 wiki를 직접 편집하지 않는다. LLM이 라이터, 사용자는 리더

3-Layer 아키텍처:
1. **Raw sources** — `./raw/` 하위. 불변. 원본 문서
2. **Wiki** — `./wiki/` 하위. LLM이 전적으로 소유. 마크다운 페이지
3. **Schema** — `./wiki/CLAUDE.md`. 도메인 맞춤 컨벤션·워크플로우 정의. 사용자와 co-evolve

---

## Obsidian Skills 의존성 (필수)

이 스킬은 [kepano/obsidian-skills](https://github.com/kepano/obsidian-skills) plugin이 함께 설치되어 있어야 동작한다. wiki 페이지·index·log를 직접 작성하지 않고, 모든 경우 해당 plugin의 전문 스킬을 호출하여 Obsidian 문법 정확성을 보장한다.

`obsidian-skills`가 설치되어 있지 않으면 **llm-wiki는 작업을 시작하지 않는다.** fallback 모드는 제공하지 않는다 — Obsidian 호환 출력이 이 스킬의 핵심 가치이므로 어설픈 일반 markdown으로 대체하지 않는다.

### 호출 규칙

| 작업 | 호출할 스킬 |
|------|----------------|
| .md 페이지 생성·편집 (frontmatter, wikilinks, callouts, properties, embeds) | `obsidian:obsidian-markdown` |
| 웹 URL을 source로 ingest | `obsidian:defuddle` |
| `.base` 파일 생성·편집 (DB-like 뷰) | `obsidian:obsidian-bases` |
| `.canvas` 파일 생성·편집 (시각적 synthesis) | `obsidian:json-canvas` |
| Vault 대량 조회·검색·플러그인 조작 | `obsidian:obsidian-cli` |

페이지 1개를 생성할 때마다 직접 Write를 호출하지 말고, `obsidian:obsidian-markdown` 스킬을 통해 "다음 frontmatter와 본문으로 페이지를 만들어줘"와 같이 위임한다. 이 방식은 다음을 보장한다:
- wikilink 문법 (`[[페이지명]]`, `[[페이지명|표시명]]`, `[[페이지명#섹션]]`) 정확성
- callout 문법 (`> [!info]`, `> [!warning]`, `> [!quote]` 등) 정확성
- frontmatter YAML 형식 (특히 list/object 표기, 날짜 포맷)
- embeds (`![[페이지명]]`, `![[이미지.png|400]]`) 문법
- properties (Obsidian Properties 뷰와 호환되는 frontmatter 키)

### Step -1: 의존성 강제 체크 (모든 모드 공통, 가장 먼저 수행)

llm-wiki가 어떤 모드(Init/Ingest/Query/Lint)로 호출되든, **Step 0 모드 감지보다 먼저** 다음을 수행한다:

1. 현재 세션에서 사용 가능한 스킬 목록을 확인하여 `obsidian:` prefix를 가진 다음 스킬들이 모두 활성화되어 있는지 검증:
   - `obsidian:obsidian-markdown` (필수)
   - `obsidian:defuddle` (필수)
   - `obsidian:obsidian-bases` (필수)
   - `obsidian:json-canvas` (필수)
   - `obsidian:obsidian-cli` (필수)

2. **하나라도 누락되면 즉시 작업을 중단하고** 사용자에게 다음을 안내:

   ```
   ⚠️ llm-wiki는 obsidian-skills plugin이 필요합니다.

   누락된 스킬: {누락 목록}

   다음 명령으로 설치해주세요:

       /plugin marketplace add kepano/obsidian-skills
       /plugin install obsidian@obsidian-skills

   설치 후 Claude Code 세션을 재시작(또는 /plugin reload)한 다음
   llm-wiki를 다시 호출하면 정상 동작합니다.
   ```

3. 사용자에게 "설치 완료 후 다시 호출해주세요"라고 안내하고 **이 턴은 그대로 종료**한다. fallback 진행 절대 금지.

4. 모든 의존 스킬이 활성화되어 있으면 Step 0(모드 감지)로 진행한다.

이 강제 체크는 사용자의 wiki가 일관된 Obsidian 호환 형식으로 누적되는 것을 보장하기 위함이다. 한 페이지는 obsidian-markdown으로 만들고 다른 페이지는 raw로 만들면 wikilink·callout·frontmatter가 서로 호환되지 않아 vault 전체가 깨질 수 있다.

---

## Step 0: 모드 감지 및 분기

스킬 실행 시 가장 먼저 현재 wiki 상태와 사용자 발화를 분석하여 4가지 모드 중 하나로 분기한다.

### 0-1. Wiki 존재 확인

cwd 기준 다음 파일들의 존재 여부를 확인한다:
- `./wiki/CLAUDE.md` (schema)
- `./wiki/index.md` (카탈로그)

둘 다 없으면 **Init 모드 후보**. 둘 중 하나라도 있으면 **운영 모드 후보**.

### 0-2. 발화 분석

사용자 메시지의 키워드/의도를 분석:
- "만들어줘", "시작", "init", "셋업" → **Init**
- 파일 경로/URL 제시, "추가", "ingest", "정리해줘" → **Ingest**
- 질문형 ("~는 뭐야", "비교해줘", "관련 source 알려줘") → **Query**
- "점검", "lint", "health check", "정리 좀" → **Lint**

### 0-3. 분기 결정

| Wiki 존재 | 발화 의도 | 진행 모드 |
|----------|----------|----------|
| 없음 | 명확하지 않음 | Init 제안 후 사용자 확인 |
| 없음 | Init | Init 모드 |
| 없음 | Ingest/Query/Lint | "wiki가 없어요. 먼저 init할까요?" 안내 후 Init |
| 있음 | 명확 | 해당 모드 진행 |
| 있음 | 애매 | AskUserQuestion으로 모드 선택 |

---

## Init 모드

새 wiki를 부트스트랩한다. 도메인에 맞춰 schema를 co-create하는 것이 핵심이다.

### Init 1. 도메인 입력

AskUserQuestion으로 다음을 묻는다:

**질문 1: Wiki 도메인은 무엇인가요?**
- 자유 입력. 예시 제공: "내가 읽는 SF 소설 시리즈", "AI safety 연구 노트", "회사 OKR 추적", "교양서 독서 동반", "여행 계획", "신규 제품 경쟁사 분석"
- 도메인이 schema의 페이지 카테고리·메타데이터·워크플로우를 결정한다.

**질문 2: 주로 다룰 source 종류는?** (멀티셀렉)
- 텍스트 문서 (md, txt, pdf)
- 웹 페이지 (URL)
- 이미지 포함 콘텐츠
- 코드/데이터 파일

**질문 3: 저장 위치 기본값을 사용할까요?**
- `./wiki/`, `./raw/` (기본, 추천)
- 다른 경로 지정 (입력)

### Init 2. 디렉토리 생성

확정된 경로에 다음을 생성:
- `./wiki/`
- `./wiki/sources/` (source 요약 페이지)
- `./wiki/entities/` (인물·장소·조직·작품 등 고유 개체)
- `./wiki/concepts/` (개념·이론·테마)
- `./wiki/synthesis/` (분석·통찰 페이지. 주로 query 답변을 filing한 결과)
- `./wiki/comparisons/` (비교표·대조)
- `./raw/`

도메인에 따라 일부 카테고리는 생략 가능 (예: 개인 저널은 entities만, concepts는 생략).

### Init 3. Schema 파일 생성 — `./wiki/CLAUDE.md`

도메인에 맞춰 다음 구조로 작성:

```markdown
# {도메인} Wiki Schema

## 목적
{도메인에 맞춘 1-2문장 요약. 예: "내가 읽는 SF 소설 시리즈의 인물·세계관·플롯·테마를 누적 정리하는 wiki"}

## 3-Layer 아키텍처
1. **Raw sources** (`../raw/`): 불변. 원본 문서. 절대 수정하지 않음
2. **Wiki** (`./`): LLM이 전적으로 소유. 마크다운 페이지
3. **Schema** (이 파일): 컨벤션과 워크플로우 정의

## 페이지 카테고리
{도메인 맞춤. 예시:}
- `sources/` — 각 source의 요약 페이지 (frontmatter: date, source URL/path, key_topics)
- `entities/` — 인물·장소·조직 (예: 등장인물, 행성, 작가)
- `concepts/` — 개념·테마 (예: "초공간 이동", "윤리적 딜레마")
- `synthesis/` — 통합 분석
- `comparisons/` — 비교

## 페이지 공통 형식
- 모든 페이지는 YAML frontmatter로 시작
- 위키링크는 `[[페이지명]]` 형식 (Obsidian 호환)
- 다른 페이지를 인용할 때는 반드시 `[[]]`로 묶기
- 새로운 주장·사실은 source 백링크 필수: `(from [[sources/source-slug]])`

## 페이지별 frontmatter 예시
{도메인 맞춤 frontmatter 필드 정의}

## 워크플로우

### Ingest
1. Source 읽기 → 핵심 takeaway 사용자에게 확인
2. `sources/{slug}.md` 생성
3. 관련 entity/concept 페이지 갱신 또는 생성
4. 모순 발견 시 `> ⚠️ 모순: ...` 표시
5. `index.md`, `log.md` 갱신

### Query
1. `index.md`에서 관련 페이지 식별
2. 해당 페이지 읽어 답변
3. 가치 있는 분석이면 `synthesis/` 또는 `comparisons/`로 filing

### Lint
모순·stale·orphan·missing cross-ref·누락 페이지·data gap 점검

## 도메인 특이사항
{사용자가 시간이 지나면서 추가하는 도메인별 규칙}
```

이 schema 파일은 사용자와 시간이 지나며 함께 진화한다. Init 시점에는 합리적인 기본값으로 채우고, 운영 중 도메인 특이사항이 발견되면 추가한다.

### Init 4. 카탈로그 파일 생성 — `./wiki/index.md`

```markdown
# {도메인} Wiki - Index

> Wiki 전체 카탈로그. 각 페이지는 한 줄 요약과 함께 등록된다.
> Query 시 LLM이 이 파일을 먼저 읽어 관련 페이지를 식별한다.

## Sources
(아직 ingest된 source 없음)

## Entities
(아직 없음)

## Concepts
(아직 없음)

## Synthesis
(아직 없음)

## Comparisons
(아직 없음)
```

### Init 5. 로그 파일 생성 — `./wiki/log.md`

```markdown
# Wiki Log

> Wiki 운영 기록 (append-only). 각 항목은 `## [YYYY-MM-DD] {operation} | {title}` 헤더로 시작.
> `grep "^## \[" log.md`로 시간순 인덱싱 가능.

## [{today}] init | Wiki 초기화
- 도메인: {도메인}
- Source 종류: {종류}
- 위치: {wiki 경로}
```

### Init 6. 완료 안내

사용자에게 알린다:
- 생성된 파일 목록
- 다음 액션 제안: "raw/ 폴더에 첫 source를 넣고 ingest 해보세요" 또는 "source URL을 알려주시면 바로 ingest 하겠습니다"
- schema 파일을 보여주고 도메인 특이사항을 추가하고 싶은지 묻기

---

## Ingest 모드

새 source를 wiki에 통합한다. 한 번의 ingest로 보통 10~15개 페이지가 영향받는다.

### Ingest 1. Source 식별

다음 중 하나:
- 사용자가 명시한 파일 경로
- 사용자가 제공한 URL
- 명시 없으면 `./raw/` 하위에서 미처리 파일 자동 탐색 (sources/에 대응 페이지가 없는 파일)

### Ingest 2. Source 읽기

종류별 도구 선택:
- 로컬 텍스트/마크다운/PDF → Read
- 웹 URL → `obsidian:defuddle` 스킬 (Step -1에서 가용성을 이미 검증했으므로 반드시 사용)
- 이미지 포함 콘텐츠 → Read (Claude의 멀티모달 능력 활용)
- 코드/데이터 → Read

### Ingest 3. 핵심 takeaway 확인

읽은 직후, 사용자에게 1-2문장으로 핵심을 요약하여 보여주고:
- "이 source의 핵심을 이렇게 이해했는데 맞나요?"
- "특별히 강조하고 싶은 부분이 있나요?"

이 단계는 인간의 큐레이션 가치를 반영하기 위함이다. 사용자가 짧게라도 방향을 잡아주면 이후 페이지 갱신의 강조점이 달라진다.

### Ingest 4. Source 요약 페이지 생성

`./wiki/sources/{slug}.md` 작성. `{slug}`은 source 제목 기반 kebab-case.

```markdown
---
title: "{source 제목}"
date: {YYYY-MM-DD}
source_type: {article|book-chapter|paper|video|note|...}
source_ref: "{URL 또는 raw/ 경로}"
key_topics:
  - {토픽1}
  - {토픽2}
related_entities:
  - "[[entities/이름]]"
related_concepts:
  - "[[concepts/이름]]"
---

# {source 제목}

## 핵심 요약
({사용자가 강조한 점을 반영한 1-3문단})

## 주요 내용
({섹션별 또는 논리 흐름별 요약})

## 새로운 사실 / 주장
- {사실1} (이 source에서 처음 등장)
- {사실2}

## 기존 wiki와의 관계
- [[entities/X]] 항목에 새 정보 추가
- [[concepts/Y]]에서 ⚠️ 모순 발견 (자세한 내용은 해당 페이지 참조)

## 인용할 만한 부분
> {중요 인용문}
```

### Ingest 5. Entity/Concept 페이지 갱신·생성

Source에서 언급된 모든 entity·concept을 식별하고, 각각에 대해:

1. `./wiki/entities/` 또는 `./wiki/concepts/` 하위에서 기존 페이지 검색 (Glob/Grep)
2. **기존 페이지**:
   - 새 정보를 적절한 섹션에 추가 (Edit)
   - 페이지 하단의 "Source 백링크" 섹션에 `- [[sources/{slug}]]` 추가
   - 기존 주장과 모순되면 `> ⚠️ 모순: {기존} vs {신규}` 블록 삽입
3. **없는 페이지**:
   - 새로 생성 (Write)
   - 표준 entity/concept 페이지 형식 사용
   - `index.md`의 해당 카테고리에 등록

표준 entity 페이지 형식:
```markdown
---
type: entity
category: {인물|장소|조직|작품|...}
first_seen: {sources/slug}
---

# {이름}

## 개요
(1-2문장 정의)

## 주요 속성
- ...

## 관련 항목
- [[entities/X]]
- [[concepts/Y]]

## Source 백링크
- [[sources/source-slug-1]]
- [[sources/source-slug-2]]
```

표준 concept 페이지 형식:
```markdown
---
type: concept
first_seen: {sources/slug}
---

# {개념명}

## 정의
(개념 정의)

## 핵심 내용
- ...

## 관련 개념
- [[concepts/X]]
- [[entities/Y]]

## Source 백링크
- [[sources/...]]
```

### Ingest 6. 상호참조 보강

새로 생성한 페이지가 기존 wiki의 다른 페이지를 언급하면 반드시 `[[]]` 위키링크로 변환한다. 텍스트 매칭만 하지 말고, 동일 개체에 대한 다른 표기(예: 약어, 별명)도 식별하여 링크한다.

### Ingest 7. Index 갱신

`./wiki/index.md`의 해당 카테고리에 항목 추가:

```markdown
## Sources
- [[sources/{slug}]] — {한 줄 요약}

## Entities
- [[entities/{이름}]] — {한 줄 요약} (신규)
```

신규 항목과 갱신 항목을 모두 반영.

### Ingest 8. Log 갱신

`./wiki/log.md`에 append:

```markdown
## [{YYYY-MM-DD}] ingest | {source 제목}
- Source: {경로 또는 URL}
- 생성된 페이지: N개
  - [[entities/X]] (신규)
  - [[concepts/Y]] (신규)
- 갱신된 페이지: M개
  - [[entities/Z]] (새 사실 2개 추가)
- 모순 표시: K건
  - [[concepts/W]]: {간단한 설명}
```

### Ingest 9. 사용자에게 변경 요약 보고

다음을 표 또는 bullet로 보고:
- 생성된 페이지 N개 (목록)
- 갱신된 페이지 M개 (목록)
- 모순 K건 (해당 페이지로 안내)
- 추천 다음 액션: "wiki 그래프 뷰로 새 연결 확인" / "모순 페이지 검토" / "관련 source 추가 탐색"

---

## Query 모드

Wiki에 질문하고 답을 받는다. 좋은 답은 wiki에 다시 filing하여 누적시킨다.

### Query 1. 관련 페이지 식별

1. `./wiki/index.md` 먼저 읽기 (전체 카탈로그)
2. 질문과 관련된 페이지를 카테고리·요약으로 식별
3. 필요시 Grep으로 키워드 검색하여 누락된 페이지 추가 발견

### Query 2. 페이지 읽기 및 답변 작성

선정된 페이지들을 Read. 답변 작성 시:
- 모든 인용·근거는 `[[페이지명]]` 위키링크로 출처 표시
- source 원문 인용이 필요하면 해당 source 페이지의 "인용할 만한 부분" 활용
- 모순이 있는 영역이면 양쪽 입장 모두 제시

### Query 3. Wiki 자산화 제안

답변이 다음 중 하나라면 사용자에게 wiki에 filing할지 묻는다:
- 여러 source를 통합한 분석
- 두 entity/concept을 비교한 결과
- 새롭게 발견한 패턴·통찰
- 사용자가 반복해서 물을 법한 질문에 대한 답

승인 시:
- `./wiki/synthesis/{topic-slug}.md` 또는 `./wiki/comparisons/{topic-slug}.md`로 저장
- 표준 synthesis 페이지 형식 사용 (질문·답변·근거 페이지 링크 포함)
- `index.md`의 Synthesis/Comparisons 섹션에 등록
- `log.md`에 `## [날짜] query-filed | {topic}` append

표준 synthesis 페이지 형식:
```markdown
---
type: synthesis
date: {YYYY-MM-DD}
question: "{원본 질문}"
sources_used:
  - "[[sources/...]]"
related_pages:
  - "[[entities/...]]"
  - "[[concepts/...]]"
---

# {제목}

## 질문
{원본 질문}

## 답변
{본문. 근거 페이지를 위키링크로 인용}

## 근거
- [[sources/...]] — {어떤 근거인지}
- ...

## 추가 탐색 거리
- ...
```

---

## Lint 모드

Wiki의 건강 상태를 점검한다. 자동 수정 가능한 항목과 판단이 필요한 항목을 구분하여 보고한다.

### Lint 1. 점검 항목

각 항목별로 다음을 수행:

**(1) 모순 점검**
- `grep "⚠️ 모순" wiki/**/*.md`로 표시된 모순 수집
- 각 모순에 대해 현재 상태 (해결됨/미해결) 보고

**(2) Stale 정보**
- `log.md`에서 각 entity/concept별 마지막 갱신 일자 추출
- 가장 최근 source date와 비교
- 갭이 크면 (예: 3개월 이상) "재검토 권장" 표시

**(3) Orphan 페이지**
- 모든 wiki 페이지에 대해 `[[페이지명]]` 인바운드 링크 수 카운트 (index.md 제외)
- 0인 페이지를 orphan으로 보고

**(4) Missing cross-ref**
- 각 페이지의 본문에서 다른 페이지 제목(또는 별명)이 언급되는지 검색
- `[[]]`로 묶이지 않은 경우 missing cross-ref로 보고

**(5) 누락 페이지**
- 여러 source에서 같은 entity/concept이 언급되지만 자체 페이지가 없는 경우 식별
- "추가 권장" 목록 작성

**(6) Data gap**
- 페이지 내 섹션이 비어있거나 "TBD"인 경우
- 외부 source 부족으로 깊이가 얕은 영역
- "추가 source 탐색" 또는 "web search 권장" 제안

### Lint 2. 자동 수정 가능 항목

다음은 사용자 승인 후 자동 수정:
- Missing cross-ref → `[[]]` 변환
- Orphan 페이지 → index.md에 누락된 항목 추가
- 빈 섹션 → 다른 페이지 참고하여 채우기 (가능한 경우)

### Lint 3. 판단이 필요한 항목

다음은 보고만 하고 사용자 판단을 기다림:
- 모순 (어느 쪽이 맞는지)
- Stale (실제로 outdated인지)
- 누락 페이지 (실제로 필요한지)
- Data gap (어떤 source로 채울지)

### Lint 4. 리포트 형식

```markdown
# Wiki Lint Report - {YYYY-MM-DD}

## 자동 수정 가능 (승인 시 처리)
- Missing cross-ref: N건
  - [[페이지A]] → [[페이지B]] 참조 누락
  ...
- Orphan 페이지: M건
  - [[페이지C]] (index 등록 누락)

## 판단 필요
- 모순: K건
- Stale: L건
- 누락 페이지 추천: P건
- Data gap: Q건

## 추천 다음 액션
- {구체적 제안}
```

### Lint 5. Log 갱신

```markdown
## [{date}] lint | {수정 N건 / 보고 M건}
- 자동 수정: ...
- 보고: ...
```

---

## 운영 원칙

### 사용자는 reader, LLM은 writer
- 사용자에게 "이 페이지 직접 수정해주세요" 요청하지 않는다
- 모든 wiki 편집은 LLM이 수행. 사용자는 raw/ 폴더 관리·source 제공·질문·승인만

### Schema는 살아있는 문서
- 운영 중 새로운 컨벤션·패턴이 발견되면 `./wiki/CLAUDE.md`에 즉시 반영
- 도메인 특이사항은 schema의 "도메인 특이사항" 섹션에 누적

### Obsidian 호환성
- 모든 페이지에 YAML frontmatter
- 페이지 간 링크는 `[[페이지명]]` 위키링크 형식
- 사용자가 Obsidian으로 vault를 열어 그래프 뷰·검색·Dataview 활용 가능

### Log는 grep 가능하게
- 모든 log 항목은 `## [YYYY-MM-DD] {operation} | {title}` 형식 엄수
- 사용자가 `grep "^## \[" log.md`로 타임라인 추출 가능

### 인용은 반드시 위키링크로
- "이 source에서 알게 된 것"이라고 쓰지 말고 `[[sources/source-slug]]`라고 쓰기
- 다른 wiki 페이지를 언급할 때 위키링크로 묶기 (Obsidian backlink가 자동 작동)

### Init 후 첫 ingest는 천천히
- 첫 source는 사용자와 함께 진행하며 schema·페이지 형식을 다듬는다
- 사용자 확인 단계를 더 많이 거쳐 도메인에 맞는 패턴을 정착시킨다
- 두 번째 source부터는 빠르게 진행

---

## 도구 사용 가이드

| 작업 | 사용할 도구 |
|------|----------|
| 디렉토리·파일 존재 확인 | Glob |
| Source 읽기 (로컬, raw 파일 자체) | Read |
| Source 읽기 (웹 URL) | `obsidian:defuddle` 스킬 |
| Wiki 페이지 키워드 검색 | Grep 또는 `obsidian:obsidian-cli` |
| **모든 .md 페이지 생성·편집** | **`obsidian:obsidian-markdown` 스킬** |
| `.base` 파일 작업 | `obsidian:obsidian-bases` 스킬 |
| `.canvas` 파일 작업 | `obsidian:json-canvas` 스킬 |
| Vault 대량 조작 | `obsidian:obsidian-cli` 스킬 |
| 사용자 입력 받기 | AskUserQuestion |

**원칙**: wiki 디렉토리 안의 모든 마크다운/베이스/캔버스 파일 작업은 반드시 `obsidian:*` 스킬을 통해 수행한다. raw Write/Edit로 wiki 파일을 직접 만들거나 수정하지 않는다 (Step -1에서 이미 의존성을 강제했으므로 미설치 상태에서는 여기까지 도달하지 않는다). 예외: `./raw/` 하위의 원본 source 파일을 읽을 때는 Read를 사용한다 — raw source는 wiki가 아니므로 obsidian 스킬을 거치지 않아도 된다.
