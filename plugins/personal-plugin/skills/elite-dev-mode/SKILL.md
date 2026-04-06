---
name: elite-dev-mode
description: "Activate elite development mode with parallel CTO-level code reviews after every code change. Use this skill when the user says 'elite dev mode', 'elite mode', 'start with review', 'CTO review mode', 'reviewed development', 'review mode', or wants rigorous parallel code reviews during development. Also trigger when the user mentions 'CTO reviewers', 'parallel review workflow', 'reviewed coding session', or any variation of starting development with continuous review gates. This is a full development workflow — code changes happen AND get reviewed, not just review alone."
---

# Elite Dev Mode

Every code change goes through parallel CTO-level reviews before proceeding. Like pair programming — but with a panel of CTOs watching every move.

## Activation

### With Parameters (fast)

```
/elite-dev-mode                                          → all defaults
/elite-dev-mode reviewers=5 final=7                      → 5 mid, 7 final
/elite-dev-mode mid-model=sonnet final-model=opus        → different models per phase
/elite-dev-mode lang=en                                  → english reviews
```

### Without Parameters (interactive)

When invoked with no arguments, ask the user each setting interactively. Show the default in parentheses — if the user just confirms or says "default", use it.

```
Elite Dev Mode 설정을 시작합니다.

1. 중간 리뷰어 수? (기본: 3, 최소 3, 홀수만 가능)
2. 최종 리뷰어 수? (기본: 5, 최소 3, 홀수만 가능)
3. 중간 리뷰어 모델? (기본: opus) [opus / sonnet / haiku]
4. 최종 리뷰어 모델? (기본: opus) [opus / sonnet / haiku]
5. 리뷰 언어? (기본: ko) [ko / en / ja / ...]

모두 기본값으로 하시려면 "기본값" 이라고 답해주세요.
```

If the user says "기본값", "default", or "all default" — skip all questions and use defaults.

### Parameters

| Parameter | Default | Constraints | Description |
|-----------|---------|-------------|-------------|
| `reviewers` | `3` | min 3, odd only | Mid-task reviewer count |
| `final` | `5` | min 3, odd only | Final reviewer count |
| `mid-model` | `opus` | opus/sonnet/haiku | Model for mid-task reviewer agents |
| `final-model` | `opus` | opus/sonnet/haiku | Model for final reviewer agents |
| `lang` | `ko` | any language code | Review output language |

### Validation

If the user provides an invalid value, reject it with a clear error — do NOT auto-correct:

- **Even number**: `❌ 리뷰어 수는 홀수만 가능합니다. (입력: 4)`
- **Less than 3**: `❌ 리뷰어는 최소 3명이어야 합니다. (입력: 1)`
- **Invalid model**: `❌ 지원하지 않는 모델입니다. [opus / sonnet / haiku] 중 선택해주세요.`

After showing the error, ask for the value again.

### Approval Threshold

- **3 reviewers** → **2/3 must approve** (special case)
- **5+ reviewers** → **80% must approve** (round up)

| Reviewers | Threshold | Required |
|-----------|-----------|----------|
| 3 | 2/3 | 2명 |
| 5 | 80% | 4명 |
| 7 | 80% | 6명 |
| 9 | 80% | 8명 |

### Confirmation

After settings are determined, display the active config:

```
✅ Elite Dev Mode 활성화
┌─────────────────────────────────────────┐
│ 중간 리뷰어: 3명 (opus)                  │
│ 최종 리뷰어: 5명 (opus)                  │
│ 승인 기준: 2/3 (중간), 4/5 (최종)         │
│ 리뷰 언어: 한국어                         │
└─────────────────────────────────────────┘
```

## Core Workflow

### Impact Level Classification

Every code change is classified into one of three impact levels. This classification determines **when** a mid-review is triggered.

#### 🔴 High — Immediate Review

Changes that affect system correctness, security, or contracts. Review is triggered **immediately** after the change.

- New architecture pattern or structural change
- Security-related code (authentication, authorization, encryption, token handling)
- Interface / contract changes (public APIs, DTOs, domain events, message contracts)
- Domain model changes (aggregates, entities, value objects)
- **Critical process logic changes** — modifications to existing code that is part of a core business flow:
  - Authentication / authorization pipelines
  - Data ingestion / processing pipelines (e.g., document status transitions)
  - Payment / billing logic
  - Global error handling / exception filters
  - Middleware that affects all requests (rate limiting, CORS, request validation)
  - Event consumers / message handlers that drive state machines
  - Database migration scripts or schema changes
- Deletion of public/shared code (interfaces, base classes, shared utilities)

#### 🟡 Medium — Batched Review

Changes that add or modify functionality but don't touch critical paths. These are **accumulated** and reviewed together when **2–3 medium changes** have been made.

- New file creation (non-critical: helpers, extensions, DTOs)
- Logic changes in existing files that are **not** part of critical processes
- Configuration changes (appsettings, DI registration, options classes)
- Test file additions or modifications
- Endpoint additions that follow established patterns

#### 🟢 Low — Skip Mid-Review

Trivial changes that don't affect runtime behavior. These are only reviewed during the **final review**.

- Renaming (variables, methods, files)
- Formatting, whitespace, code style fixes
- Comment or documentation changes
- `using` statement additions/removals
- Moving code without logic changes (extract method with identical behavior)
- `.csproj` package version bumps (non-major)

### How to Classify

When classifying, ask these questions in order:

1. **Does it touch security, auth, or contracts?** → 🔴 High
2. **Does it modify logic in a critical process flow?** → 🔴 High
3. **Does it add new functionality or change existing behavior?** → 🟡 Medium
4. **Is it purely cosmetic or structural with no behavior change?** → 🟢 Low

> **When in doubt, round UP** — classify Medium as High, not as Low.

### Phase 1: Develop → Smart Review → Repeat

For each code change:

1. **Write code** — implement the change
2. **Classify** — determine the Impact Level (🔴/🟡/🟢)
3. **Apply trigger rule**:

| Impact | Action |
|--------|--------|
| 🔴 High | **Immediately** launch mid-reviewers in parallel |
| 🟡 Medium | **Accumulate**. When 2–3 medium changes are pending, launch mid-reviewers for the batch |
| 🟢 Low | **Skip**. Will be covered in the final review |

4. **When review triggers** — summarize the change(s), then spawn ALL reviewer agents simultaneously in a single message using the configured `mid-model`. NEVER launch them sequentially.
5. **Collect verdicts** — each returns APPROVE or REQUEST_CHANGES
6. **Check threshold** — apply approval rules above
7. **Approved** → proceed to next change
8. **Not approved** → discuss, revise, re-review

**Review tracking**: Maintain a running count of pending medium changes. Display it after each change:

```
[Medium 변경 누적: 2/3] — 다음 Medium 변경 시 일괄 리뷰 실행
```

**Force flush**: If a 🔴 High change occurs while medium changes are pending, include the pending medium changes in the same review batch.

**Skip review entirely for:**
- Reading files for research
- Running commands to gather info
- Planning or discussing approach

### Phase 1 → Phase 2 Transition

**Mid-review stops** when the implementation is considered complete and the build step begins:

- Once `dotnet build` (or equivalent build/compile command) is executed for final verification, **no more mid-reviews are triggered**
- Any remaining accumulated medium changes are folded into the final review
- From this point forward, only the final review applies

> This prevents redundant review cycles after the "code complete" point. Bug fixes from build errors are covered by the final review.

### Phase 2: Final Review

After ALL tasks are complete and build succeeds:

1. **Launch final reviewers in parallel** — spawn ALL final reviewer agents simultaneously in a single message using the configured `final-model`. NEVER launch them sequentially.
2. **Each reviewer examines the ENTIRE set of changes** — holistic view, including any 🟢 Low changes that were never mid-reviewed
3. **Additional final-review criteria**:
   - How all changes work together as a whole
   - Overall architecture coherence
   - Cross-cutting concerns (security, performance, consistency)
   - Completeness — is anything forgotten?
   - Any accumulated 🟡 Medium or 🟢 Low changes not yet reviewed
4. **Same approval threshold** — apply approval rules above
5. **Not approved** → address feedback and re-run final review

## Spawning Reviewer Agents

Each reviewer is spawned via the Agent tool. The `model` parameter MUST match the configured model for that phase.

Example for mid-task review with `mid-model=sonnet`:
```
Agent(
  description="Mid reviewer 1",
  model="sonnet",             ← uses mid-model value
  prompt="<reviewer prompt>"
)
Agent(
  description="Mid reviewer 2",
  model="sonnet",             ← uses mid-model value
  prompt="<reviewer prompt>"
)
```

Example for final review with `final-model=opus`:
```
Agent(
  description="Final reviewer 1",
  model="opus",               ← uses final-model value
  prompt="<reviewer prompt>"
)
```

All agents for the same phase MUST be launched in a single message to run in parallel.

## Reviewer Prompt Template

Adapt the language naturally based on the `lang` parameter — don't just swap words, write fluently in the target language.

### Mid-Task Reviewer

```
You are CTO-level code reviewer #{N}. Review the following code change with deep, thorough analysis.

[Review language: {lang}]

## Context
{what was changed and why}

## Changed Files
{file paths with diffs or full content}

## Review Dimensions — evaluate ALL, skip none:

1. **Direction & Suitability** — Right approach? Better alternatives exist?
2. **Bugs & Errors** — Logic errors, null refs, race conditions, edge cases?
3. **Code Quality** — Clean, readable, maintainable? Follows codebase patterns?
4. **Project Structure** — Files in right place? Responsibilities separated?
5. **Naming** — Variables, methods, classes named clearly and consistently?
6. **Security** — Vulnerabilities? Input validation? Exposed secrets?
7. **Performance** — N+1 queries, unnecessary allocations, blocking calls?
8. **Reusability** — Appropriate reuse? Over-engineered for one-time use?

## Rules
- Be constructively critical — don't manufacture issues
- If something is good, say so
- Focus on REAL production-impacting problems
- Provide specific fix suggestions, not vague complaints
- Consider full context — don't review in isolation

## Verdict
End with exactly one of:
**APPROVE** — Good to proceed (minor suggestions OK)
**REQUEST_CHANGES** — Issues must be addressed before proceeding

Explain your reasoning.
```

### Final Reviewer

Same as mid-task prompt, plus this additional section:

```
You are reviewing the COMPLETE set of changes from this development session.
In addition to individual change quality, also evaluate:
- How all changes work together as a cohesive whole
- Overall architectural coherence
- Cross-cutting concerns (security, performance, consistency across files)
- Completeness — is the implementation missing anything?
```

## Displaying Results

### Approved

```
## 리뷰 결과 (2/3 승인 ✅)

### 리뷰어 1: APPROVE ✅
- 기존 패턴과 일관성 있는 구현
- 제안: `temp`를 `pendingResult`로 리네이밍 고려

### 리뷰어 2: APPROVE ✅
- 버그 없음, 엣지케이스 잘 처리됨

### 리뷰어 3: REQUEST_CHANGES ❌
- 42번 줄 null 체크 누락 — NullReferenceException 발생 가능
- 재시도 로직에 exponential backoff 적용 필요

→ 다음 작업으로 진행합니다 (기준 충족: 2/3)
```

### Not Approved

```
## 리뷰 결과 (1/3 승인 ❌)

### 차단 이슈 요약:
1. null 체크 누락 (리뷰어 2, 3 공통 지적)
2. 동시성 문제 (리뷰어 3)

수정 후 재리뷰를 진행합니다.
```

## User Override

The user can always override a failed review by saying "proceed anyway" or "무시하고 진행". Reviews are advisory, not a hard block. But clearly state what issues they are overriding before proceeding.

## Extended Thinking

All reviewer agents should analyze deeply — trace logic paths, consider edge cases, think about interactions with existing code. No surface-level scans. Every reviewer invests real cognitive effort before rendering a verdict.
